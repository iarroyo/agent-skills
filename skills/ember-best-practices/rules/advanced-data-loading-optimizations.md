---
title: Advanced Data Loading Optimizations
impact: MEDIUM
impactDescription: Improves perceived performance and offline capabilities
tags: caching, service-worker, workbox, optimistic-ui, performance
---

## Advanced Data Loading Optimizations

This guide covers advanced caching and optimization strategies that complement the core data loading patterns:

- **Multi-layer caching**: Combining Service Workers with application-level caching
- **Optimistic UI**: Updating the UI before server confirmation
- **Cross-tab synchronization**: Keeping multiple tabs in sync

### Application Cache vs Service Worker Cache

These serve different purposes at different layers - they're complementary, not mutually exclusive.

#### Application-level Caching Service

**Pros:**
- Fine-grained control over cache invalidation (after mutations, on user actions)
- Reactive state integration with `getPromiseState`
- Request deduplication for simultaneous consumers
- Business logic awareness (knows when data is stale semantically)
- Works with any data transformation/normalization

**Cons:**
- Only works while the app is running (memory-based)
- Doesn't help with offline or slow networks
- You maintain the logic yourself

#### Workbox / Service Worker Caching

**Pros:**
- Works offline
- Persists across page reloads
- Network-level caching (faster than hitting the server)
- Strategies like stale-while-revalidate for perceived performance
- Handles static assets too

**Cons:**
- Harder to invalidate precisely (cache by URL, not by business logic)
- No awareness of your app's reactive state
- Doesn't deduplicate in-flight requests within the app
- Debugging can be tricky

### Multi-layer Caching Strategy

Use both layers together: Workbox for network-level caching and offline support, application service for request deduplication and business-logic-aware invalidation.

#### Layer 1: Workbox (Service Worker)

```js
// service-worker.js
import { registerRoute } from 'workbox-routing';
import { StaleWhileRevalidate } from 'workbox-strategies';

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/users/'),
  new StaleWhileRevalidate({
    cacheName: 'api-users'
  })
);
```

What Workbox handles:
- First visit to `/api/users/123` → network request, response cached
- Return visit (next day) → instantly returns cached response, fetches fresh copy in background
- Offline → returns cached response

What Workbox doesn't know:
- That 5 components want the same data right now
- That the user just updated their name
- That the cache should be invalidated after a mutation

#### Layer 2: Application Service

```js
// app/services/user-data.js
import Service from '@ember/service';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

export default class UserDataService extends Service {
  #requests = new Map();

  getUser(userId) {
    // Deduplication: 5 components call this, only 1 fetch happens
    if (!this.#requests.has(userId)) {
      const promise = fetch(`/api/users/${userId}`).then(r => r.json());
      this.#requests.set(userId, promise);
    }
    return this.#requests.get(userId);
  }

  getUserState(userId) {
    return getPromiseState(this.getUser(userId));
  }

  async updateUser(userId, data) {
    await fetch(`/api/users/${userId}`, {
      method: 'PATCH',
      body: JSON.stringify(data)
    });

    // Business logic: invalidate after mutation
    this.#requests.delete(userId);

    // Also bust Workbox cache for this specific user
    const cache = await caches.open('api-users');
    await cache.delete(`/api/users/${userId}`);
  }
}
```

#### Timeline Example

Scenario: User profile page with `UserHeader`, `UserSidebar`, and `UserActivity` components all needing user data.

| Time | Event | Workbox | App Service |
|------|-------|---------|-------------|
| 0ms | User lands on profile page | — | — |
| 1ms | `UserHeader` requests user 123 | Cache miss → network | Creates promise, stores in Map |
| 2ms | `UserSidebar` requests user 123 | — | Returns same promise (no new fetch) |
| 3ms | `UserActivity` requests user 123 | — | Returns same promise (no new fetch) |
| 200ms | Network responds | Caches response | Promise resolves, all 3 components update |
| 5min | User navigates away, comes back | Returns cached (instant) | New promise, but Workbox serves from cache |
| 6min | User edits their name | — | — |
| 6min | `updateUser()` called | Cache entry deleted | Map entry deleted |
| 6min | Components re-fetch | Cache miss → network | New promise created |

#### Responsibility Matrix

| Concern | Workbox | App Service |
|---------|---------|-------------|
| Offline support | Yes | No |
| Instant load on return visit | Yes | No |
| Deduplicate simultaneous requests | No | Yes |
| Reactive state (`getPromiseState`) | No | Yes |
| Invalidate after mutation | Manual | Built-in |
| Knows about your components | No | Yes |

### Optimistic UI Updates

Instead of invalidating cache and waiting for refetch, update the UI immediately and sync in the background.

#### The Flow

```
User clicks "Save"
    ↓
1. Optimistically update local state (instant UI update)
    ↓
2. Update SW cache with new data (other tabs get it too)
    ↓
3. POST to server in background
    ↓
4. If success → stale-while-revalidate ensures eventual consistency
   If failure → rollback local state, show error
```

#### Implementation

