---
title: Announce Route Transitions to Screen Readers
impact: HIGH
impactDescription: Critical for screen reader navigation
tags: accessibility, a11y, routing, screen-readers
---

## Announce Route Transitions to Screen Readers

Announce page title changes and route transitions to screen readers so users know when navigation has occurred.

**Incorrect (no announcements):**

```javascript
// app/router.js
export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
}
```

**Correct (with route announcements using ember-a11y):**

```bash
ember install ember-a11y
```

```javascript
// app/router.js
import EmberRouter from '@ember/routing/router';
import config from './config/environment';

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
}

Router.map(function() {
  this.route('about');
  this.route('dashboard');
  this.route('posts', function() {
    this.route('post', { path: '/:post_id' });
  });
});
```

```javascript
// app/routes/application.js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';

export default class ApplicationRoute extends Route {
  @service router;
  
  constructor() {
    super(...arguments);
    
    this.router.on('routeDidChange', (transition) => {
      // Update document title
      const title = this.getPageTitle(transition.to);
      document.title = title;
      
      // Announce to screen readers
      this.announceRouteChange(title);
    });
  }
  
  getPageTitle(route) {
    // Get title from route metadata or generate it
    return route.metadata?.title || route.name;
  }
  
  announceRouteChange(title) {
    const announcement = document.getElementById('route-announcement');
    if (announcement) {
      announcement.textContent = `Navigated to ${title}`;
    }
  }
}
```

```javascript
// app/routes/application.gjs
<template>
  <div 
    id="route-announcement" 
    role="status" 
    aria-live="polite" 
    aria-atomic="true"
    class="sr-only"
  ></div>

  {{outlet}}
</template>
```

```css
/* app/styles/app.css */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}
```

**Alternative: Use ember-page-title with announcements:**

```bash
ember install ember-page-title
```

```javascript
// app/routes/dashboard.gjs
import { pageTitle } from 'ember-page-title';

<template>
  {{pageTitle "Dashboard"}}

  <div class="dashboard">
    {{outlet}}
  </div>
</template>
```

Route announcements ensure screen reader users know when navigation occurs, improving the overall accessibility experience.

Reference: [Ember Accessibility - Page Titles](https://guides.emberjs.com/release/accessibility/page-template-considerations/)
