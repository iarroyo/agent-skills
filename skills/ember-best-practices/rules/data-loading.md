---
title: Data Loading and Async Concurrency Patterns
impact: HIGH
impactDescription: Prevents infinite render loops and provides better control of async operations
tags: ember-concurrency, tasks, data-loading, concurrency-patterns, getPromiseState
---

## Data Loading and Async Concurrency Patterns

This guide covers patterns for loading and sharing data in Ember applications:

- **Initial data loading**: Constructor, `@cached` getter, and Request component patterns
- **Shared requests**: Service caching, parent request distribution, and request managers with TTL
- **Route-based loading**: When to use route model hooks (first-level components only)
- **URL state management**: Query params driving service state for shareable, bookmarkable views
- **User-initiated concurrency**: ember-concurrency for debouncing, throttling, and preventing double-clicks

**Key principle**: Use `getPromiseState` for data loading; use ember-concurrency only for **user-initiated** actions.

### Why Not ember-concurrency for Data Loading?

**Incorrect (using ember-concurrency for data loading):**

```glimmer-js
// app/components/user-profile.gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { task } from 'ember-concurrency';

class UserProfile extends Component {
  @tracked userData = null;
  @tracked error = null;

  // WRONG: Setting tracked state inside task
  loadUserTask = task(async () => {
    try {
      const response = await fetch(`/api/users/${this.args.userId}`);
      this.userData = await response.json(); // Anti-pattern!
    } catch (e) {
      this.error = e; // Anti-pattern!
    }
  });

  <template>
    {{#if this.loadUserTask.isRunning}}
      Loading...
    {{else if this.userData}}
      <h1>{{this.userData.name}}</h1>
    {{/if}}
  </template>
}
```

**Why This Is Wrong:**
- Setting tracked state during render can cause infinite render loops
- ember-concurrency adds overhead unnecessary for simple data loading
- Makes component state harder to reason about
- Can trigger multiple re-renders

**Correct (use getPromiseState):**

```glimmer-js
// app/components/user-profile.gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

class UserProfile extends Component {
  @cached
  get userData() {
    const promise = fetch(`/api/users/${this.args.userId}`)
      .then(r => r.json());
    return getPromiseState(promise);
  }

  <template>
    {{#if this.userData.isPending}}
      <div>Loading...</div>
    {{else if this.userData.isRejected}}
      <div>Error: {{this.userData.error.message}}</div>
    {{else if this.userData.isFulfilled}}
      <h1>{{this.userData.value.name}}</h1>
    {{/if}}
  </template>
}
```

### Initial Data Loading Patterns

For data that should load immediately when a component renders (not triggered by user action), choose based on your needs:

| Pattern | Use When |
|---------|----------|
| Constructor | One-time fetch, no reactive dependencies |
| `@cached` getter | Fetch depends on reactive args that may change |
| Request component | Declarative loading, consistent UI patterns |

#### Constructor-based Loading

Use the constructor when you need to fetch data once when the component is instantiated.

> **Important:** The constructor runs only once when the component is first created. It will **never** re-run when arguments change. If your fetch depends on reactive args that may change over time, use a `@cached` getter instead.

```glimmer-js
// app/components/dashboard.gjs
import Component from '@glimmer/component';
import { getPromiseState } from 'reactiveweb';

class Dashboard extends Component {
  #request;

  constructor(owner, args) {
    super(owner, args);
    this.#request = fetch('/api/dashboard/stats').then(r => r.json());
  }

  get stats() {
    return getPromiseState(this.#request);
  }

  <template>
    {{#if this.stats.isPending}}
      <div class="skeleton-loader">Loading dashboard...</div>
    {{else if this.stats.isRejected}}
      <div class="error">Failed to load: {{this.stats.error.message}}</div>
    {{else if this.stats.isFulfilled}}
      <div class="dashboard">
        <p>Total users: {{this.stats.value.totalUsers}}</p>
        <p>Active sessions: {{this.stats.value.activeSessions}}</p>
      </div>
    {{/if}}
  </template>
}
```

