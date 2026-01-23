# memo() notes

Pure Component always renders the same output given the same inputs (props, state, and context) and has no side effects during rendering.

The primary purpose of pure components is performance optimization, as React can safely skip unnecessary re-renders when the inputs have not changed.

`React.memo` is a higher order component.

If your component renders the same result given the same props, you can wrap it in `React.memo` for a performance boost in some cases by memoizing the result. This means that React will skip rendering the component, and reuse the last rendered result.

`React.memo` only checks for prop changes. If your function component wrapped in `React.memo` has a `useState`, `useReducer` or `useContext` hook in its implementation, it will still re-render when state or context change.

By default it will only shallowly compare complex objects in the props object. If you want control over the comparison, you can also provide a custom comparison function as the second argument.
