# useRef() notes

## Understanding useRef

- `useRef` values are mutable containers that persist across renders but do not trigger re-renders when changed.

- `ref.current` is like editing a normal JS object property that React does not track for re-render triggers.

- Refs are intentionally designed as “escape hatches” — minimum overhead, no reconciliation, no scheduling, no renders.

- Perfect for:
    - Storing values between renders
    - Tracking state that doesn't belong in UI
    - Storing DOM nodes
    - Flags (like skipping the first effect)

- Quick rules-of-thumb
    - Local variables (declared inside component) are reinitialized on every render — they don’t persist.
    - `useRef` persists across renders and mutations to `.current` do not trigger re-renders.
    - `useState` persists and does trigger re-renders when updated — use this when UI must reflect the change.

```javascript
const App = () => {
	const isFirstRun = useRef(true);

	useEffect(() => {
		if (isFirstRun.current) {
			isFirstRun.current = false; // This will NOT re-render
			return; // skip first effect
		}

		// ...

		return () => {
			// ...
		};
	});

	return /*...*/;
};
```

---

## Why a ref does not recreate a new variable on every render?

Because React does something special with hooks:

- Every render creates a new lexical environment.
- BUT `useRef()` returns the same object instance across renders.

That "sameness" is the entire purpose of refs.

> React manages refs outside the component's JS scope.

```markdown
Render 1 → refVariable = { current: 0 }
Render 2 → refVariable = SAME OBJECT
Render 3 → refVariable = SAME OBJECT
...
Render N → refVariable = SAME OBJECT
```

_The object instance does not change. Only `.current` changes._

```javascript
const SomeComponent = ({ num }) => {
	const latestNum = useRef(num);
	latestNum.current = num; // keep ref up-to-date on every render

	useEffect(() => {
		console.log("EFFECT mounted", latestNum.current);

		return () => {
			// read the latest value at unmount time
			console.log("EFFECT cleanup (read from ref)", latestNum.current);
		};
	}, []); // runs only once
};
```
