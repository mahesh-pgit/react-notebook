# useEffect() deeeeep-notes

## Understanding useEffect

```javascript
// Debouncing in React
const SearchBox = () => {
	const [searchTerm, setSearchTerm] = useState("");

	useEffect(() => {
		const timer = setTimeout(() => fetchData(serachTerm), 300);

		return () => clearTimeout(timer);
	}, [searchTerm]);
};
```

- The `useEffect` callback function will be executed on mount (first time by default), and everytime when the `serachTerm` changes (dependency change) or when there is no dependency array.

- The callback function returns either `undefined` or a cleanup function.

- The cleanup function gets executed BEFORE React runs the next effect (due to dependency changes or if no dependency array) and also when the `SearchBox` component unmounts.

---

If you `return;` early (no function), React registers no cleanup for that run. No cleanup → nothing to call later.

```javascript
useEffect(() => {
	// ...

	if (condition) return; // no cleanup for this run

	return () => {
		// ...
	};
});
```

---

## Examples of side effects

- Making HTTP (API/`fetch`) requests.

- Printing to the console (`console.log`).

- Mutating input data or external variables.

- DOM manipulation or querying.

- Using `Math.random()` or getting the current time (`new Date()`), as they produce inconsistent results.

---

## Can a componet have multiple useEffects?

YES. A component can totally have as many `useEffect` hooks as you want.

Instead of shoving everything into one monster `useEffect`, you can split them up for clarity and control.

> [!NOTE]
> React runs all `useEffect`s in order (top → bottom).

```javascript
const MultiEffectComponent = ({ userId }) => {
	// Effect 1: Fetch user data when userId changes
	useEffect(() => {
		console.log("Fetching data for user");
		// Simulate fetch
	}, [userId]);

	// Effect 2: Log every render
	useEffect(() => {
		console.log("Component rendered!");
	});

	// Effect 3: Setup event listener
	useEffect(() => {
		const disableImageInteractions = (e) => {
			if (e.target.tagName === "IMG") {
				e.preventDefault();
			}
		};

		document.addEventListener("contextmenu", disableImageInteractions);
		document.addEventListener("dragstart", disableImageInteractions);

		return () => {
			document.removeEventListener("contextmenu", disableImageInteractions);
			document.removeEventListener("dragstart", disableImageInteractions);
		};
	}, []);

	return <div>This FC has multiple useEffects</div>;
};
```

---

## Understanding cleanup and unmounting with single component

> [!NOTE]
> React guarantees that every effect cleans up its previous side effects before creating new ones.

```javascript
// index.js
const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

```javascript
// App.js
const App = () => {
	const [num, setNum] = useState(0);

	useEffect(() => {
		console.log("EFFECT mounted", num);

		return () => console.log("EFFECT cleanup", num);
	});

	console.log("RENDER run for num =", num);

	return num < 5 && <button onClick={() => setNum((prev) => prev + 1)}>App Component</button>;
};
```

```markdown
<!-- console -->

App.js:12 RENDER run for num = 0
App.js:7 EFFECT mounted 0
App.js:12 RENDER run for num = 1
App.js:9 EFFECT cleanup 0
App.js:7 EFFECT mounted 1
App.js:12 RENDER run for num = 2
App.js:9 EFFECT cleanup 1
App.js:7 EFFECT mounted 2
App.js:12 RENDER run for num = 3
App.js:9 EFFECT cleanup 2
App.js:7 EFFECT mounted 3
App.js:12 RENDER run for num = 4
App.js:9 EFFECT cleanup 3
App.js:7 EFFECT mounted 4
App.js:12 RENDER run for num = 5
App.js:9 EFFECT cleanup 4
App.js:7 EFFECT mounted 5
```

During render `num = 5`, cleanup 4 is executed, till here everything looks predictable, but why the hell is `EFFECT mounted 5` logged even it got unmounted from the DOM during render 5?

This is the key misunderstanding:

> [!IMPORTANT]
> Returning `null`/`false` from a component does not remove the component instance — it just renders nothing. It is not the same as unmounting the component instance.

_You will see `EFFECT mounted 5` because the component is still mounted and the effect runs for that render, even the button disappears. There is no final cleanup for `num = 5` unless the component instance is unmounted by its parent. So the effect lifecycle for that render still runs._

---

Now I actually simulated a real unmount, and the logs are as expected!

```javascript
// App.js
const App = () => {
	const [num, setNum] = useState(0);

	return num < 5 && <SomeComponent num={num} setNum={setNum} />;
};
```

```javascript
// SomeComponent.js
const SomeComponent = ({ num, setNum }) => {
	useEffect(() => {
		console.log("EFFECT mounted", num);

		return () => console.log("EFFECT cleanup", num);
	});

	console.log("RENDER run for num =", num);

	return <button onClick={() => setNum((prev) => prev + 1)}>Some Component</button>;
};
```

```markdown
<!-- console -->

