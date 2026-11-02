# browser, createPortal, flushSync, resource hints and head elements

## Contents

- browser
- createPortal
- flushSync
- Resource hints
- img
- link and style
- script
- meta and title

## browser

- `use(browser(reason?))`, with `use` from `react` and `browser` from `react-dom`, stops server rendering of the component and leaves the closest `<Suspense>` fallback in the HTML. In the browser it returns `undefined`. https://react.dev/reference/react-dom/browser
- It replaces `typeof window` checks, a "mounted" state set in an Effect, and framework options that disable SSR. https://react.dev/reference/react-dom/browser
- It needs a `<Suspense>` boundary during server rendering and a Client Component. `browser()` alone does nothing: pass it to `use`, never throw it. https://react.dev/reference/react-dom/browser
- `reason` is a string or a function, called on the server only, never serialized into the HTML. It becomes the `cause` of the Error passed to `onBrowserBailout`. https://react.dev/reference/react-dom/browser
- Like any `use`, it can be conditional, so a Hook opts out of SSR only when needed. https://react.dev/reference/react-dom/browser

```js
export function useTimeZone(defaultTimeZone) {
  if (defaultTimeZone !== undefined) return defaultTimeZone;
  use(browser('No default time zone was provided.'));
  return Intl.DateTimeFormat().resolvedOptions().timeZone;
}
```

https://react.dev/reference/react-dom/browser

- `abort(browser('timed out'))` leaves pending boundaries in fallback for the browser to render. https://react.dev/reference/react-dom/browser

## createPortal

- `createPortal(children, domNode, key?)` moves only the DOM placement. Context still flows and events bubble through the React tree: a click in a portal fires `onClick` on a wrapping React parent. https://react.dev/reference/react-dom/createPortal
- `domNode` must already exist. Changing it recreates the portal content. https://react.dev/reference/react-dom/createPortal
- Portals may need keyboard focus management. Follow the WAI-ARIA Modal Authoring Practices for modals. https://react.dev/reference/react-dom/createPortal
- Into a third-party widget node: store the node in state from an Effect, render the portal once it is non-null. https://react.dev/reference/react-dom/createPortal

```js
{showModal && createPortal(
  <ModalContent onClose={() => setShowModal(false)} />,
  document.body
)}
```

https://react.dev/reference/react-dom/createPortal

## flushSync

- A last resort that can hurt performance. It may show pending Suspense fallbacks, run pending Effects, and flush updates outside its callback. https://react.dev/reference/react-dom/flushSync
- Valid for browser APIs that need the DOM written synchronously, such as `beforeprint`. https://react.dev/reference/react-dom/flushSync

```js
function handleBeforePrint() {
  flushSync(() => {
    setIsPrinting(true);
  });
}
```

https://react.dev/reference/react-dom/flushSync

- Inside render or an Effect it does nothing and warns "flushSync was called from inside a lifecycle method". Move it to an event handler. https://react.dev/reference/react-dom/flushSync

## Resource hints

- `prefetchDNS`, `preconnect`, `preload`, `preloadModule`, `preinit`, `preinitModule` from `react-dom`. Frameworks often call them: check the framework first. https://react.dev/reference/react-dom
- Duplicate calls count as one. In the browser they work anywhere. On the server they count only during render or async work started by render. https://react.dev/reference/react-dom/preload
- Call one in an event handler before a navigation to start earlier. https://react.dev/reference/react-dom/preload
- Pick: download only, `preload` (or `preloadModule` for ESM). Download and run a script or apply a stylesheet now, `preinit` (or `preinitModule`). https://react.dev/reference/react-dom/preinit
- `preload` requires `as`, and `crossOrigin` when `as` is `"fetch"`. When preloading a stylesheet, also preload its fonts. https://react.dev/reference/react-dom/preload
- `preinit` of a stylesheet requires `precedence`. https://react.dev/reference/react-dom/preinit

```js
preinit("https://example.com/style.css", { as: "style", precedence: "medium" });
```

https://react.dev/reference/react-dom/preinit

- `preconnect` and `prefetchDNS` bring nothing for the page's own host. For many speculative domains, `prefetchDNS` costs less. https://react.dev/reference/react-dom/prefetchDNS

## img

- Never pass `src=""`: the browser may refetch the page. Omit the element or pass `null`. https://react.dev/reference/react-dom/components/img
- During server rendering React emits a preload hint for each `<img>` by default. Opt out with `loading="lazy"` or `fetchPriority="low"`. No hint inside `<picture>` or `<noscript>`, or for data URLs. Component libraries may add `loading="lazy"` by default. https://react.dev/reference/react-dom/components/img
- Set `width` and `height` to reserve space. https://react.dev/reference/react-dom/components/img

## link and style

- `<link rel="stylesheet" precedence>` is hoisted to `<head>`, deduplicated by `href`, and suspends the component while loading. https://react.dev/reference/react-dom/components/link
- No `precedence`, or any of `onLoad`, `onError`, `disabled`, turns this off. `itemProp` keeps the link in place. https://react.dev/reference/react-dom/components/link
- Prop changes after the first render are ignored, and the link may stay after unmount. https://react.dev/reference/react-dom/components/link
- `precedence` values are names: first discovered is lower, later is higher. Many style systems work fine with a single value. Also call `preinit` so fetching can start earlier. https://react.dev/reference/react-dom/components/link
- `<style href precedence>` is hoisted and deduplicated: make `href` unique per content, such as `"PieChart-" + JSON.stringify(colors)`. It does not suspend. https://react.dev/reference/react-dom/components/style
- `blocking="render"` makes the browser hold rendering until the stylesheet loads: Suspense gives finer control. https://react.dev/reference/react-dom/components/link

## script

- Hoisting and deduplication need `src` and `async={true}`. Inline scripts are neither moved nor deduplicated. https://react.dev/reference/react-dom/components/script
- The component may commit before the script loads: depend on it only after `onLoad`, which turns off the special handling. https://react.dev/reference/react-dom/components/script
- `defer` is not compatible with streaming server-rendered components: use `async`. https://react.dev/reference/react-dom/components/script

## meta and title

- `<meta>` always goes to `<head>`, except with `itemProp`. Give exactly one of `name`, `httpEquiv`, `charset`, `itemProp`. https://react.dev/reference/react-dom/components/meta
- Render one `<title>` at a time. Children must be one string: `<title>{`Results page ${pageNumber}`}</title>`, not `<title>Results page {pageNumber}</title>`. https://react.dev/reference/react-dom/components/title
- Libraries such as react-helmet still help for route-level overrides. https://react.dev/blog/2024/12/05/react-19