#### Reactive Cached Getter

Use `@cached` when the fetch depends on reactive arguments - automatically re-fetches when dependencies change. See the user-profile example above.

#### Reusable Request Component

Create a reusable `Request` component for declarative, composable data loading with named blocks:

```glimmer-ts
// app/components/request.gts
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

interface RequestSignature<T> {
  Args: {
    fetch: () => Promise<T>;
  };
  Blocks: {
    loading: [];
    error: [error: Error];
    default: [data: T];
  };
}

export default class Request<T> extends Component<RequestSignature<T>> {
  @cached
  get state() {
    const promise = this.args.fetch();
    return getPromiseState(promise);
  }

  <template>
    {{#if this.state.isPending}}
      {{#if (has-block "loading")}}
        {{yield to="loading"}}
      {{else}}
        <div>Loading...</div>
      {{/if}}
    {{else if this.state.isRejected}}
      {{#if (has-block "error")}}
        {{yield this.state.error to="error"}}
      {{else}}
        <div>Error: {{this.state.error.message}}</div>
      {{/if}}
    {{else if this.state.isFulfilled}}
      {{yield this.state.value}}
    {{/if}}
  </template>
}
```

**Usage with named blocks:**

```glimmer-js
// app/components/order-list.gjs
import Request from './request';

const fetchOrders = () => fetch('/api/orders').then(r => r.json());

<template>
  <Request @fetch={{fetchOrders}}>
    <:loading>
      <div class="skeleton">
        <div class="skeleton-row"></div>
        <div class="skeleton-row"></div>
      </div>
    </:loading>

    <:error as |error|>
      <div class="error-banner">
        <p>Failed to load orders: {{error.message}}</p>
        <button type="button">Retry</button>
      </div>
    </:error>

    <:default as |orders|>
      <ul class="order-list">
        {{#each orders as |order|}}
          <li>{{order.id}} - {{order.status}}</li>
        {{/each}}
      </ul>
    </:default>
  </Request>
</template>
```

### Shared Requests Across Multiple Components

When multiple components need the same data simultaneously, avoid duplicate network requests by sharing the promise.

| Pattern | Use When |
|---------|----------|
| Service caching | Multiple unrelated components need same data |
| Parent Request | Components are in the same subtree |
| Request manager | Many shared requests with TTL/invalidation needs |
| Route model | First-level components only (avoid prop drilling) |
| URL state + Service | Shareable/bookmarkable state with deep component trees |

#### Service-based Request Caching

```glimmer-js
// app/services/data-cache.js
import Service from '@ember/service';
import { getPromiseState } from 'reactiveweb';

export default class DataCacheService extends Service {
  #requests = new Map();

  fetch(key, fetchFn) {
    if (!this.#requests.has(key)) {
      const promise = fetchFn();
      this.#requests.set(key, promise);
    }
    return this.#requests.get(key);
  }

  getState(key, fetchFn) {
    return getPromiseState(this.fetch(key, fetchFn));
  }

  invalidate(key) {
    this.#requests.delete(key);
  }
}
```

**Components sharing the same request:**

```glimmer-js
// app/components/user-header.gjs
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { cached } from '@glimmer/tracking';

class UserHeader extends Component {
  @service dataCache;

  @cached
  get user() {
    return this.dataCache.getState(
      `user:${this.args.userId}`,
      () => fetch(`/api/users/${this.args.userId}`).then(r => r.json())
    );
  }

  <template>
    {{#if this.user.isFulfilled}}
      <header>
        <img src={{this.user.value.avatar}} alt={{this.user.value.name}} />
        <h1>{{this.user.value.name}}</h1>
      </header>
    {{/if}}
  </template>
}
```

