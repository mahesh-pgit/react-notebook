# useEffect() deeeeep-notes

## Understanding `useEffect`

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

## Understanding cleanup and unmounting

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

During render `num=5`, cleanup 4 is executed, till here everything looks predictable, but why the hell is `EFFECT mounted 5` logged even it got unmounted from the DOM during render 5?

This is the key misunderstanding:

> Returning `null`/`false` from a component does not remove the component instance — it just renders nothing. It is not the same as unmounting the component instance.

_You will see `EFFECT mounted 5` because the component is still mounted and the effect runs for that render, even the `button` disappears. There is no final cleanup for `num=5` unless the component instance is unmounted by its parent. So the effect lifecycle for that render still runs._

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

> The cleanup holds the value that existed when the effect executed.

_React’s render creates a new lexical scope on every render (i.e. fresh `num` variable) , and the effect and its cleanup closes over the variable binding (`num`) that existed in the render when the effect executed. Later renders create new `num` bindings — but those are different variables (different lexical environments), not mutations of the original binding, so the cleanup sees the old value._

> New render → new function call → new lexical scope → new variable binding.

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
