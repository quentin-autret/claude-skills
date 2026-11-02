# Form elements, common props and events

## Contents

- Controlled and uncontrolled inputs
- select, option, textarea, progress
- Buttons and FormData
- Common props
- Events
- Custom elements

## Controlled and uncontrolled inputs

- `value` (or `checked`) makes an input controlled and requires an `onChange` that updates it synchronously. `defaultValue` (or `defaultChecked`) leaves it uncontrolled. https://react.dev/reference/react-dom/components/input
- `value` without `onChange` renders read-only and logs an error. Add `onChange`, switch to `defaultValue`, or set `readOnly`. https://react.dev/reference/react-dom/components/input
- An input never switches between controlled and uncontrolled. A controlled text input always gets a string: `value={someValue ?? ''}`. A checkbox always gets a boolean. https://react.dev/reference/react-dom/components/input
- Checkboxes use `checked` and `e.target.checked`, never `value`. https://react.dev/reference/react-dom/components/input
- Caret jumps to the start on each keystroke: set state to exactly `e.target.value`, synchronously (no `toUpperCase()`, no `setTimeout`). Otherwise the input remounts through a changing `key` or a component defined inside another. https://react.dev/reference/react-dom/components/input

```js
function handleChange(e) {
  setFirstName(e.target.value);
}
```

https://react.dev/reference/react-dom/components/input

- `onChange` fires on every keystroke, like the native `input` event: prefer it to `onInput`. https://react.dev/reference/react-dom/components/input
- A Transition cannot control a text input. https://react.dev/reference/react/useTransition

## select, option, textarea, progress

- `selected` on `<option>` is not supported. Put the value on `<select value>` or `<select defaultValue>`. https://react.dev/reference/react-dom/components/option
- With `multiple={true}`, `value` and `defaultValue` are arrays of strings. https://react.dev/reference/react-dom/components/select
- `Object.fromEntries(formData.entries())` drops repeated values of a multiple select. Read `[...formData.entries()]`. https://react.dev/reference/react-dom/components/select
- `<textarea>` takes no children: use `defaultValue`. Same controlled rules as `<input>`. https://react.dev/reference/react-dom/components/textarea
- `<progress>`: `max` defaults to 1, `value={null}` is indeterminate. https://react.dev/reference/react-dom/components/progress

## Buttons and FormData

- A `<button>` without `type` inside a `<form>` submits it. Custom button components set `type="button"` or `type="submit"` explicitly. https://react.dev/reference/react-dom/components/input
- Give every field a `name`, or it is missing from `FormData`. https://react.dev/reference/react-dom/components/input
- `formAction` on `<input type="submit">` or `type="image"` overrides the form `action`. https://react.dev/reference/react-dom/components/input

## Common props

- `dangerouslySetInnerHTML={{ __html }}` is an XSS risk with untrusted HTML and cannot be combined with `children`. A `TrustedHTML` value passes through unchanged: your Trusted Types policy must still sanitize. https://react.dev/reference/react-dom/components/common
- React passes `TrustedHTML`, `TrustedScript` and `TrustedScriptURL` without string coercion, compatible with `require-trusted-types-for 'script'`. https://react.dev/blog/2026/09/09/react-19-3
- `javascript:` URLs in `src` and `href` are an error. An empty string `src` or `href` is warned and not set, anchors excepted. https://react.dev/blog/2024/04/25/react-19-upgrade-guide
- Prefer `className` to `style`, which is for dynamic values. Numbers in `style` get `px` except unitless properties. https://react.dev/reference/react-dom/components/common
- `suppressHydrationWarning` works one level deep and is an escape hatch. https://react.dev/reference/react-dom/components/common
- Generate ids for `htmlFor` with `useId`. https://react.dev/reference/react-dom/components/common

## Events

- `onFocus`, `onBlur`, `onAbort`, `onLoad` bubble in React, not in the browser. https://react.dev/reference/react-dom/components/common
- `onInvalid` bubbles in React, unlike the built-in `invalid` event. https://react.dev/reference/react-dom/components/input
- `onDragOver` needs `e.preventDefault()` to allow a drop. https://react.dev/reference/react-dom/components/common
- `e.persist()` and `e.isPersistent()` are not used with React DOM. `e.nativeEvent` is the original browser event. https://react.dev/reference/react-dom/components/common
- Handlers have a `...Capture` variant that fires in the capture phase. `onMouseEnter`, `onMouseLeave`, `onPointerEnter` and `onPointerLeave` have no capture phase. https://react.dev/reference/react-dom/components/common
- `onSelect` also fires on empty selection and edits. https://react.dev/reference/react-dom/components/input
- Events of a portal bubble through the React tree, not the DOM tree. https://react.dev/reference/react-dom/createPortal

## Custom elements

- Props become attributes, non-strings serialized (`[1,2,3]` becomes `"1,2,3"`), unless the element class defines the property at construction, then React sets the property. https://react.dev/reference/react-dom/components
- Custom element events keep case and dashes after `on`: `onsay-hi`, `onsayHi`. https://react.dev/reference/react-dom/components
- On the server, primitives and `true` render as attributes. Objects, symbols, functions and `false` are omitted. https://react.dev/blog/2024/12/05/react-19