SomeComponent.js:10 RENDER run for num = 0
SomeComponent.js:5 EFFECT mounted 0
SomeComponent.js:10 RENDER run for num = 1
SomeComponent.js:7 EFFECT cleanup 0
SomeComponent.js:5 EFFECT mounted 1
SomeComponent.js:10 RENDER run for num = 2
SomeComponent.js:7 EFFECT cleanup 1
SomeComponent.js:5 EFFECT mounted 2
SomeComponent.js:10 RENDER run for num = 3
SomeComponent.js:7 EFFECT cleanup 2
SomeComponent.js:5 EFFECT mounted 3
SomeComponent.js:10 RENDER run for num = 4
SomeComponent.js:7 EFFECT cleanup 3
SomeComponent.js:5 EFFECT mounted 4
SomeComponent.js:7 EFFECT cleanup 4
```

_There is no `RENDER run for num = 5` because `SomeComponent` never gets rendered for `num === 5` — the parent returns `false` so React unmounts the component instead. The cleanup you see is React unmounting the last mounted instance (which had `num = 4`)._

---

Now, if the `useEffect` has an empty dependency array (`[]`), then see what happened!

```javascript
// SomeComponent.js
const SomeComponent = ({ num, setNum }) => {
	useEffect(() => {
		console.log("EFFECT mounted", num);

		return () => console.log("EFFECT cleanup", num);
	}, []);

	console.log("RENDER run for num =", num);

	return <button onClick={() => setNum((prev) => prev + 1)}>Some Component</button>;
};
```

```markdown
<!-- console -->

SomeComponent.js:10 RENDER run for num = 0
SomeComponent.js:5 EFFECT mounted 0
SomeComponent.js:10 RENDER run for num = 1
SomeComponent.js:10 RENDER run for num = 2
SomeComponent.js:10 RENDER run for num = 3
SomeComponent.js:10 RENDER run for num = 4
SomeComponent.js:7 EFFECT cleanup 0
```

You thought the cleanup logs `EFFECT cleanup 4` during unmount, right? HAHA, GOTCHU!

> [!IMPORTANT]
> The cleanup holds the value that existed when the effect executed.

_React’s render creates a new lexical scope on every render (i.e. fresh `num` variable) , and the effect and its cleanup closes over the variable binding (`num`) that existed in the render when the effect executed. Later renders create new `num` bindings — but those are different variables (different lexical environments), not mutations of the original binding, so the cleanup sees the old value._

> **New render → new function call → new lexical scope → new variable binding.**

Visualize it like:

- Render 0 → `num_0` exists → effect closes over `num_0`
- Render 1 → `num_1` exists → different `num` variable

So the cleanup created in Render 0 still references `num_0`. It never points to `num_1`, `num_2`, etc.

That’s why your cleanup logged 0 even though `num` later became 5.

---

If you want the cleanup to read the latest `num`, use `useRef` as follows.

```javascript
// SomeComponent.js
const SomeComponent = ({ num, setNum }) => {
	const refNum = useRef(num);
	refNum.current = num; // keep ref up-to-date on every render

	useEffect(() => {
		console.log("EFFECT mounted", refNum.current);

		return () => {
			// read the latest value at unmount time
			console.log("EFFECT cleanup", refNum.current);
		};
	}, []);

	console.log("RENDER run for num =", num);

	return <button onClick={() => setNum((prev) => prev + 1)}>Some Component</button>;
};
```

```markdown
<!-- console -->

