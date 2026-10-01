# React Code Review

## Purpose

Catch the React design and performance problems that are cheap to fix in review but expensive to fix later: wrong keys, render-driven re-render storms, context over-subscription, and reaching for `memo`/`useMemo` before fixing the actual structure. The goal is to identify these issues *way earlier* -- at write/review time -- instead of after they ship as jank.

## When to Use

- Reviewing a React PR or diff
- Writing new components or hooks
- "Why is this re-rendering so much?"
- Before reaching for `React.memo`, `useMemo`, or `useCallback`
- Refactoring a large component or a shared context
- Before merging UI work that touches hot paths (typing, lists, open/mount flows)

---

## Profile critical paths before you ship

For changes that touch hot UI (suggestion list, typing, streaming, large lists), **profile the user path you changed** before calling it done:

- [ ] Record the flow in Chrome **Performance** — flag **long tasks** (≥50ms) on the main thread during open, type, scroll, and tab/mode switch.
- [ ] Fix what the trace shows: structural splits (§2–3) before `memo`; defer heavy mount (`startTransition`, `React.lazy`); remove duplicate subscriptions. Re-profile until long tasks and surprise re-renders along that path are gone or justified.

---

## Core Principle: Structure Before Optimization

> Before you apply optimizations like memo or useMemo, it might make sense to look if you can split the parts that change from the parts that don't change.

> The interesting part about these approaches is that they don't really have anything to do with performance, per se. Using the children prop to split up components usually makes the data flow of your application easier to follow and reduces the number of props plumbed down through the tree. Improved performance in cases like this is a cherry on top, not the end goal.

When you see a performance problem, the first question is **not** "where do I add `memo`?" It is **"can I split the parts that change from the parts that don't?"** Memoization is a patch over a structure problem. Fix the structure and the memo is often unnecessary.

---

## Review Checklist

Run these in order. The first three (keys, composition, context) catch the majority of real bugs.

### 1. Keys must be appropriate

- [ ] **No array index as key** when the list can reorder, insert, or delete. Index keys make React reuse the wrong DOM/state, causing stale inputs, wrong animations, and ghost values.
- [ ] Keys are **stable and unique** across renders -- derived from data identity (`item.id`), not from render-time values.
- [ ] **One key per element**, on the outermost element returned by `.map()`. Watch for double `key` (key on a wrapper *and* a child) -- usually a copy-paste smell.
- [ ] A **deliberate** `key` change to force a remount (e.g. `key={tabId}` to reset state on tab switch) is good -- but verify it's intentional and commented, not accidental.

```tsx
// BAD - index key, breaks on reorder/delete, carries stale state
{items.map((item, i) => <Row key={i} item={item} />)}

// GOOD - stable identity
{items.map((item) => <Row key={item.id} item={item} />)}

// GOOD - intentional remount to reset internal state per tab
<StateSyncWrapper key={tabId} tabId={tabId} />
```

### 2. Split what changes from what doesn't (composition)

Before memoizing, look for a structural split.

- [ ] A frequently re-rendering component (typing, streaming, animation, timers) does **not** wrap large static subtrees. Pull the static part out.
- [ ] **High-frequency / reactive state** (typing, scroll position, streaming tokens, animation frames) lives in a **sibling** or narrow **`children` wrapper** — not in the same component that renders an expensive subtree. **Do not pass that state as a prop through the expensive tree** unless the leaf genuinely needs it; `memo` on descendants will not help if a parent above them re-renders every keystroke.
- [ ] Prefer a **sibling** when the reactive UI does not need to wrap the expensive tree. Prefer **`children`** when the wrapper must own the DOM shell (layout, click target, animation container).
- [ ] Use the **`children` prop** to keep expensive/static subtrees out of a re-rendering parent. Children passed in are created by the *parent's* parent, so they keep the same reference and don't re-render when the stateful wrapper updates.
- [ ] Props aren't being **plumbed down many levels** ("prop drilling") when composition would let the consumer render the child directly.

```tsx
// BAD - <ExpensiveTree> re-renders every time `value` changes
function App() {
  const [value, setValue] = useState('');
  return (
    <div onMouseMove={(e) => setValue(e.clientX)}>
      <ExpensiveTree />
    </div>
  );
}

// GOOD (children) - isolate the changing state; ExpensiveTree is passed as children
// and keeps its reference, so it doesn't re-render on mouse move
function App() {
  return (
    <MouseMoveTracker>
      <ExpensiveTree />
    </MouseMoveTracker>
  );
}

function MouseMoveTracker({ children }) {
  const [value, setValue] = useState('');
  return <div onMouseMove={(e) => setValue(e.clientX)}>{children}</div>;
}

```

The payoff isn't only performance -- the data flow gets easier to follow and fewer props get threaded through the tree. Performance is the cherry on top.

### 3. Context subscribes to ALL values

`useContext` (and any context consumer) re-renders **whenever any field of the context value changes**, regardless of which field the component actually reads. A single context holding for eg. both `inputText` (changes on every keystroke) and `isLoggedIn` (changes rarely) will re-render every consumer of `isLoggedIn` on every keystroke.

