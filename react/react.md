React is an open-source JavaScript library developed by Meta for building modern, fast, and interactive user interfaces—primarily for single-page applications (SPAs). Instead of directly manipulating the browser's Document Object Model (DOM), React lets you write declarative UI code driven by state.

---

## 1. Components: The Building Blocks

Components are self-contained, reusable modules that control a piece of the user interface. A React application is structured as a tree of components nested inside each other.

### Types of Components

- **Functional Components:** Standard JavaScript functions that return JSX. Modern React development relies almost exclusively on functional components.
- **Class Components:** Older ES6 class-based components. While still supported, functional components with Hooks have superseded them.

```jsx
// Example of a Functional Component
function UserProfile({ username }) {
  return (
    <div className="profile-card">
      <h2>User Profile</h2>
      <p>Welcome back, {username}!</p>
    </div>
  );
}
```

---

## 2. JSX (JavaScript XML)

JSX is a syntax extension for JavaScript that allows you to write HTML-like markup inside your JavaScript files. While not strictly required, it makes defining UI structure much more intuitive.

### Key Rules of JSX:

1. **Single Root Element:** JSX elements must be wrapped in a single parent container or fragment (`<> ... </>`).
2. **Embedded Expressions:** Wrap any JavaScript variable or expression in curly braces `{}` to render dynamic content.
3. **CamelCase Attributes:** HTML attributes use camelCase equivalents (e.g., `class` becomes `className`, `for` becomes `htmlFor`).

---

## 3. Props (Properties)

Props are read-only inputs passed down from a parent component to a child component. They function similarly to arguments passed into a function or attributes on an HTML tag.

### Key Characteristics:

- **Unidirectional Data Flow:** Props flow in one direction—downward from parent to child.
- **Immutability:** A child component **must never modify** its own `props`.

```jsx
// Parent Component
function App() {
  return <Greeting name="Alice" age={30} />;
}

// Child Component
function Greeting(props) {
  return (
    <h1>
      Hello {props.name}, you are {props.age} years old.
    </h1>
  );
}
```

---

## 4. State: Internal Component Memory

While `props` are read-only values supplied from the outside, **State** is internal memory managed entirely _within_ the component. When state changes, React automatically re-renders the component to reflect the update in the UI.

```jsx
import { useState } from "react";

function Counter() {
  // Declare a state variable named "count" initialized to 0
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Current Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

---

## 5. React Hooks

Introduced in React 16.8, **Hooks** are built-in functions that let functional components use state, lifecycle methods, and other React features without writing class components.

| Hook                          | Purpose                                                                                                        |
| :---------------------------- | :------------------------------------------------------------------------------------------------------------- |
| **`useState`**                | Manages local component state.                                                                                 |
| **`useEffect`**               | Handles side-effects like fetching data, subscriptions, or direct DOM manipulation.                            |
| **`useContext`**              | Consumes values from React's Context API to avoid passing props down manually ("prop drilling").               |
| **`useRef`**                  | Holds mutable references to DOM elements or values that persist across renders without triggering a re-render. |
| **`useMemo` & `useCallback`** | Performance optimization hooks used to cache computed values and callback functions.                           |

---

## 6. The Virtual DOM & Reconciliation

React achieves high performance by managing an in-memory representation of the real browser DOM, known as the **Virtual DOM**.

### How it Works:

1. **Render:** When a component's state or props change, React creates a new Virtual DOM tree representing the updated UI.
2. **Diffing:** React compares this new tree with the previous Virtual DOM tree using an efficient comparison algorithm (**Reconciliation**).
3. **Commit:** React calculates the precise differences and patches _only_ those changes into the actual browser DOM, minimizing expensive browser re-layouts.

---

## 7. Event Handling

Event handling in React is similar to standard HTML events but uses normalized cross-browser wrappers called `SyntheticEvents`.

- Event handlers are named using **camelCase** (e.g., `onClick`, `onSubmit`, `onChange`).
- Instead of passing a string, you pass an actual function reference.

```jsx
function Button() {
  const handleClick = (event) => {
    event.preventDefault();
    alert("Button was clicked!");
  };

  return <button onClick={handleClick}>Click Me</button>;
}
```

---

## 8. Lists and Keys

When rendering lists of data in React, each rendered item must be given a unique `key` prop.

```jsx
const items = ["React", "Vue", "Angular"];

function FrameworkList() {
  return (
    <ul>
      {items.map((framework, index) => (
        <li key={framework}>{framework}</li>
      ))}
    </ul>
  );
}
```

- **Why keys matter:** React uses `key` identifiers during the diffing process to recognize which items were added, removed, or re-ordered, ensuring efficient updating and maintaining local component state correctly.

---

## Summary Checklist

- **Components:** Modular UI units (Functions).
- **JSX:** XML/HTML embedded in JavaScript.
- **Props:** Input parameters passed down from parents (Immutable).
- **State:** Local component data (Mutable via setter functions).
- **Hooks:** Functions enabling state and side-effects in functional components.
- **Virtual DOM:** Fast in-memory diffing mechanism for real DOM updates.