SomeComponent.js:16 RENDER run for num = 0
SomeComponent.js:8 EFFECT mounted 0
SomeComponent.js:16 RENDER run for num = 1
SomeComponent.js:16 RENDER run for num = 2
SomeComponent.js:16 RENDER run for num = 3
SomeComponent.js:16 RENDER run for num = 4
SomeComponent.js:12 EFFECT cleanup 4
```

---

## Understanding render lifecycle with component composition

Take a moment to understand the code below and observe the console logs.

```javascript
// App.js
const App = () => {
	const [num, setNum] = useState(0);

	console.log("App RENDER run for num =", num);

	return (
		<>
			<button onClick={() => setNum((prev) => prev + 1)}>Increment</button>
			{num < 3 && <GrandParent num={num} />}
		</>
	);
};

const GrandParent = ({ num }) => {
	useEffect(() => {
		console.log("GrandParent EFFECT", num);

		return () => console.log("GrandParent CLEANUP", num);
	}, [num]);

	console.log("GrandParent RENDER", num);

	return (
		<>
			<ParentOne num={num} />
			<ParentTwo num={num} />
		</>
	);
};

const ParentOne = ({ num }) => {
	useEffect(() => {
		console.log("ParentOne EFFECT", num);

		return () => console.log("ParentOne CLEANUP", num);
	}, [num]);

	console.log("ParentOne RENDER", num);

	return <Child num={num} />;
};

const Child = ({ num }) => {
	useEffect(() => {
		console.log("ParentOne's Child EFFECT", num);

		return () => console.log("ParentOne's Child CLEANUP", num);
	}, [num]);

	console.log("ParentOne's Child RENDER", num);

	return <></>;
};

const ParentTwo = ({ num }) => {
	useEffect(() => {
		console.log("ParentTwo EFFECT", num);

		return () => console.log("ParentTwo CLEANUP", num);
	}, [num]);

	console.log("ParentTwo RENDER", num);

	return <></>;
};

export default App;
```

```markdown
<!--console-->

App.js:6 App RENDER run for num = 0
App.js:23 GrandParent RENDER 0
App.js:40 ParentOne RENDER 0
App.js:52 ParentOne's Child RENDER 0
App.js:64 ParentTwo RENDER 0
App.js:47 ParentOne's Child EFFECT 0
App.js:35 ParentOne EFFECT 0
App.js:59 ParentTwo EFFECT 0
App.js:18 GrandParent EFFECT 0
App.js:6 App RENDER run for num = 1
App.js:23 GrandParent RENDER 1
App.js:40 ParentOne RENDER 1
App.js:52 ParentOne's Child RENDER 1
App.js:64 ParentTwo RENDER 1
App.js:49 ParentOne's Child CLEANUP 0
App.js:37 ParentOne CLEANUP 0
App.js:61 ParentTwo CLEANUP 0
App.js:20 GrandParent CLEANUP 0
App.js:47 ParentOne's Child EFFECT 1
App.js:35 ParentOne EFFECT 1
App.js:59 ParentTwo EFFECT 1
App.js:18 GrandParent EFFECT 1
App.js:6 App RENDER run for num = 2
App.js:23 GrandParent RENDER 2
App.js:40 ParentOne RENDER 2
App.js:52 ParentOne's Child RENDER 2
App.js:64 ParentTwo RENDER 2
App.js:49 ParentOne's Child CLEANUP 1
App.js:37 ParentOne CLEANUP 1
App.js:61 ParentTwo CLEANUP 1
App.js:20 GrandParent CLEANUP 1
App.js:47 ParentOne's Child EFFECT 2
App.js:35 ParentOne EFFECT 2
App.js:59 ParentTwo EFFECT 2
App.js:18 GrandParent EFFECT 2
App.js:6 App RENDER run for num = 3
App.js:20 GrandParent CLEANUP 2
App.js:37 ParentOne CLEANUP 2
App.js:49 ParentOne's Child CLEANUP 2
App.js:61 ParentTwo CLEANUP 2
App.js:6 App RENDER run for num = 4
App.js:6 App RENDER run for num = 5
```

```markdown
# Our Component Tree (Conceptual Representation)