- [ ] **Don't put fast-changing and slow-changing state in the same context value.** Split into multiple contexts (e.g. a stable "actions/dispatch" context vs. a volatile "state" context) so consumers only subscribe to what they need.
- [ ] The context **value object is memoized** (`useMemo`) or stable. A new object literal each render (`value={{ a, b }}`) re-renders every consumer every time the provider renders.
- [ ] For large/hot stores, prefer a **selector pattern** (`use-context-selector`, a store, or `useSyncExternalStore`) so a consumer re-renders only when its selected slice changes.
- [ ] **Move state down**: if only a small subtree uses a piece of state, push the state into that subtree instead of holding it in a high context/provider that re-renders everything.
- [ ] **Lift state up** only as far as the nearest common ancestor that needs it -- no higher.

```tsx
// BAD - one context, every keystroke re-renders every consumer
const AppContext = createContext();
<AppContext.Provider value={{ inputText, setInputText, user, theme }}>

// GOOD - split by change frequency; consumers subscribe narrowly
<UserContext.Provider value={userValue}>
  <InputContext.Provider value={inputValue}>

// GOOD - selector: re-renders only when `currentView` slice changes
const currentView = useAIChatSelector(selectors.currentView);
```

### 4. Stable references (only after structure is right)

- [ ] No **new object / array / function literals** passed as props to memoized children or used in dependency arrays -- they break `React.memo` and re-fire effects every render.
- [ ] Inline handlers in `.map()` rows are fine for small lists, but for hot/large lists hoist the handler or pass a stable callback + the item id.
- [ ] Constant JSX/values that never change are **hoisted out of the component** (module scope), not recreated each render.

```tsx
// GOOD - hoisted constant element, created once
const ARROW_ICON = <ArrowRight size="medium" />;
```

### 5. memo / useMemo / useCallback -- last resort, used correctly

Only reach for these **after** steps 1-4. When you do:

- [ ] `useMemo`/`useCallback` have **correct, exhaustive dependency arrays** (trust the eslint rule; don't silence it).
- [ ] The memoized value is actually expensive or actually needs reference stability -- not memoizing a primitive or a cheap computation (the memo bookkeeping can cost more than the work).
- [ ] `React.memo` is paired with **stable props** -- otherwise it never bails out and just adds overhead.
- [ ] A `ref` is used (e.g. `latestCallbackRef`) when an effect needs the *latest* value of a frequently-changing callback **without** re-subscribing -- instead of putting that callback in the dependency array.

```tsx
// GOOD - effect subscribes once; ref always points at the latest callback
const resetRef = useRef(reset);
useEffect(() => { resetRef.current = reset; }, [reset]);
useEffect(() => {
  const handler = () => resetRef.current();
  store.addEventListener('change', handler);
  return () => store.removeEventListener('change', handler);
}, []); // subscribe once, no churn
```

### 6. Hooks and effect hygiene

- [ ] Effects have **cleanup** for every subscription/listener/timer/observer they create.
- [ ] **No state updates after unmount** -- guard async effects with a `didUnmount` / `isMounted` flag or `AbortController`.
- [ ] Effects don't run more than necessary -- check the dependency array isn't over-broad (re-subscribing storms) or under-broad (stale closures).
- [ ] Business logic lives in **custom hooks or modules**, not inline in the component body -- components stay presentational.
- [ ] Hooks are called unconditionally at the top level (no hooks inside conditions/loops/early returns).
- [ ] No deriving state into `useState` + `useEffect` when it can just be **computed during render** (derived state is a common re-render and bug source).

---

## Reviewer Output Format

Group findings by severity. Reference the checklist item.

```
## React Review

### CRITICAL (will cause bugs)
1. [file:line] Index used as key in a reorderable list -> stale row state. (Checklist 1)
   Fix: key={item.id}

### CRITICAL (re-render / structure)
1. [file:line] Fast-changing inputText shares a context with rarely-changing user. (Checklist 3)
   Fix: split into two contexts or use a selector.

### CRITICAL (memo doesnt bail out)
1. [file:line] memo added to <Foo> but it receives a new inline object each render. (Checklist 4/5)
   Fix: stabilize the prop, or split the changing part out instead of memoizing.
```

---

## Quick Heuristics

- Re-render problem? **Split before you memo.**
- Fast-changing state (typing, scroll, streaming) above an expensive list? **Sibling or `children` split — don't prop-drill it through the tree.**
- Reaching for `useContext`? **Ask what else is in that context value** -- you subscribe to all of it.
- Same storage hook in every list row? **Subscribe once at the list parent, context down.**
- Threading a prop 3+ levels? **Use `children` / composition.**
- Writing `key={index}`? **Stop** unless the list is static and append-only.
- Adding `useMemo`/`useCallback`? **Confirm the dependency array is honest** and the cost is real.
- Touching a hot UI path? **Profile it** — Performance for long tasks, React Profiler for surprise re-renders.