```js
// app/services/user-data.js
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';
import { getPromiseState } from 'reactiveweb';

export default class UserDataService extends Service {
  #requests = new Map();
  @tracked #optimisticUpdates = new Map();

  getUserState(userId) {
    const state = getPromiseState(this.getUser(userId));

    // Apply optimistic update if present
    const optimistic = this.#optimisticUpdates.get(userId);
    if (optimistic && state.isFulfilled) {
      return {
        ...state,
        value: { ...state.value, ...optimistic.data }
      };
    }

    return state;
  }

  async updateUserOptimistic(userId, data) {
    // 1. Store previous state for rollback
    const previousData = this.#optimisticUpdates.get(userId)?.data;

    // 2. Apply optimistic update immediately
    this.#optimisticUpdates.set(userId, { data, pending: true });
    this.#optimisticUpdates = new Map(this.#optimisticUpdates); // Trigger reactivity

    // 3. Update SW cache so other tabs get it
    await this.updateSwCache(userId, data);

    // 4. Notify other tabs
    this.broadcastUpdate(userId);

    try {
      // 5. Send to server
      const response = await fetch(`/api/users/${userId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });

      if (!response.ok) {
        throw new Error('Update failed');
      }

      // 6. Success - clear optimistic state, refetch for server truth
      this.#optimisticUpdates.delete(userId);
      this.#requests.delete(userId);

    } catch (error) {
      // 7. Rollback on failure
      if (previousData) {
        this.#optimisticUpdates.set(userId, { data: previousData, pending: false });
      } else {
        this.#optimisticUpdates.delete(userId);
      }
      this.#optimisticUpdates = new Map(this.#optimisticUpdates);

      // Restore SW cache
      await this.invalidateSwCache(userId);

      throw error;
    }
  }

  async updateSwCache(userId, data) {
    const cache = await caches.open('api-users');
    const url = `/api/users/${userId}`;

    // Get existing cached response
    const existingResponse = await cache.match(url);
    if (existingResponse) {
      const existingData = await existingResponse.json();
      const updatedData = { ...existingData, ...data };

      // Store updated data in cache
      await cache.put(url, new Response(JSON.stringify(updatedData), {
        headers: { 'Content-Type': 'application/json' }
      }));
    }
  }

  async invalidateSwCache(userId) {
    const cache = await caches.open('api-users');
    await cache.delete(`/api/users/${userId}`);
  }

  broadcastUpdate(userId) {
    // Notify other tabs to refetch from updated SW cache
    const channel = new BroadcastChannel('user-updates');
    channel.postMessage({ type: 'USER_UPDATED', userId });
    channel.close();
  }
}
```

#### Listening for Cross-tab Updates

```js
// app/services/user-data.js (add to constructor)
constructor() {
  super(...arguments);

  const channel = new BroadcastChannel('user-updates');
  channel.onmessage = (event) => {
    if (event.data.type === 'USER_UPDATED') {
      // Invalidate local cache, SW cache already has new data
      this.#requests.delete(event.data.userId);
    }
  };
}
```

#### Component Usage

```glimmer-js
// app/components/user-profile-form.gjs
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

class UserProfileForm extends Component {
  @service userData;
  @tracked error = null;
  @tracked saving = false;

  @action
  async save(event) {
    event.preventDefault();
    const formData = new FormData(event.target);

    this.saving = true;
    this.error = null;

    try {
      await this.userData.updateUserOptimistic(this.args.userId, {
        name: formData.get('name'),
        email: formData.get('email')
      });
      // UI already updated optimistically - no loading state visible to user
    } catch (e) {
      this.error = 'Failed to save. Please try again.';
      // UI already rolled back
    } finally {
      this.saving = false;
    }
  }

  <template>
    <form {{on "submit" this.save}}>
      {{#if this.error}}
        <div class="error">{{this.error}}</div>
      {{/if}}

      <input name="name" value={{this.userData.getUserState(@userId).value.name}} />
      <input name="email" value={{this.userData.getUserState(@userId).value.email}} />

      <button type="submit" disabled={{this.saving}}>
        Save
      </button>
    </form>
  </template>
}
```

### When to Use Optimistic UI

**Good candidates:**

| Scenario | Why |
|----------|-----|
| User-owned data | Low conflict probability |
| Profile updates, preferences | Single user editing their own data |
| Toggles (favorite, read status) | Simple, reversible operations |
| Draft saving | Non-critical, can retry |

**Avoid for:**

| Scenario | Why |
|----------|-----|
| Collaborative/shared data | Someone else might have changed it |
| Server-side validation | Server might reject or transform data |
| Financial transactions | Must confirm before showing success |
| Inventory/quantity changes | Race conditions with other users |
| Complex dependencies | Computed values may differ from optimistic guess |

### Tradeoffs to Consider

| Concern | Simple Invalidate + Refetch | Optimistic UI |
|---------|----------------------------|---------------|
| Perceived performance | Loading state after save | Instant feedback |
| Implementation complexity | Simple | More complex |
| Error handling | Straightforward | Requires rollback logic |
| Data consistency | Always server truth | Temporary optimistic state |
| User expectations | "Saving..." is understood | Failure after "success" is jarring |
| Debugging | Easy to trace | Harder to debug state issues |

### Recommendation

Use optimistic UI selectively. Don't build a generic "optimistic everything" system - that gets complex fast. Instead:

1. Identify specific interactions where latency hurts UX (toggling a favorite, updating a name)
2. Implement optimistic updates just for those cases
3. Keep simple invalidate + refetch for most CRUD operations

Users understand a brief loading state after clicking "Save." Reserve optimistic UI for interactions where instant feedback significantly improves the experience.

References:
- [Workbox](https://developer.chrome.com/docs/workbox/)
- [BroadcastChannel API](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)
- [Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