```glimmer-js
// app/components/user-sidebar.gjs - Same pattern, shares the request!
class UserSidebar extends Component {
  @service dataCache;

  @cached
  get user() {
    // Same key = same promise = no duplicate request!
    return this.dataCache.getState(
      `user:${this.args.userId}`,
      () => fetch(`/api/users/${this.args.userId}`).then(r => r.json())
    );
  }

  <template>
    {{#if this.user.isFulfilled}}
      <aside>
        <p>Email: {{this.user.value.email}}</p>
        <p>Role: {{this.user.value.role}}</p>
      </aside>
    {{/if}}
  </template>
}
```

#### Parent Request with Data Distribution

Lift the Request component to a common parent and pass data down:

```glimmer-js
// app/components/user-page.gjs
import Request from './request';
import UserHeader from './user-header';
import UserSidebar from './user-sidebar';

const fetchUser = (userId) => () =>
  fetch(`/api/users/${userId}`).then(r => r.json());

<template>
  <Request @fetch={{(fetchUser @userId)}} as |user|>
    <div class="user-page">
      {{! All children receive the same data - single request }}
      <UserHeader @user={{user}} />
      <UserSidebar @user={{user}} />
    </div>
  </Request>
</template>
```

```glimmer-js
// app/components/user-header.gjs - Receives data as arg, no fetching
<template>
  <header>
    <img src={{@user.avatar}} alt={{@user.name}} />
    <h1>{{@user.name}}</h1>
  </header>
</template>
```

#### Request Manager with TTL

For applications with many shared requests needing cache expiration:

```glimmer-js
// app/services/request-manager.js
import Service from '@ember/service';
import { getPromiseState } from 'reactiveweb';

export default class RequestManagerService extends Service {
  #cache = new Map();
  #defaultTTL = 5 * 60 * 1000; // 5 minutes

  request(key, fetchFn, options = {}) {
    const ttl = options.ttl ?? this.#defaultTTL;
    const cached = this.#cache.get(key);

    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.promise;
    }

    const promise = fetchFn();
    this.#cache.set(key, { promise, timestamp: Date.now() });
    promise.catch(() => this.#cache.delete(key)); // Allow retry on failure

    return promise;
  }

  getState(key, fetchFn, options) {
    return getPromiseState(this.request(key, fetchFn, options));
  }

  invalidate(key) {
    this.#cache.delete(key);
  }

  invalidatePattern(pattern) {
    for (const key of this.#cache.keys()) {
      if (key.includes(pattern)) {
        this.#cache.delete(key);
      }
    }
  }
}
```

**Important: Avoid body stream consumption errors**

A `Response` body can only be consumed once (via `.json()`, `.text()`, etc.). If you cache the raw `fetch()` promise and multiple consumers each call `.json()` on the resolved `Response`, the second consumer will throw `"Response: body stream already read"`. Use `Response.clone()` to ensure each consumer gets its own body stream:

```js
// WRONG - causes "body stream already consumed" errors
getUser(id) {
  if (!this.#cache.has(id)) {
    this.#cache.set(id, fetch(`/api/users/${id}`)); // Response object
  }
  return this.#cache.get(id).then(r => r.json()); // Second call fails!
}

// CORRECT - clone the Response before consuming the body
// Each consumer gets a clone, so the original Response stays unconsumed
getUser(id) {
  if (!this.#cache.has(id)) {
    this.#cache.set(id, fetch(`/api/users/${id}`));
  }
  return this.#cache.get(id).then(r => r.clone().json());
}

// ALSO CORRECT - cache the fully resolved data promise
// Sidesteps the issue entirely since all consumers share the parsed result
getUser(id) {
  if (!this.#cache.has(id)) {
    const promise = fetch(`/api/users/${id}`).then(r => r.json());
    this.#cache.set(id, promise);
  }
  return this.#cache.get(id); // All consumers get same resolved value
}
```

### Route-based Data Loading

Load data in the route's `model` hook and pass it to first-level components. This pattern works well when consumers are direct children of the route template.

**Warnings:**
- Only use for first-level components. For deeply nested components, use a service instead to avoid prop drilling.
- Do not preload data that isn't immediately required. Routes block rendering until `model` resolves.

