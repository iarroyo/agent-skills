---
title: Use Helper Functions for Reusable Logic
impact: LOW-MEDIUM
impactDescription: Better code reuse and testability
tags: helpers, templates, reusability, advanced
---

## Use Helper Functions for Reusable Logic

Extract reusable template logic into helper functions that can be tested independently and used across templates.

**Incorrect (logic duplicated in components):**

```javascript
// app/components/user-card.js
class UserCard extends Component {
  get formattedDate() {
    const date = new Date(this.args.user.createdAt);
    const now = new Date();
    const diffMs = now - date;
    const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
    
    if (diffDays === 0) return 'Today';
    if (diffDays === 1) return 'Yesterday';
    if (diffDays < 7) return `${diffDays} days ago`;
    return date.toLocaleDateString();
  }
}

// app/components/post-card.js - same logic duplicated!
class PostCard extends Component {
  get formattedDate() {
    // Same implementation...
  }
}
```

**Correct (reusable helper - co-located with component):**

```javascript
// app/components/post-list/format-relative-date.js
// Co-locate with the component that uses it for better organization
import { helper } from '@ember/component/helper';

function formatRelativeDate([date]) {
  const dateObj = new Date(date);
  const now = new Date();
  const diffMs = now - dateObj;
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
  
  if (diffDays === 0) return 'Today';
  if (diffDays === 1) return 'Yesterday';
  if (diffDays < 7) return `${diffDays} days ago`;
  return dateObj.toLocaleDateString();
}

export default helper(formatRelativeDate);
```

**Alternative (shared helper in utils):**

```javascript
// app/utils/helpers/format-relative-date.js
// Use utils/ directory for helpers shared across many components
import { helper } from '@ember/component/helper';

function formatRelativeDate([date]) {
  const dateObj = new Date(date);
  const now = new Date();
  const diffMs = now - dateObj;
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
  
  if (diffDays === 0) return 'Today';
  if (diffDays === 1) return 'Yesterday';
  if (diffDays < 7) return `${diffDays} days ago`;
  return dateObj.toLocaleDateString();
}

export default helper(formatRelativeDate);
```

**Note**: The `app/helpers/` directory has a smaller role in modern Ember. Prefer co-locating helpers with components for better modularity, or use `app/utils/` for truly shared helpers.

```javascript
// app/components/user-card.gjs
import { formatRelativeDate } from '../helpers/format-relative-date';

<template>
  <p>Joined: {{formatRelativeDate @user.createdAt}}</p>
</template>
```

```javascript
// app/components/post-card.gjs
import { formatRelativeDate } from '../helpers/format-relative-date';

<template>
  <p>Posted: {{formatRelativeDate @post.createdAt}}</p>
</template>
```

**For helpers with state, use class-based helpers:**

```javascript
// app/helpers/format-currency.js
import Helper from '@ember/component/helper';
import { service } from '@ember/service';

export default class FormatCurrencyHelper extends Helper {
  @service intl;
  
  compute([amount], { currency = 'USD' }) {
    return this.intl.formatNumber(amount, {
      style: 'currency',
      currency
    });
  }
}
```

**Common helpers to create:**
- Date/time formatting
- Number formatting
- String manipulation
- Array operations
- Conditional logic

Helpers promote code reuse, are easier to test, and keep components focused on behavior.

Reference: [Ember Helpers](https://guides.emberjs.com/release/components/helper-functions/)