App
└── GrandParent
    ├── ParentOne
    │   └── Child
    └── ParentTwo
```

> [!NOTE]
>
> - React renders the tree top → bottom in hierarchy (depth-first traversal).
> - React batches rendering, and once the render phase is complete, it runs all `useEffect`s.
> - Effects run bottom → top in hierarchy (after paint).

1️⃣ Initial mount (`num = 0`)

Render phase (top → bottom)

```markdown
<!--console-->

App RENDER run for num = 0
GrandParent RENDER 0
ParentOne RENDER 0
ParentOne's Child RENDER 0
ParentTwo RENDER 0
```

Effect phase (bottom → top)

```markdown
<!--console-->

ParentOne's Child EFFECT 0
ParentOne EFFECT 0
ParentTwo EFFECT 0
GrandParent EFFECT 0
```

We already know from the previous section that cleanup always runs before the next effect. But what we didn’t know earlier is this:

> [!NOTE]
> Cleanups also run bottom → top in the hierarchy before effects run.

Why?

_React guarantees that no parent cleanup runs while a child still depends on it._

2️⃣ State updates (`num = 1`, `num = 2`)

Render phase (top → bottom)

```markdown
<!--console-->

App RENDER run for num = 1
GrandParent RENDER 1
ParentOne RENDER 1
ParentOne's Child RENDER 1
ParentTwo RENDER 1
```

Cleanup of previous effects (bottom → top)

```markdown
<!--console-->

ParentOne's Child CLEANUP 0
ParentOne CLEANUP 0
ParentTwo CLEANUP 0
GrandParent CLEANUP 0
```

Run new effects (bottom → top)

```markdown
<!--console-->

ParentOne's Child EFFECT 1
ParentOne EFFECT 1
ParentTwo EFFECT 1
GrandParent EFFECT 1
```

Till now, everything aligns perfectly with the stated React rules.

At this point, it’s reasonable to expect that during unmount, cleanups would also run bottom → top (Child → Parent → GrandParent), just like with `num = 1` and `num = 2`.

But React changes the cleanup order during unmount (`num = 3`).

3️⃣ Unmount (`num = 3`)

Render phase (top → bottom)

(`GrandParent` subtree is removed)

```markdown
<!--console-->

App RENDER run for num = 3
```

Unmount cleanup (top → bottom)

```markdown
<!--console-->

GrandParent CLEANUP 2
ParentOne CLEANUP 2
ParentOne's Child CLEANUP 2
ParentTwo CLEANUP 2
```

_Cleanup order during dependency changes ≠ cleanup order during unmount._

React did not violate its own rules. It followed a different rule set.

> [!TIP]
> Unmount cleanup is not effect cleanup. It is component destruction.

```ini
Update   = reconciliation
Unmount  = destruction
```

#### Why dependency-change cleanup is bottom → top

When `num` changes from `0 → 1` or `1 → 2`:

- Components remain mounted
- React therefore ensures:
    - Child cleanup runs first
    - Parent cleanup runs after

This prevents parents from tearing down resources that children still rely on.

#### Why unmount cleanup is top → bottom

When `num = 3`, the `GrandParent` subtree must be removed from the tree.

- React does not render it anymore
- React marks the root of the deleted subtree (`GrandParent`)
- Traverses the subtree top → bottom
- Runs cleanups as it dismantles each fiber node (component)

Why this order?

Because at this point:

- No component in that subtree will render again
- No effect will be re-run
- No parent-child dependency needs to be preserved

So React optimizes for:

- Simpler traversal
- Faster commit
- Fewer memory checks