```js
// app/routes/user.js
import Route from '@ember/routing/route';
import { action } from '@ember/object';
import { service } from '@ember/service';

export default class UserRoute extends Route {
  @service router;

  async model({ user_id }) {
    const response = await fetch(`/api/users/${user_id}`);
    if (!response.ok) throw new Error(`Failed to load user (${response.status})`);
    return response.json();
  }

  // Optional: intercept errors to handle specific cases (e.g., redirect on 403)
  // Return true to bubble up and render the error substate template
  @action
  error(error) {
    if (error.message.includes('403')) {
      this.router.replaceWith('login');
    } else {
      return true;
    }
  }
}
```

```glimmer-js
// app/templates/user.gjs
import UserHeader from '../components/user-header';
import UserProfile from '../components/user-profile';

<template>
  <div class="user-page">
    {{! First-level components - OK to pass model data }}
    <UserHeader @user={{@model}} />
    <UserProfile @user={{@model}} />
  </div>
</template>
```

```glimmer-js
// app/components/user-header.gjs - First-level, receives data directly
<template>
  <header>
    <h1>{{@user.name}}</h1>
  </header>
</template>
```

#### Loading & Error Substates (optional)

Since route `model` hooks block rendering until resolved, users see nothing while data loads. Ember provides **loading and error substates** to give feedback during transitions. These are not added to `router.js` — they exist implicitly as part of each route.

**Loading substate:** Ember looks for a loading template while a route's `model` hook is pending. For a route named `user`, it searches in order:

1. `user-loading` (sibling template)
2. `loading` or `application-loading` (application-level fallback)

For nested routes like `user.posts`, Ember searches upward through the hierarchy: `user.posts-loading` → `user.loading` / `user-loading` → `loading` / `application-loading`.

```glimmer-js
// app/templates/user-loading.gjs
// Shown automatically while the user route's model hook is pending
<template>
  <div class="loading-spinner">
    <p>Loading user...</p>
  </div>
</template>
```

```glimmer-js
// app/templates/loading.gjs
// Application-wide fallback for any route without a specific loading template
<template>
  <div class="loading-spinner">
    <p>Loading...</p>
  </div>
</template>
```

**Error substate:** When a route's `model` hook rejects, Ember transitions to the error substate. The rejected error becomes the substate's model. The search hierarchy mirrors loading: `user-error` → `error` / `application-error`.

```glimmer-js
// app/templates/user-error.gjs
// Shown automatically when the user route's model hook rejects
<template>
  <div class="error-page">
    <h2>Failed to load user</h2>
    <p>{{@model.message}}</p>
  </div>
</template>
```

> **Note:** Loading and error substates only apply to route `model` hooks. They do not affect component-level data loading with `getPromiseState`. See the route example above for programmatic error handling with the `error` action.

**When route loading causes prop drilling (avoid this):**

```glimmer-js
// BAD - Prop drilling through multiple levels
<template>
  <UserPage @user={{@model}}>
    {{! UserPage passes to UserSection, which passes to UserCard, etc. }}
  </UserPage>
</template>
```

For deeply nested consumers, use a service instead (see Service-based Request Caching above).

#### Avoid Preloading Data That Isn't Required

Routes block rendering until the `model` hook resolves. Do not load data for components that may not be displayed (tabs, modals, conditional sections).

**Incorrect (preloading all tab data in route):**

```js
// app/routes/user.js
// BAD - Loads ALL data even if user only views one tab
export default class UserRoute extends Route {
  async model({ user_id }) {
    const [user, posts, followers, settings] = await Promise.all([
      fetch(`/api/users/${user_id}`).then(r => r.json()),
      fetch(`/api/users/${user_id}/posts`).then(r => r.json()),
      fetch(`/api/users/${user_id}/followers`).then(r => r.json()),
      fetch(`/api/users/${user_id}/settings`).then(r => r.json())
    ]);
    return { user, posts, followers, settings };
  }
}
```

**Solution 1: Component loads its own data**

Let each tab component load data when it becomes visible:

