---
title: Use Resources for Declarative Data Management
impact: HIGH
impactDescription: Better lifecycle management and reactivity
tags: resources, lifecycle, data-management, declarative
---

## Use Resources for Declarative Data Management

Use ember-resources for declarative data management with automatic cleanup and lifecycle management instead of manual imperative code.

**Incorrect (manual lifecycle management):**

```javascript
// app/components/live-data.gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

class LiveData extends Component {
  @tracked data = null;
  intervalId = null;
  
  constructor() {
    super(...arguments);
    this.fetchData();
    this.intervalId = setInterval(() => this.fetchData(), 5000);
  }
  
  async fetchData() {
    const response = await fetch('/api/data');
    this.data = await response.json();
  }
  
  willDestroy() {
    super.willDestroy(...arguments);
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }

  <template>
    <div>{{this.data}}</div>
  </template>
}
```

**Correct (using Resources - preferred pattern):**

```javascript
// app/components/live-data.gjs
import Component from '@glimmer/component';
import { resource, use } from 'ember-resources';

// Define resource outside component class for better composition
const liveData = resource(({ on }) => {
  const poll = async () => {
    const response = await fetch('/api/data');
    return response.json();
  };
  
  const intervalId = setInterval(poll, 5000);
  
  // Automatic cleanup
  on.cleanup(() => clearInterval(intervalId));
  
  return poll();
});

class LiveData extends Component {
  // Use the resource in component via @use decorator or directly
  data = use(this, liveData);

  <template>
    <div>{{this.data.value}}</div>
  </template>
}
```

**For tracked resources with arguments:**

```javascript
// app/components/user-profile.gjs
import Component from '@glimmer/component';
import { resource, resourceFactory } from 'ember-resources';

const UserData = resourceFactory((userId) => 
  resource(async ({ on }) => {
    const controller = new AbortController();
    
    on.cleanup(() => controller.abort());
    
    const response = await fetch(`/api/users/${userId}`, {
      signal: controller.signal
    });
    
    return response.json();
  })
);

class UserProfile extends Component {
  userData = UserData(() => this.args.userId);

  <template>
    {{#if this.userData.value}}
      <h1>{{this.userData.value.name}}</h1>
    {{/if}}
  </template>
}
```

Resources provide automatic cleanup, prevent memory leaks, and offer better composition patterns.

Reference: [ember-resources](https://github.com/NullVoxPopuli/ember-resources)
