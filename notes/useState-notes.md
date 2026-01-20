# useState() notes

```javascript
const App = () => {
	const [isOpen, setIsOpen] = useState(false);

	return (
		<button
			onClick={() => {
				setIsOpen(!isOpen); // won't get updated here
				console.log(isOpen); // false
				if (isOpen) /*...*/ ; // this line gets skipped iykyk
			}}>
			Click to open
		</button>
	);
};
```

Initially the `isOpen` is `false` and on clicking the button, `console.log` must be `true` cause its updating the `isOpen` in the before statement. Right? NOPE.

> `setState` in React is asynchronous.

`console.log(isOpen)` will still log the previous value, not the updated one. Why?

#### React batches state updates for performance. So when you call `setIsOpen(!isOpen)`, the state change is scheduled, but not immediately applied in the same tick.

---

```javascript
const App = () => {
	const [obj, setObj] = useState({ num: 1 });

	useEffect(() => {
		console.log("Effect ran");
	}, [obj]);

	console.log("Render");

	return (
		<button
			onClick={() =>
				setObj((prevState) => {
					console.log("Setter function called");
					prevState.num = 2; // state mutated
					console.log("obj.num =", obj.num);
					return prevState; // same reference -> no re-render
				})
			}>
			Click Me
		</button>
	);
};
```

```
<!-- console -->
App.js:11 Render
App.js:8 Effect ran
App.js:17 Setter function called
App.js:19 obj.num = 2
```

Here, we might think that the state `obj` gets updated, so React should trigger a re-render. But here’s the thing: React does not re-render, even though the state mutates behind the scenes.

> React compares objects by reference, not by value.

#### Since we’re returning the same object from the setter function, React sees that the reference has not changed. So it assumes the state has not changed, even though it has.

The same applies to `useEffect`. Dependencies are also compared by reference, not by value.

This is why state must be treated as immutable in React. The correct way to update this state is:

```javascript
setObj((prev) => ({
	...prev,
	num: 2,
}));
```

Now React sees a new reference, triggers a re-render, and re-runs effects.