```glimmer-js
// app/components/user-posts-tab.gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

class UserPostsTab extends Component {
  @cached
  get posts() {
    // Only fetches when this component renders
    const promise = fetch(`/api/users/${this.args.userId}/posts`)
      .then(r => r.json());
    return getPromiseState(promise);
  }

  <template>
    {{#if this.posts.isPending}}
      <div>Loading posts...</div>
    {{else if this.posts.isFulfilled}}
      {{#each this.posts.value as |post|}}
        <article>{{post.title}}</article>
      {{/each}}
    {{/if}}
  </template>
}
```

```glimmer-js
// app/templates/user.gjs
import UserPostsTab from '../components/user-posts-tab';
import UserFollowersTab from '../components/user-followers-tab';

<template>
  <div class="user-page">
    <h1>{{@model.name}}</h1>

    <div class="tabs">
      {{#if (eq @controller.activeTab "posts")}}
        {{! Data loads only when tab is active }}
        <UserPostsTab @userId={{@model.id}} />
      {{else if (eq @controller.activeTab "followers")}}
        <UserFollowersTab @userId={{@model.id}} />
      {{/if}}
    </div>
  </div>
</template>
```

**Solution 2: URL params trigger conditional route loading**

Use query params to load only the data needed for the current view:

```js
// app/routes/user.js
import Route from '@ember/routing/route';

export default class UserRoute extends Route {
  queryParams = {
    tab: { refreshModel: true }
  };

  async model({ user_id, tab }) {
    // Always load base user data
    const user = await fetch(`/api/users/${user_id}`).then(r => r.json());

    // Conditionally load tab data based on URL
    let tabData = null;
    if (tab === 'posts') {
      tabData = await fetch(`/api/users/${user_id}/posts`).then(r => r.json());
    } else if (tab === 'followers') {
      tabData = await fetch(`/api/users/${user_id}/followers`).then(r => r.json());
    }

    return { user, tabData };
  }
}
```

```glimmer-js
// app/templates/user.gjs
<template>
  <div class="user-page">
    <h1>{{@model.user.name}}</h1>

    <nav class="tab-nav">
      <LinkTo @route="user" @query={{hash tab="posts"}}>Posts</LinkTo>
      <LinkTo @route="user" @query={{hash tab="followers"}}>Followers</LinkTo>
    </nav>

    {{! tabData contains only what's needed for current tab }}
    {{#if @model.tabData}}
      {{#each @model.tabData as |item|}}
        <div>{{item.name}}</div>
      {{/each}}
    {{/if}}
  </div>
</template>
```

**When to use each approach:**

| Approach | Use When |
|----------|----------|
| Component loads data | Tab/section may never be viewed, no URL state needed |
| URL params + route | View state should be shareable/bookmarkable |

#### Refreshing Data on Back/Forward Navigation

The route `model` hook does **not** re-trigger when navigating back/forward in browser history. If you need fresh data on every route visit, use the router's `routeDidChange` event.

```js
// app/services/user-data.js
import Service from '@ember/service';
import { service } from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

export default class UserDataService extends Service {
  @service router;

  @tracked currentUserId = null;
  #promise = null;

  constructor() {
    super(...arguments);
    // Listen for route changes (including back/forward navigation)
    this.router.on('routeDidChange', this.handleRouteChange);
  }

  willDestroy() {
    super.willDestroy(...arguments);
    this.router.off('routeDidChange', this.handleRouteChange);
  }

  handleRouteChange = (transition) => {
    // Check if we're entering a user route
    const userRouteInfo = transition.to?.find(route => route.name === 'user');

    if (userRouteInfo) {
      const userId = userRouteInfo.params.user_id;
      this.loadUser(userId, { forceRefresh: true });
    }
  };

  loadUser(userId, { forceRefresh = false } = {}) {
    if (forceRefresh || this.currentUserId !== userId) {
      this.currentUserId = userId;
      this.#promise = null; // Invalidate cache
    }
  }

  @cached
  get user() {
    if (!this.currentUserId) {
      return { isPending: false, isFulfilled: false, isRejected: false, value: null };
    }

    if (!this.#promise) {
      this.#promise = fetch(`/api/users/${this.currentUserId}`).then(r => r.json());
    }

    return getPromiseState(this.#promise);
  }
}
```

