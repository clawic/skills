# Forms — Inputs, Validation, and Submission

Three independent decisions, usually conflated: how the value is bound, when validation runs, and what happens between submit and response. Most form bugs live in the third.

## v-model on Native Inputs

| Element | Binds | Watch out |
|---|---|---|
| `text`, `textarea` | `value` + `input` | `.lazy` switches to the `change` event (blur) — the fix for expensive per-keystroke work |
| `number` | `value` + `input` | Without `.number` you get a string; with it, an empty field gives `''`, not `0`, and a partial `"1e"` gives the raw string |
| `checkbox` (single) | `checked` | `true-value` / `false-value` bind non-boolean pairs (`'yes'`/`'no'`) |
| `checkbox` (group) | array of values | The bound ref must already be an array, or the first click replaces it with a boolean |
| `radio` | `checked` by `value` | All radios in a group share one ref and need distinct `value`s |
| `select` | `value`, or array with `multiple` | Binding objects requires `:value="obj"`, and equality is by reference — the selected option must be the same object from the list |
| `file` | nothing | `v-model` on a file input is a no-op: `value` is read-only. Use `@change="e => files = e.target.files"` |
| `contenteditable` | nothing | No v-model; handle `input` and write the DOM yourself, guarding against cursor reset |

- Coerce at the boundary, not at the point of use: `const age = computed({ get: () => raw.value, set: v => raw.value = v === '' ? null : Number(v) })` keeps the rest of the form typed.
- IME composition (Japanese, Chinese, Korean input): `v-model` skips updates mid-composition by design; a hand-rolled `@input` handler does not, and produces garbled text.

## v-model on Components

```ts
// Child
const model = defineModel<string>({ required: true })              // vue >=3.4
const [name, nameModifiers] = defineModel<string>('name', {
  set: (v) => nameModifiers.trim ? v.trim() : v
})
```

- Below `vue 3.4`: declare the `modelValue` prop and emit `update:modelValue` (`components.md`).
- Custom modifiers arrive in `<name>Modifiers` — `v-model.trim="x"` on your component does nothing unless you implement it.
- A wrapper input must forward attributes to the inner element: `defineOptions({ inheritAttrs: false })` plus `v-bind="$attrs"` on the `<input>`, otherwise `placeholder` and `required` land on the wrapper div.
- Pass through `id` and label association explicitly. `useId()` (`vue >=3.5`) generates SSR-stable ids for label/`aria-describedby` pairs; `Math.random()` in a template guarantees a hydration mismatch.

## Validation Timing

The rule that avoids both hostile and useless forms: **validate a field on blur after the first interaction, re-validate on input only once it is already invalid, and validate everything on submit.**

- Validating on every keystroke from an empty field tells the user "required" while they are typing the first letter.
- Validating only on submit hides three errors until the end and scrolls them back to the top.
- Track `touched` per field (set on first blur) and `submitted` for the form; an error renders when `(touched && invalid) || submitted`.
- Errors belong next to the field, with `aria-describedby` linking input to message and `aria-invalid` on the input. A summary at the top is an addition for long forms, not a replacement.

## Schema Validation

```ts
const schema = z.object({ email: z.string().email(), age: z.number().min(18) })
const result = schema.safeParse(values)         // never throw on user input
errors.value = result.success ? {} : result.error.flatten().fieldErrors
```

- One schema, two uses: client feedback and the server's own check. The server must validate independently — client validation is UX, never a security control (`security.md`).
- Field-level and form-level rules are different shapes: cross-field rules ("end date after start date") need the whole object, so run them at form level and attach the message to the field the user should fix.
- A library (`vee-validate`, `@vueuse/core` form helpers, a schema resolver) earns its place once you need arrays of fields, nested objects, or async uniqueness checks. Below that, a `computed` returning an error map is less code than the library's setup.

## Submission

```ts
async function submit() {
  if (isSubmitting.value) return                 // the double-click guard
  submitted.value = true
  if (!isValid.value) return focusFirstError()
  isSubmitting.value = true
  try {
    await api.save(payload.value)
    isDirty.value = false                        // release the leave guard BEFORE navigating
    router.push({ name: 'done' })
  } catch (e) {
    serverErrors.value = mapServerErrors(e)      // field-level where possible
  } finally {
    isSubmitting.value = false                   // in finally: clears even on throw
  }
}
```

- Disable the submit button while `isSubmitting`, and guard in the handler too — a keyboard `Enter` bypasses a visually disabled button in some browsers, and a slow network invites the second click.
- Map server errors back onto fields. A generic red banner for "email already taken" makes the user hunt for the field.
- Make the request idempotent (client-generated request id) if a duplicate submission would create two records; guards reduce the odds, networks retry anyway.

## Unsaved Changes

```ts
onBeforeRouteLeave(() => !isDirty.value || confirm('Discard unsaved changes?'))
useEventListener(window, 'beforeunload', (e) => { if (isDirty.value) e.preventDefault() })
```

- Two mechanisms are required: the router guard covers in-app navigation, `beforeunload` covers tab close and reload. Neither covers the other.
- Browsers ignore custom `beforeunload` text and show their own; do not spend effort on the wording.
- Compute `isDirty` by comparing against the loaded snapshot, not by setting a flag in every handler — a flag misses programmatic changes and never resets correctly.

## Dynamic and Repeating Fields

- Key rows by a generated id, never by index: removing row 2 with index keys leaves row 3's input value displayed in row 2's slot (SKILL.md rule 7).
- Keep the array of values and the array of errors keyed by the same id, or the two drift after a removal.
- Focus management on add: append the row, `await nextTick()`, then focus its first input — before the tick the element does not exist.

## Accessibility Minimums

- Every input has a `<label for>` or an `aria-label`; a placeholder is not a label and disappears on focus.
- Group radios and related checkboxes in a `<fieldset>` with a `<legend>`.
- On failed submit, move focus to the first invalid field — screen reader users otherwise get no signal that anything happened.
- Never rely on color alone for the error state; the message text carries it.

## Form Review Checks

- Does every field survive a page refresh mid-edit, or is losing the draft acceptable and stated?
- Is the submit button guarded against double submission in the handler, not only visually?
- Does the server validate the same rules independently?
- Do errors appear next to fields with `aria-describedby`, and does focus move on failed submit?
- Do dynamic rows use stable ids as keys?
