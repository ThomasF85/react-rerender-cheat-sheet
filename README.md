# React rerender cheat sheet

- [Default behavior (ordinary state and props)](#default-behavior-ordinary-state-and-props)
- [Using React.memo](#using-reactmemo)
- [Zustand and Redux](#zustand-and-redux)
- [Context API](#context)
- [Jotai](#jotai)

## Default behavior (ordinary state and props)

A component rerenders when either of the following happens:

- a state changes (referential comparison)
- a prop changes (referential comparison)
- the parent component rerenders

### example 1

Ask yourself if the component will rerender on button click (and why)

```js
export default function Game() {
  const [player, setPlayer] = useState({
    name: "Carla",
    colors: ["red", "green"],
  });

  function updatePlayer() {
    setPlayer((prev) => ({ ...prev }));
  }

  return (
    <>
      <h1>{player.name}</h1>
      <button onClick={updatePlayer}>update player</button>
    </>
  );
}
```

Answer: Component will rerender on button click, since we update the state to a new object that is not reference equal to the old state.

### example 2

Ask yourself if the component will rerender on button click (and why)

```js
export default function Game() {
  const [player, setPlayer] = useState("Carla");

  function updatePlayer() {
    setPlayer("Carla");
  }

  return (
    <>
      <h1>{player}</h1>
      <button onClick={updatePlayer}>update player</button>
    </>
  );
}
```

Answer: Component will not rerender on button click, since "Carla" is a primitive value and the value does not change.

## Using React.memo

If you use React.memo your components will not rerender anymore when the parent component rerenders. They will still rerender on props or state changes.

```js
const MemoizedComponent = React.memo(MyComponent);
```

You can pass a custom equality function to React.memo to change the default behavior of comparing props by reference equality:

```js
import deepEqual from "fast-deep-equal";

const MemoizedComponent = React.memo(MyComponent, deepEqual);
```

### example 1

Which components rerender on button click?

```js
export default function Game() {
  const [player, setPlayer] = useState({
    name: "Carla",
    colors: ["red", "green"],
  });

  function updatePlayer() {
    setPlayer((prev) => ({ ...prev }));
  }

  return (
    <>
      <MemoizedPlayername name={player.name} />
      <button onClick={updatePlayer}>update player</button>
    </>
  );
}

function PlayerName({ name }) {
  return <p>{name}</p>;
}

const MemoizedPlayername = React.memo(PlayerName);
```

Answer: The Game component rerenders. The PlayerName component does not rerender, since the prop name is a primitive value that does not change.

### example 2

Which component rerenders on button click?

```js
export default function Game() {
  const [player, setPlayer] = useState({
    name: "Carla",
    colors: ["red", "green"],
  });

  function updatePlayer() {
    setPlayer((prev) => ({ ...prev }));
  }

  return (
    <>
      <MemoizedColors colors={player.colors} />
      <button onClick={updatePlayer}>update player</button>
    </>
  );
}

function Colors({ colors }) {
  return (
    <ul>
      {colors.map((color) => (
        <li key={color}>{color}</li>
      ))}
    </ul>
  );
}

const MemoizedColors = React.memo(Colors);
```

Answer: The Game component rerenders. The Colors component does not rerender, since the array behind the prop colors remains the same object (reference equal) after button clicks.

### example 3

Which component rerenders on button click?

```js
export default function Game() {
  const [player, setPlayer] = useState({
    name: "Carla",
    colors: ["red", "green"],
  });

  function updatePlayer() {
    setPlayer({
      name: "Carla",
      colors: ["red", "green"],
    });
  }

  return (
    <>
      <MemoizedColors colors={player.colors} />
      <button onClick={updatePlayer}>update player</button>
    </>
  );
}

function Colors({ colors }) {
  return (
    <ul>
      {colors.map((color) => (
        <li key={color}>{color}</li>
      ))}
    </ul>
  );
}

const MemoizedColors = React.memo(Colors);
```

Answer: The Game component rerenders. The Colors component also rerenders, since the array behind the prop colors is a brand new object after button click and not reference equal to the previous array.

### example 3b

Which component rerenders on button click if we make the following small change to example 3?:

We hand over a custom equality function to React.memo like so (everything else stays the same as in example 3):

```js
import deepEqual from "fast-deep-equal";

const MemoizedColors = React.memo(Colors, deepEqual);
```

Answer: Now the Colors component does not rerender, since the array behind the prop colors is "deep equal" before and after button clicks.

## Zustand and Redux

With Zustand and Redux you usually manage global state in a single store. Components make use of that state through `useStore(...)` in Zustand and `useSelector(...)` in Redux.
Every time your global state is updated a component rerender might be triggered. Your components will rerender when the state they get through `useStore` or `useSelector` changes, based on reference equality (or based on a custom equality function).

### example

We have a global state that looks like this:

```js
const stateV1 = {
  name: "Carla",
  colors: ["green", "blue"],
  // when using Zustand you might have two more properties like 'setName' and 'setColors' or similar
};
```

Some part of our application changes the global state, so that it looks like the following afterwards:

```js
const stateV2 = { ...stateV1, name: "Mario" };
```

We have a component _MyComponent_. The parent component of _MyComponent_ is not effected by the state update and does not rerender.
Question: Which of the following states (colors1 till colors4) would lead to a rerender of _MyComponent_?:

```js
import deepEqual from "fast-deep-equal";

// Zustand version:
function MyComponent() {
  const colors1 = useStore((state) => state.colors);
  const colors2 = useStore((state) =>
    state.colors.filter((color) => color !== "yellow")
  );
  const colors3 = useStore(
    (state) => state.colors.filter((color) => color !== "yellow").length
  );
  const colors4 = useStore(
    (state) => state.colors.filter((color) => color !== "yellow"),
    deepEqual
  );

  // return ...
}

// Redux version:
function MyComponent() {
  const colors1 = useSelector((state) => state.colors);
  const colors2 = useSelector((state) =>
    state.colors.filter((color) => color !== "yellow")
  );
  const colors3 = useSelector(
    (state) => state.colors.filter((color) => color !== "yellow").length
  );
  const colors4 = useSelector(
    (state) => state.colors.filter((color) => color !== "yellow"),
    deepEqual
  );

  // return ...
}
```

Answers:

- colors1: Does not cause a rerender, since state.colors is reference equal in stateV1 and stateV2.
- colors2: Does cause a rerender, since the selector function `(state) => state.colors.filter((color) => color !== "yellow")` returns a new array every time it is being called. The array returned after the state update is not reference equal to the array returned before the state update.
- colors3: Does not cause a rerender, since the selector function `(state) => state.colors.filter((color) => color !== "yellow").length` returns a primitive value that does not change through the state update.
- colors4: Does not cause a rerender, since the array retrieved through `useStore` or `useSelector` after the state update is deep equal to the array retrieved before the state update.

## Context

You can use context to manage states similar to global state management libraries. Contexts are scoped, so that only descendants of a context provider can access that particular context.

It is important to understand that whenever the value of a context changes (reference change), all components using that context through `useContext` rerender. This has one immediate implication:

- If the value of your context is an object with multiple properties and a component only uses one property of that object through `useContext`, it will still rerender on every update of that context - even if the used property did not change.

# example 1 Part A

We create a context that exposes an object with two properties: `count` and `incrementCount`.
We also have a component that uses only the incrementCount function:

```js
function Increment() {
  const { incrementCount } = useContext(CountContext);

  return <button onClick={incrementCount}>increment</button>;
}
```

This component actually never needs to rerender. Is still does for some reason.

Question: Why does this component rerender every time the button is clicked? The context provider looks like this:

```js
export const CountContext = createContext();

const CountProvider = ({ children }) => {
  const [count, setCount] = useState(0);

  function incrementCount() {
    setCount((prev) => prev + 1);
  }

  return (
    <CountContext.Provider value={{ count, incrementCount }}>
      {children}
    </CountContext.Provider>
  );
};

export default function App({ Component, pageProps }) {
  return (
    <CountProvider>
      <Component {...pageProps} />
    </CountProvider>
  );
}
```

Answer: Every time the state count changes, the context provider produces a new object `{ count, incrementCount }` with a new reference. Hence all components accessing this object through `useContext` rerender.

# example 1 Part B

Question: will the following change fix the Increment component's rerendering problem and why / why not?

We are splitting the context, so that the count (which changes) is separated from our function that updates the state and never needs to change:

```js
export const CountContext = createContext();
export const CountApiContext = createContext();

const CountProvider = ({ children }) => {
  const [count, setCount] = useState(0);

  function incrementCount() {
    setCount((prev) => prev + 1);
  }

  return (
    <CountContext.Provider value={count}>
      <CountApiContext.Provider value={incrementCount}>
        {children}
      </CountApiContext.Provider>
    </CountContext.Provider>
  );
};
```

Our Increment component slightly changes:

```js
function Increment() {
  const incrementCount = useContext(CountApiContext);

  return <button onClick={incrementCount}>increment</button>;
}
```

Answer: Our rerendering problem is not solved and the Increment component will still rerender after every button click. This is because every time our count state changes in the CountProvider component, we create a brand new incrementCount function with a new reference. Hence the value of our countApiContext has a new reference and components using that context will rerender.

# example 1 Part C

Question: will the following change fix the Increment component's rerendering problem and why / why not?

We wrap our incrementCount function in a useCallback, so that the function only gets recreated when the setCount function changes (which it never does, because it has a stable reference thanks to the awesomeness of useState).

```js
export const CountContext = createContext();
export const CountApiContext = createContext();

const CountProvider = ({ children }) => {
  const [count, setCount] = useState(0);

  const incrementCount = useCallback(
    function () {
      setCount((prev) => prev + 1);
    },
    [setCount]
  );

  return (
    <CountContext.Provider value={count}>
      <CountApiContext.Provider value={incrementCount}>
        {children}
      </CountApiContext.Provider>
    </CountContext.Provider>
  );
};
```

Answer: This fixed the rerendering problem of the Increment component. Note:

- If you create functions in your provider you should use `useCallback`
- If you create objects in your provider you should use `useMemo`, like so:

```js
export const CountContext = createContext();
export const CountApiContext = createContext();

const CountProvider = ({ children }) => {
  const [count, setCount] = useState(0);

  const api = useMemo(
    () => ({
      incrementCount: () => setCount((prev) => prev + 1),
      decrementCount: () => setCount((prev) => prev - 1),
    }),
    [setCount]
  );

  return (
    <CountContext.Provider value={count}>
      <CountApiContext.Provider value={api}>
        {children}
      </CountApiContext.Provider>
    </CountContext.Provider>
  );
};
```

## Jotai

This is a recipe for a way to avoid unnecessary rerenders and to have a somewhat nice API on the side of the components which use the global state provided by jotai.
This recipe gives you a syntax similar to the dispatch / action syntax seen when using reducers.

**Setup**:

- define your atom
- define functions that return a function which receives the previous value and returns the updated value

```js
import { atom } from "jotai";

export const increment =
  (delta = 1) =>
  (prev) =>
    prev + delta;

export const decrement =
  (delta = 1) =>
  (prev) =>
    prev - delta < 0 ? 0 : prev - delta;

export const countAtom = atom(0);
```

**Usage**:

Whenever you have a component that only needs to read the state, but never updates the state, you can use the `useAtomValue` hook:

```js
import { countAtom } from "@/lib/atoms/count";
import { useAtomValue } from "jotai";

// The useAtomValue hook will trigger a rerender whenever the value (or the reference for objects) of your atom changes
function CounterValue() {
  const count = useAtomValue(countAtom);

  return <p>{count}</p>;
}
```

Whenever you have a component that only needs to update the state, but never needs to read the state, you can use the `useSetAtom` hook:

```js
import { countAtom, increment, decrement } from "@/lib/atoms/count";
import { useSetAtom } from "jotai";

// The useSetAtom hook will never trigger a rerender, since the returned function has a stable reference across renders
function CounterButtons() {
  const dispatch = useSetAtom(countAtom);

  return (
    <>
      <button onClick={() => dispatch(increment(5))}>+ 5</button>
      <button onClick={() => dispatch(decrement(5))}>- 5</button>
    </>
  );
}
```

If you have a component that needs to read and update the state, you can use the `useAtom` hook:

```js
function Counter() {
  // The useAtom hook will trigger a rerender whenever the value (or the reference for objects) of your atom changes
  const [count, dispatch] = useAtom(countAtom);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => dispatch(increment(5))}>+ 5</button>
      <button onClick={() => dispatch(decrement(5))}>- 5</button>
    </>
  );
}
```