```js
// app/routes/user.js
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class UserRoute extends Route {
  @service userData;

  model({ user_id }) {
    // Initial load - service handles subsequent refreshes via routeDidChange
    this.userData.loadUser(user_id);
  }
}
```

```glimmer-js
// app/components/user-profile.gjs
import Component from '@glimmer/component';
import { service } from '@ember/service';

class UserProfile extends Component {
  @service userData;

  <template>
    {{#if this.userData.user.isPending}}
      <div>Loading...</div>
    {{else if this.userData.user.isFulfilled}}
      <h1>{{this.userData.user.value.name}}</h1>
      <p>{{this.userData.user.value.email}}</p>
    {{/if}}
  </template>
}
```

**When to use `routeDidChange`:**
- Data must be fresh on every route visit (not cached)
- Back/forward navigation should refetch data
- Route params alone don't trigger model hook (same route, different context)

**Alternative: Force model refresh with `refreshModel`**

For query param changes, you can use `refreshModel: true`:

```js
// app/routes/users.js
export default class UsersRoute extends Route {
  queryParams = {
    page: { refreshModel: true },
    sort: { refreshModel: true }
  };

  async model({ page, sort }) {
    // This WILL re-run when page or sort query params change
    const response = await fetch(`/api/users?page=${page}&sort=${sort}`);
    return response.json();
  }
}
```

Note: `refreshModel` only works for query param changes, not for back/forward navigation to the same URL.

### URL State Management with Query Params

For shareable, bookmarkable state (filters, pagination, search), use query params in the route to trigger service requests. Components then consume reactive data from the service.

This pattern enables:
- Shareable URLs with current state
- Browser back/forward navigation
- Bookmarkable filtered views
- Service-based reactivity for deeply nested components

```js
// app/services/products.js
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

export default class ProductsService extends Service {
  @tracked category = null;
  @tracked sortBy = 'name';
  @tracked page = 1;

  #promise = null;
  #lastParams = null;

  get currentParams() {
    return `${this.category}-${this.sortBy}-${this.page}`;
  }

  updateFilters({ category, sortBy, page }) {
    this.category = category ?? this.category;
    this.sortBy = sortBy ?? this.sortBy;
    this.page = page ?? this.page;
    this.#promise = null; // Invalidate cache
  }

  @cached
  get products() {
    // Invalidate if params changed
    if (this.#lastParams !== this.currentParams) {
      this.#promise = null;
      this.#lastParams = this.currentParams;
    }

    if (!this.#promise) {
      const params = new URLSearchParams({
        category: this.category || '',
        sort: this.sortBy,
        page: this.page
      });
      this.#promise = fetch(`/api/products?${params}`).then(r => r.json());
    }

    return getPromiseState(this.#promise);
  }
}
```

```js
// app/routes/products.js
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class ProductsRoute extends Route {
  @service products;

  queryParams = {
    category: { refreshModel: true },
    sort: { refreshModel: true },
    page: { refreshModel: true }
  };

  model({ category, sort, page }) {
    // Route triggers service update from URL params
    this.products.updateFilters({
      category,
      sortBy: sort || 'name',
      page: page || 1
    });

    // No need to return anything - components read from service
  }
}
```

```js
// app/controllers/products.js
import Controller from '@ember/controller';
import { service } from '@ember/service';
import { action } from '@ember/object';

export default class ProductsController extends Controller {
  @service products;

  queryParams = ['category', 'sort', 'page'];

  // Default values
  category = null;
  sort = 'name';
  page = 1;

  @action
  updateCategory(category) {
    this.category = category; // Updates URL, triggers route model hook
  }

  @action
  updateSort(sort) {
    this.sort = sort;
  }

  @action
  goToPage(page) {
    this.page = page;
  }
}
```

