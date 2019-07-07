---
title: "How to use react-tracked: React hooks-oriented Todo List example"
subtitle: "With immer"
date: 2019-07-08T01:00:00+09:00
tags: ["react", "context", "hooks", "global-state", "typescript"]
---

### Introduction

React hooks have changed the way to compose components.
This post will show an example which is very hooks-oriented.

We use two libraries: react-tracked and immer.
While immer makes it easy to update state in an immutable way,
react-tracked makes it easy to read state with tracking for optimization.
Please check out the repo for more details.

https://github.com/dai-shi/react-tracked

The example we show is the one from Redux: [Todo List](https://redux.js.org/basics/example)

### Folder structure

```yaml
- src/
  - index.tsx
  - state.ts
  - hooks/
    - useAddTodo.ts
    - useToggleTodo.ts
    - useVisibilityFilter.ts
    - useVisibleTodos.ts
  - components/
    - AddTodo.tsx
    - App.tsx
    - FilterLink.tsx
    - Footer.tsx
    - Todo.tsx
    - VisibleTodoList.tsx
```

We have two folders `components` and `hooks`.
Components are basically views. Hooks include logic.

### src/state.ts

In this example, we don't use reducers.
We define a state only and some types.

```typescript
import { useState } from 'react';

export type VisibilityFilterType =
  | 'SHOW_ALL'
  | 'SHOW_COMPLETED'
  | 'SHOW_ACTIVE';

export type TodoType = {
  id: number;
  text: string;
  completed: boolean;
};

export type State = {
  todos: TodoType[];
  visibilityFilter: VisibilityFilterType;
};

const initialState: State = {
  todos: [],
  visibilityFilter: 'SHOW_ALL',
};

export const useValue = () => useState(initialState);

export type SetState = ReturnType<typeof useValue>[1];
```

Notice the last line. It might be a bit tricky.
`SetState` is a type for `setState`.

### src/hooks/useAddTodo.ts

```typescript
import { useCallback } from 'react';
import { useDispatch } from 'react-tracked';
import produce from 'immer';

import { SetState } from '../state';

let nextTodoId = 0;

const useAddTodo = () => {
  const setState = useDispatch<SetState>();
  const addTodo = useCallback((text: string) => {
    setState(s => produce(s, (draft) => {
      draft.todos.push({
        id: nextTodoId++,
        text,
        completed: false,
      });
    }));
  }, [setState]);
  return addTodo;
};

export default useAddTodo;
```

This is the hook responsible for adding an item.
We use immer here, but it's not necessary.

### src/hooks/useToggleTodo.ts

```typescript
import { useCallback } from 'react';
import { useDispatch } from 'react-tracked';
import produce from 'immer';

import { SetState } from '../state';

const useToggleTodo = () => {
  const setState = useDispatch<SetState>();
  const toggleTodo = useCallback((id: number) => {
    setState(s => produce(s, (draft) => {
      const found = draft.todos.find(todo => todo.id === id);
      if (found) {
        found.completed = !found.completed;
      }
    }));
  }, [setState]);
  return toggleTodo;
};

export default useToggleTodo;
```

Same idea of this hook to toggle an item.

### src/hooks/useVisibilityFilter.ts

```typescript
import { useCallback } from 'react';
import { useTracked } from 'react-tracked';
import produce from 'immer';

import { VisibilityFilterType, State, SetState } from '../state';

const useVisibilityFilter = () => {
  const [state, setState] = useTracked<State, SetState>();
  const setVisibilityFilter = useCallback((filter: VisibilityFilterType) => {
    setState(s => produce(s, (draft) => {
      draft.visibilityFilter = filter;
    }));
  }, [setState]);
  return [state.visibilityFilter, setVisibilityFilter] as [
    VisibilityFilterType,
    typeof setVisibilityFilter,
  ];
};

export default useVisibilityFilter;
```

This hook is for both returning the current `visibilityFilter`
and a setter function. We use `useTracked` for this.
It is a combined hook to combine `useTrackedState` and `useDispatch`.

### src/hooks/useVisibleTodos.ts

```typescript
import { useTrackedState } from 'react-tracked';

import { TodoType, VisibilityFilterType, State } from '../state';

const getVisibleTodos = (todos: TodoType[], filter: VisibilityFilterType) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos;
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed);
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed);
    default:
      throw new Error(`Unknown filter: ${filter}`);
  }
};

const useVisibleTodos = () => {
  const state = useTrackedState<State>();
  return getVisibleTodos(state.todos, state.visibilityFilter);
};

export default useVisibleTodos;
```

This hook handles filtering of Todo items.

### src/components/AddTodo.tsx

```jsx
import * as React from 'react';
import { useState } from 'react';

import useAddTodo from '../hooks/useAddTodo';

const AddTodo: React.FC = () => {
  const [text, setText] = useState('');
  const addTodo = useAddTodo();
  return (
    <div>
      <form
        onSubmit={(e) => {
          e.preventDefault();
          if (!text.trim()) {
            return;
          }
          addTodo(text);
          setText('');
        }}
      >
        <input value={text} onChange={e => setText(e.target.value)} />
        <button type="submit">Add Todo</button>
      </form>
    </div>
  );
};

export default AddTodo;
```

There's nothing special to note except for `useAddTodo` being imported
from the `hooks` folder.

### src/components/Todo.tsx

```jsx
import * as React from 'react';

type Props = {
  onClick: (e: React.MouseEvent) => void;
  completed: boolean;
  text: string;
};

const Todo: React.FC<Props> = ({ onClick, completed, text }) => (
  <li
    onClick={onClick}
    role="presentation"
    style={{
      textDecoration: completed ? 'line-through' : 'none',
      cursor: 'pointer',
    }}
  >
    {text}
  </li>
);

export default Todo;
```

This is a componet with no hooks dependency.

### src/components/VisibleTodoList.tsx

```jsx
import * as React from 'react';

import useVisibleTodos from '../hooks/useVisibleTodos';
import useToggleTodo from '../hooks/useToggleTodo';
import Todo from './Todo';

const VisibleTodoList: React.FC = () => {
  const visibleTodos = useVisibleTodos();
  const toggleTodo = useToggleTodo();
  return (
    <ul>
      {visibleTodos.map(todo => (
        <Todo key={todo.id} {...todo} onClick={() => toggleTodo(todo.id)} />
      ))}
    </ul>
  );
};

export default VisibleTodoList;
```

This is different from the original example. We moved the filtering logic to hooks.

### src/components/FilterLink.tsx

```jsx
import * as React from 'react';

import useVisibilityFilter from '../hooks/useVisibilityFilter';
import { VisibilityFilterType } from '../state';

type Props = {
  filter: VisibilityFilterType;
};

const FilterLink: React.FC<Props> = ({ filter, children }) => {
  const [visibilityFilter, setVisibilityFilter] = useVisibilityFilter();
  const active = filter === visibilityFilter;
  return (
    <button
      type="button"
      onClick={() => setVisibilityFilter(filter)}
      disabled={active}
      style={{
        marginLeft: '4px',
      }}
    >
      {children}
    </button>
  );
};

export default FilterLink;
```

This uses the `useVisibilityFilter` hook.
Notice the hook returns a tuple, a value and a setter function.

### src/components/Footer.tsx

```jsx
import * as React from 'react';

import FilterLink from './FilterLink';

const Footer: React.FC = () => (
  <div>
    <span>Show: </span>
    <FilterLink filter="SHOW_ALL">All</FilterLink>
    <FilterLink filter="SHOW_ACTIVE">Active</FilterLink>
    <FilterLink filter="SHOW_COMPLETED">Completed</FilterLink>
  </div>
);

export default Footer;
```

Nothing special to note for this component.

### src/components/App.tsx

```jsx
import * as React from 'react';

import Footer from './Footer';
import AddTodo from './AddTodo';
import VisibleTodoList from './VisibleTodoList';

const App: React.FC = () => (
  <div>
    <AddTodo />
    <VisibleTodoList />
    <Footer />
  </div>
);

export default App;
```

This is the component to compose other components all together.

### src/index.tsx

Finally, we need the entry point.

```jsx
import * as React from 'react';
import { render } from 'react-dom';
import { Provider } from 'react-tracked';

import { useValue } from './state';
import App from './components/App';

const Index = () => (
  <Provider useValue={useValue}>
    <App />
  </Provider>
);

render(React.createElement(App), document.getElementById('app'));
```

Notice the `<Provider>` passes the `useValue` from state.ts.

### Online demo

<iframe src="https://codesandbox.io/embed/github/dai-shi/react-tracked/tree/master/examples/07_todolist?fontsize=14" title="react-tracked-example" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

[Source code in the repo](https://github.com/dai-shi/react-tracked/tree/master/examples/07_todolist)

### Closing notes

As I wrote this post, I noticed something.
My original motivation is to show how to use react-tracked.
However, this example is also good to show how setState and
custom hooks can separate concerns without reducers.
The other minor finding for me is that immer doesn't help much
in custom hooks in this example.

We didn't discuss much about performance optimization.
There's some room for improvement.
One of the easiest ones is to use `React.memo`.
Optimization could be a separate topic for future posts.
