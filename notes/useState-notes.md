# useState() notes

```javascript
const App = () => {
	const [isOpen, setIsOpen] = useState(false);

	return (
		<button
			onClick={() => {
				setIsOpen(!isOpen); // won't get updated here
				console.log(isOpen); //false
				if (isOpen) //...; //this line gets skipped iykyk
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