```glimmer-js
// app/components/product-filters.gjs
// Can be deeply nested - reads from service, not props
import Component from '@glimmer/component';
import { service } from '@ember/service';

class ProductFilters extends Component {
  @service products;

  <template>
    <div class="filters">
      <label>
        Category:
        <select {{on "change" (fn @onCategoryChange (pick "target.value"))}}>
          <option value="">All</option>
          <option value="electronics" selected={{eq this.products.category "electronics"}}>
            Electronics
          </option>
          <option value="clothing" selected={{eq this.products.category "clothing"}}>
            Clothing
          </option>
        </select>
      </label>

      <label>
        Sort by:
        <select {{on "change" (fn @onSortChange (pick "target.value"))}}>
          <option value="name" selected={{eq this.products.sortBy "name"}}>Name</option>
          <option value="price" selected={{eq this.products.sortBy "price"}}>Price</option>
        </select>
      </label>
    </div>
  </template>
}
```

```glimmer-js
// app/components/product-list.gjs
// Deeply nested - consumes reactive data from service
import Component from '@glimmer/component';
import { service } from '@ember/service';

class ProductList extends Component {
  @service products;

  <template>
    {{#if this.products.products.isPending}}
      <div class="loading">Loading products...</div>
    {{else if this.products.products.isRejected}}
      <div class="error">Failed to load: {{this.products.products.error.message}}</div>
    {{else if this.products.products.isFulfilled}}
      <ul class="product-grid">
        {{#each this.products.products.value.items as |product|}}
          <li class="product-card">
            <h3>{{product.name}}</h3>
            <p>{{product.price}}</p>
          </li>
        {{/each}}
      </ul>

      <Pagination
        @currentPage={{this.products.page}}
        @totalPages={{this.products.products.value.totalPages}}
        @onPageChange={{@onPageChange}}
      />
    {{/if}}
  </template>
}
```

```glimmer-js
// app/templates/products.gjs
import ProductFilters from '../components/product-filters';
import ProductList from '../components/product-list';

<template>
  <div class="products-page">
    <h1>Products</h1>

    {{! Components can be deeply nested - they read from service }}
    <ProductFilters
      @onCategoryChange={{@controller.updateCategory}}
      @onSortChange={{@controller.updateSort}}
    />

    <ProductList @onPageChange={{@controller.goToPage}} />
  </div>
</template>
```

**Benefits of this pattern:**
- URL reflects current state: `/products?category=electronics&sort=price&page=2`
- Users can bookmark/share filtered views
- Browser back/forward works correctly
- No prop drilling - deeply nested components read from service
- Single source of truth for filters and data

### User Input Concurrency with ember-concurrency

Use ember-concurrency for managing **user-initiated** async operations. It provides automatic cancelation, debouncing, and prevents race conditions.

**Incorrect (manual async handling):**

```glimmer-js
// app/components/search.gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

class Search extends Component {
  @tracked results = [];
  @tracked isSearching = false;
  currentRequest = null;

  @action
  async search(event) {
    const query = event.target.value;

    // Manual cancelation - error-prone
    if (this.currentRequest) {
      this.currentRequest.abort();
    }

    this.isSearching = true;
    const controller = new AbortController();
    this.currentRequest = controller;

    try {
      const response = await fetch(`/api/search?q=${query}`, {
        signal: controller.signal
      });
      this.results = await response.json();
    } catch (e) {
      if (e.name !== 'AbortError') {
        console.error(e);
      }
    } finally {
      this.isSearching = false;
    }
  }

  <template>
    <input {{on "input" this.search}} />
    {{#if this.isSearching}}Loading...{{/if}}
  </template>
}
```

**Correct (using ember-concurrency):**

