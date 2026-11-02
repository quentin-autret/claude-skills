# ViewTransition, addTransitionType and Activity

## Contents

- ViewTransition activation
- Triggers and props
- Styling and events
- Suspense and images
- addTransitionType
- Activity

## ViewTransition activation

- `import { ViewTransition } from 'react'`. DOM only, not React Native. https://react.dev/reference/react/ViewTransition
- It animates only updates inside a Transition, a `<Suspense>` reveal or `useDeferredValue`. A plain `setState` does not animate. https://react.dev/reference/react/ViewTransition
- `<ViewTransition>` must come before any DOM node: a `<div>` above it disables enter and exit. https://react.dev/reference/react/ViewTransition
- Never call `document.startViewTransition` yourself. React calls it and interrupts other running view transitions. https://react.dev/reference/react/ViewTransition
- `flushSync` mid-sequence makes React skip the transition. https://react.dev/reference/react/ViewTransition
- `useEffect` runs after the animation ends. Layout Effects and refs run inside the update callback. https://react.dev/reference/react/ViewTransition
- Updates arriving mid-animation batch: A to B, then C and D, ends as B to D. https://react.dev/reference/react/ViewTransition

## Triggers and props

- `enter` (first inserted), `exit` (first deleted), `update` (DOM mutation inside, size or position change, nested boundaries take the mutation), `share` (same-named boundaries in a deleted and an inserted subtree). The default is the browser cross-fade. https://react.dev/reference/react/ViewTransition
- `name` is only for shared element transitions. Two mounted boundaries with the same `name` are an error: use `` `item-${id}` ``. https://react.dev/reference/react/ViewTransition
- `default`, `enter`, `exit`, `update`, `share` accept `"auto"`, `"none"`, a class name, or `{ [type]: value, default: value }`. `default="none"` disables every trigger not listed. https://react.dev/reference/react/ViewTransition
- React sets `view-transition-name` inline on the nearest DOM node only while animating. https://react.dev/reference/react/ViewTransition
- It animates a snapshot, not each child. Add nested boundaries where continuity is lost. https://react.dev/reference/react/ViewTransition

## Styling and events

- Style through the class: `::view-transition-group(.cls)`, `::view-transition-old(.cls)`, `::view-transition-new(.cls)`. Do not style by `view-transition-name`. https://react.dev/reference/react/ViewTransition
- React does not honor `prefers-reduced-motion`. Disable or soften animations in `@media (prefers-reduced-motion)`. https://react.dev/reference/react/ViewTransition
- `onEnter`, `onExit`, `onUpdate`, `onShare` receive `(instance, types)`. One event per boundary per Transition, `onShare` wins over `onEnter` and `onExit`. Always return a cleanup, for example `() => anim.cancel()`. https://react.dev/reference/react/ViewTransition
- Routers unblock a pending navigation in `useLayoutEffect`, since `useEffect` deadlocks. Back-button animations from the legacy `popstate` event are skipped: use the Navigation API. https://react.dev/reference/react/ViewTransition

## Suspense and images

- Fallbacks appear without animation, the fallback-to-content swap animates, children that do not suspend appear without animation. https://react.dev/blog/2026/09/09/react-19-3

```jsx
<ViewTransition update="auto" default="none">
  <Suspense fallback={...}>...</Suspense>
</ViewTransition>
```

https://react.dev/blog/2026/09/09/react-19-3

- Use it sparingly, and not for cached UI. https://react.dev/blog/2026/09/09/react-19-3
- During a `<ViewTransition>` update React waits up to 500 ms for visible images. Opt out with `loading="lazy"` or an `onLoad` handler. `fetchPriority="low"` does not opt out. https://react.dev/reference/react-dom/components/img

## addTransitionType

- `addTransitionType(type)` inside `startTransition` labels the cause. https://react.dev/reference/react/addTransitionType

```js
startTransition(() => {
  addTransitionType('submit-click');
  action();
});
```

https://react.dev/reference/react/addTransitionType

- Types reset after each commit: a fallback gets the types, the later reveal does not. https://react.dev/reference/react/addTransitionType
- Consume them in CSS (`:root:active-view-transition-type(my-type)`), in the object form `enter={{ next: 'from-right', previous: 'from-left' }}`, or in the `types` argument of events. https://react.dev/reference/react/addTransitionType and https://react.dev/blog/2026/09/09/react-19-3
- Several matching types are joined, no match uses `default`, a matching `"none"` disables the ViewTransition. https://react.dev/reference/react/addTransitionType

## Activity

- `<Activity mode={isVisible ? 'visible' : 'hidden'}>`. Hidden children get `display: none`, their Effects are cleaned up, their state and DOM are kept. https://react.dev/reference/react/Activity
- Hidden children still re-render on new props at low priority. Mounted hidden, they pre-render at low priority without Effects, so code and data load ahead. https://react.dev/reference/react/Activity
- DOM side effects persist while hidden: a playing `<video>` keeps playing. Pause it in the Effect cleanup. https://react.dev/reference/react/Activity
- A hidden child that returns only text renders nothing: there is no element to hide. https://react.dev/reference/react/Activity
- Keep `<StrictMode>` to expose Effects that break on hide and show. https://react.dev/reference/react/Activity
- Inside a `<ViewTransition>`, a visibility switch in a Transition triggers `enter` and `exit`. Activity boundaries also split selective hydration. https://react.dev/reference/react/Activity
