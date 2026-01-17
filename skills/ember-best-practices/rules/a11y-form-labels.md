---
title: Form Labels and Error Announcements
impact: HIGH
impactDescription: Essential for screen reader users
tags: accessibility, a11y, forms, aria-live
---

## Form Labels and Error Announcements

All form inputs must have associated labels, and validation errors should be announced to screen readers using ARIA live regions.

**Incorrect (missing labels and announcements):**

```javascript
// app/components/form.gjs
<template>
  <form {{on "submit" this.handleSubmit}}>
    <input 
      type="email" 
      value={{this.email}}
      {{on "input" this.updateEmail}}
      placeholder="Email"
    />
    
    {{#if this.emailError}}
      <span class="error">{{this.emailError}}</span>
    {{/if}}
    
    <button type="submit">Submit</button>
  </form>
</template>
```

**Correct (with labels and announcements):**

```javascript
// app/components/form.gjs
<template>
  <form {{on "submit" this.handleSubmit}}>
    <div class="form-group">
      <label for="email-input">
        Email Address
        {{#if this.isEmailRequired}}
          <span aria-label="required">*</span>
        {{/if}}
      </label>
      
      <input 
        id="email-input"
        type="email" 
        value={{this.email}}
        {{on "input" this.updateEmail}}
        aria-describedby={{if this.emailError "email-error"}}
        aria-invalid={{if this.emailError "true"}}
        required={{this.isEmailRequired}}
      />
      
      {{#if this.emailError}}
        <span 
          id="email-error" 
          class="error"
          role="alert"
          aria-live="polite"
        >
          {{this.emailError}}
        </span>
      {{/if}}
    </div>
    
    <button type="submit" disabled={{this.isSubmitting}}>
      {{#if this.isSubmitting}}
        <span aria-live="polite">Submitting...</span>
      {{else}}
        Submit
      {{/if}}
    </button>
  </form>
</template>
```

**For complex forms, use ember-changeset-validations:**

```javascript
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { Changeset } from 'ember-changeset';
import lookupValidator from 'ember-changeset-validations';
import { validatePresence, validateFormat } from 'ember-changeset-validations/validators';

const UserValidations = {
  email: [
    validatePresence({ presence: true, message: 'Email is required' }),
    validateFormat({ type: 'email', message: 'Must be a valid email' })
  ]
};

export default class UserFormComponent extends Component {
  changeset = Changeset(this.args.user, lookupValidator(UserValidations), UserValidations);
  
  @action
  async handleSubmit(event) {
    event.preventDefault();
    await this.changeset.validate();
    
    if (this.changeset.isValid) {
      await this.args.onSubmit(this.changeset);
    }
  }
}
```

Always associate labels with inputs and announce dynamic changes to screen readers using aria-live regions.

Reference: [Ember Accessibility - Application Considerations](https://guides.emberjs.com/release/accessibility/application-considerations/)