```glimmer-js
// app/components/search.gjs
import Component from '@glimmer/component';
import { restartableTask, timeout } from 'ember-concurrency';
import { on } from '@ember/modifier';

class Search extends Component {
  searchTask = restartableTask(async (event) => {
    const query = event.target.value;
    await timeout(300); // Debounce
    const response = await fetch(`/api/search?q=${query}`);
    return response.json(); // Return, don't set @tracked
  });

  <template>
    <input type="search" {{on "input" this.searchTask.perform}} />

    {{#if this.searchTask.isRunning}}
      <div>Searching...</div>
    {{/if}}

    {{#if this.searchTask.lastSuccessful}}
      <ul>
        {{#each this.searchTask.lastSuccessful.value as |result|}}
          <li>{{result.name}}</li>
        {{/each}}
      </ul>
    {{/if}}

    {{#if this.searchTask.last.isError}}
      <div>Error: {{this.searchTask.last.error.message}}</div>
    {{/if}}
  </template>
}
```

### Task Modifiers

Choose the right modifier for your concurrency pattern:

```glimmer-js
import Component from '@glimmer/component';
import { dropTask, enqueueTask, restartableTask } from 'ember-concurrency';

class FormActions extends Component {
  // restartableTask: Cancels previous, starts new (search/autocomplete)
  searchTask = restartableTask(async (query) => {
    const response = await fetch(`/api/search?q=${query}`);
    return response.json();
  });

  // dropTask: Ignores new requests while running (prevent double-submit)
  saveTask = dropTask(async (data) => {
    const response = await fetch('/api/save', {
      method: 'POST',
      body: JSON.stringify(data)
    });
    return response.json();
  });

  // enqueueTask: Queues requests sequentially
  processTask = enqueueTask(async (item) => {
    const response = await fetch('/api/process', {
      method: 'POST',
      body: JSON.stringify(item)
    });
    return response.json();
  });

  <template>
    <button
      {{on "click" (fn this.saveTask.perform @data)}}
      disabled={{this.saveTask.isRunning}}
    >
      {{if this.saveTask.isRunning "Saving..." "Save"}}
    </button>

    {{#if this.saveTask.lastSuccessful}}
      <div>Saved successfully!</div>
    {{/if}}

    {{#if this.saveTask.last.isError}}
      <div>Error: {{this.saveTask.last.error.message}}</div>
    {{/if}}
  </template>
}
```

### TaskInstance API for Derived Data

ember-concurrency provides derived state - no tracked properties needed:

- `task.isRunning` - Boolean if any instance is running
- `task.last` - Most recent TaskInstance (successful or failed)
- `task.lastSuccessful` - Most recent successful TaskInstance (persists during new attempts)
- `taskInstance.value` - The returned value
- `taskInstance.isError` - Boolean if failed
- `taskInstance.error` - The error object

### Quick Reference

**Use getPromiseState for:**
- Component initialization data loading
- Service-based reactive data
- Simple API calls without user concurrency concerns

**Use route model hook for:**
- First-level components only (direct children of route template)
- Avoid for deeply nested components (causes prop drilling)

**Use URL state + service for:**
- Shareable/bookmarkable filtered views
- Query params driving service state
- Deep component trees needing reactive data

**Use ember-concurrency for:**
- User input debouncing (search, autocomplete)
- Form submission (prevent double-click with `dropTask`)
- Polling (user-controlled refresh intervals)
- Multi-step wizards (sequential async operations)

**Key Principles:**
1. **Derive data, don't set it** - Use `task.lastSuccessful.value`, never set `@tracked` inside tasks
2. **Return values from tasks** - Let the TaskInstance API manage state
3. **Choose the right modifier**: `restartableTask` for search, `dropTask` for submit, `enqueueTask` for queues

**When NOT to use ember-concurrency:**
- Component initialization data loading (use `getPromiseState`)
- Setting tracked state inside tasks (causes render loops)
- Route model hooks (return promises directly)

References:
- [ember-concurrency](https://ember-concurrency.com/)
- [TaskInstance API](https://ember-concurrency.com/api/TaskInstance.html)
- [reactiveweb](https://github.com/universal-ember/reactiveweb)
