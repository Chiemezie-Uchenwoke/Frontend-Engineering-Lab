# Accessible Forms in Practice

> **Topic:** Accessibility · **Level:** Intermediate · **Author:** [@davex-ai](https://github.com/davex-ai)

## The Problem

Forms are where most accessibility issues live, and most frontend engineers interact with accessibility daily without realizing it. Every sign-up, checkout, and settings page is a form, and most of them are subtly broken for screen reader and keyboard users: labels that aren't really labels, error messages that appear visually but are never announced, and focus that gets lost the moment something goes wrong.

The frustrating part is that none of this requires exotic ARIA knowledge. It requires treating four specific moments in the form lifecycle as first-class concerns: labeling, error announcement, focus management, and keyboard navigation.

## The Pattern

### 1. Labels are not placeholders

A `placeholder` is not a label. It disappears the moment the user starts typing, it's rendered with low-contrast styling by default (a contrast failure waiting to happen), and depending on the screen reader and browser combination, it may not be announced consistently when the field receives focus. A `<label>` persists, is always announced, and is programmatically tied to its input.

```tsx
// ❌ Looks fine, fails for screen reader and low-vision users
<input name="email" type="email" placeholder="Email address" />

// ✅ Persistent, programmatically associated
<label htmlFor="email">Email address</label>
<input id="email" name="email" type="email" />
```

If you want the compact look of a placeholder-only input, keep the `<label>` in the DOM and visually hide it with a `.sr-only` utility class rather than removing it — never use `display: none` or `visibility: hidden`, since both remove the element from the accessibility tree too.

### 2. Announcing validation errors

A red border and a paragraph of red text is a purely visual signal. To make it work for screen readers, the input needs to point to its own error message via `aria-describedby`, and the field itself should be marked `aria-invalid` while the error is active.

```tsx
type FormErrors = {
  email?: string;
  password?: string;
};

function SignUpForm() {
  const [errors, setErrors] = useState<FormErrors>({});
  const emailRef = useRef<HTMLInputElement>(null);
  const passwordRef = useRef<HTMLInputElement>(null);

  const validate = (data: FormData): FormErrors => {
    const newErrors: FormErrors = {};
    const email = data.get('email') as string;
    const password = data.get('password') as string;

    if (!email.includes('@')) {
      newErrors.email = 'Enter a valid email address.';
    }
    if (password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters.';
    }
    return newErrors;
  };

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    const newErrors = validate(data);
    setErrors(newErrors);

    // Focus management — see section 3
    if (newErrors.email) {
      emailRef.current?.focus();
    } else if (newErrors.password) {
      passwordRef.current?.focus();
    }
  };

  return (
    <form onSubmit={handleSubmit} noValidate>
      {Object.keys(errors).length > 0 && (
        <div role="alert" className="error-summary">
          There are {Object.keys(errors).length} errors in this form.
        </div>
      )}

      <label htmlFor="email">Email</label>
      <input
        id="email"
        name="email"
        type="email"
        ref={emailRef}
        aria-invalid={!!errors.email}
        aria-describedby={errors.email ? 'email-error' : undefined}
      />
      {errors.email && (
        <p id="email-error" role="alert">
          {errors.email}
        </p>
      )}

      <label htmlFor="password">Password</label>
      <input
        id="password"
        name="password"
        type="password"
        ref={passwordRef}
        aria-invalid={!!errors.password}
        aria-describedby={errors.password ? 'password-error' : undefined}
      />
      {errors.password && (
        <p id="password-error" role="alert">
          {errors.password}
        </p>
      )}

      <button type="submit">Create account</button>
    </form>
  );
}
```

A few details worth calling out:

- `role="alert"` implicitly creates a live region — it doesn't need a separate `aria-live` attribute. It interrupts the screen reader to announce its contents, which is appropriate for "you just tried to submit and it failed."
- `aria-describedby` only needs to be present when there's actually an error. Leaving it `undefined` rather than pointing to a non-existent or empty element avoids screen readers announcing nothing.
- `noValidate` on the `<form>` disables the browser's native validation bubbles so your custom error UI is the only thing the user sees — otherwise you can end up with both competing for attention.

### 3. Focus management on submit failure

This is the step most forms skip entirely. If a sighted mouse user submits a broken form, they scan the page and spot the red text. A screen reader user has no equivalent "scan" — if focus stays on the submit button, they have no idea anything went wrong unless something is announced and focus is moved to where they need to act.

The pattern: on a failed submit, move focus either to the first invalid field, or to an error summary if there are several errors and you want to give an overview first. The example above does the former — `emailRef.current?.focus()` runs synchronously inside the failed-submit branch, so the next thing the screen reader announces is the email field's label, its `aria-invalid` state, and its associated error.

### 4. The keyboard-only walkthrough

Before calling a form "accessible," unplug your mouse and tab through it:

- Does tab order follow the visual order of the form?
- Is the focus indicator visible on every interactive element (don't `outline: none` without a replacement)?
- Can you submit with Enter from any text field, not just by clicking the button?
- If you open a date picker, dropdown, or modal from within the form, can you close it and return focus to where you were, without getting trapped?

This costs five minutes and catches a surprising fraction of real-world issues that automated linting tools miss entirely.

## Tradeoffs

- **`role="alert"` vs `aria-live="polite"`** — `alert` is assertive and interrupts immediately, which is right for "submission failed." For non-critical, ongoing feedback (like a live character counter), `aria-live="polite"` is less disruptive since it waits for a pause before announcing.
- **Native HTML5 validation vs custom validation** — native `required`/`pattern` attributes are zero-JS and accessible by default, but the error styling and messaging are inconsistent across browsers and hard to localize or style precisely. Custom validation (as above) gives full control but means you own the accessibility wiring yourself.
- **Focusing the first field vs an error summary** — focusing the first invalid field is faster for one or two errors. For longer forms with many errors, an error summary with links to each field (a pattern used by GOV.UK and other accessibility-mature teams) gives the user an overview before they start fixing things one at a time.

## Key Takeaways

- A `placeholder` is never a substitute for a `<label>` — associate them with `htmlFor`/`id`, and visually hide the label if needed rather than removing it.
- Use `aria-describedby` to link an input to its error message, and `aria-invalid` to flag the field's state, so screen readers announce both together.
- On a failed submit, actively move focus to the first error or an error summary — don't leave the screen reader user stranded on the submit button.
- Use assertive announcements (`role="alert"`) for failures, and reserve `aria-live="polite"` for non-critical updates.
- Test every form keyboard-only before considering it done. It's the cheapest accessibility test you can run and it catches real bugs.

## References

**Standards & specifications**
- [WCAG 2.2 Success Criterion 3.3.1 — Error Identification](https://www.w3.org/WAI/WCAG22/Understanding/error-identification.html)
- [WCAG 2.2 Success Criterion 3.3.2 — Labels or Instructions](https://www.w3.org/WAI/WCAG22/Understanding/labels-or-instructions.html)
- [WCAG 2.2 Success Criterion 4.1.2 — Name, Role, Value](https://www.w3.org/WAI/WCAG22/Understanding/name-role-value.html)
- [WCAG 2.2 Success Criterion 2.4.7 — Focus Visible](https://www.w3.org/WAI/WCAG22/Understanding/focus-visible.html)
- [WAI-ARIA Authoring Practices Guide (APG) — Forms](https://www.w3.org/WAI/ARIA/apg/practices/forms/)
- [WAI-ARIA 1.2 Specification — `alert` role](https://www.w3.org/TR/wai-aria-1.2/#alert)

**API references**
- [MDN — `<label>`: The Label element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/label)
- [MDN — `aria-describedby`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-describedby)
- [MDN — `aria-invalid`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-invalid)
- [MDN — ARIA `alert` role](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/alert_role)
- [MDN — `aria-live`](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Attributes/aria-live)

**Established patterns & further reading**
- [GOV.UK Design System — Error summary pattern](https://design-system.service.gov.uk/components/error-summary/)
- [GOV.UK Design System — Error messages pattern](https://design-system.service.gov.uk/components/error-message/)
- [WebAIM — Creating Accessible Forms](https://webaim.org/techniques/forms/)
- [WebAIM — Usable and Accessible Form Validation and Error Recovery](https://webaim.org/techniques/formvalidation/)
- [Deque University — Focus Management](https://dequeuniversity.com/checklists/web/javascript)

