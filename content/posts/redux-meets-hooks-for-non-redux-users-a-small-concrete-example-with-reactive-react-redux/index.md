---
title: "Redux meets hooks for non-redux users: a small concrete example with reactive-react-redux"
subtitle: "The Todo List example from Redux"
date: 2019-06-03T23:00:00+09:00
tags: ["react", "hooks", "global-state", "redux", "typescript"]
---

### Introduction

If you had already used Redux and loved it,
you might not understand why people try
using React context and hooks to replace Redux (a.k.a no Redux hype).
For those who would think Redux DevTools Extension and middleware
are nice to have, Redux and context + hooks are actually two options.
Context + hooks is just fine to share state among components,
however if apps get bigger, they are likely to require
Redux or other similar solutions; otherwise they end up having many contexts
that can't be handled very easily. (I'd admit this is hypothetical
and we would be able to find better solutions in the future.)

I've been developing a library called "reactive-react-redux"
and although it's based on Redux, it's different.

https://github.com/dai-shi/reactive-react-redux

Its API is very straightforward, and thanks to Proxy
it's optimized for performance.
With the hope that this library would pull back
people who seek Redux alternatives with context + hooks,
I created example code.
It's the famous Todo List example from Redux.

https://redux.js.org/basics/example

The rest of this post shows example code rewritten with reactive-react-redux.

### Types and reducers

The example is written in TypeScript.
If you are not familiar with TypeScript,
try ignoring `State`, `Action` and `*Type`.

The following is the type definitions for State and Action.

#### ./src/types/index.ts

```javascript
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

export type Action =
  | { type: 'ADD_TODO'; id: number; text: string }
  | { type: 'SET_VISIBILITY_FILTER'; filter: VisibilityFilterType }
  | { type: 'TOGGLE_TODO'; id: number };
```

----

The reducers are almost identical to the original example as follows.

#### ./src/reducers/index.ts

```javascript
import { combineReducers } from 'redux';

import todos from './todos';
import visibilityFilter from './visibilityFilter';

export default combineReducers({
  todos,
  visibilityFilter,
});
```

#### ./src/reducers/todos.ts

```javascript
import { TodoType, Action } from '../types';

const todos = (state: TodoType[] = [], action: Action): TodoType[] => {
  switch (action.type) {
    case 'ADD_TODO':
      return [
        ...state,
        {
          id: action.id,
          text: action.text,
          completed: false,
        },
      ];
    case 'TOGGLE_TODO':
      return state.map((todo: TodoType) => (
        todo.id === action.id ? { ...todo, completed: !todo.completed } : todo
      ));
    default:
      return state;
  }
};

export default todos;
```

#### ./src/reducers/visibilityFilter.ts

```javascript
import { Action, VisibilityFilterType } from '../types';

const visibilityFilter = (
  state: VisibilityFilterType = 'SHOW_ALL',
  action: Action,
): VisibilityFilterType => {
  switch (action.type) {
    case 'SET_VISIBILITY_FILTER':
      return action.filter;
    default:
      return state;
  }
};

export default visibilityFilter;
```

### Action creators

There could be several ways to dispatch actions.
I would create hooks for each actions.
Note that we still explore better practices.

#### ./src/actions/index.ts

```javascript
import { useCallback } from 'react';
import { useReduxDispatch } from 'reactive-react-redux';

import { Action, VisibilityFilterType } from '../types';

let nextTodoId = 0;

export const useAddTodo = () => {
  const dispatch = useReduxDispatch<Action>();
  return useCallback((text: string) => {
    dispatch({
      type: 'ADD_TODO',
      id: nextTodoId++,
      text,
    });
  }, [dispatch]);
};

export const useSetVisibilityFilter = () => {
  const dispatch = useReduxDispatch<Action>();
  return useCallback((filter: VisibilityFilterType) => {
    dispatch({
      type: 'SET_VISIBILITY_FILTER',
      filter,
    });
  }, [dispatch]);
};

export const useToggleTodo = () => {
  const dispatch = useReduxDispatch<Action>();
  return useCallback((id: number) => {
    dispatch({
      type: 'TOGGLE_TODO',
      id,
    });
  }, [dispatch]);
};
```

They are not really action creators, but hooks that return action dispatchers.

### Components

We don't distinguish container components from container components.
It could be still open for discussion about how to structure components,
but for this example, components are just flat.

#### ./src/components/App.tsx

App is also identical to the original example.

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

#### .src/components/Todo.tsx

There are small modifications but not major.

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

#### .src/components/VisibleTodoList.tsx

We don't have `mapStateToProps` or selectors.
`getVisibleTodos` is simply called in render.

```jsx
import * as React from 'react';
import { useReduxState } from 'reactive-react-redux';

import { TodoType, State, VisibilityFilterType } from '../types';
import { useToggleTodo } from '../actions';
import Todo from './Todo';

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

const VisibleTodoList: React.FC = () => {
  const state = useReduxState<State>();
  const visibleTodos = getVisibleTodos(state.todos, state.visibilityFilter);
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

#### .src/components/FilterLink.tsx

Again, as `useReduxState` return the entire state,
it simply uses its property to evaluate `active`.

```jsx
import * as React from 'react';
import { useReduxState } from 'reactive-react-redux';

import { useSetVisibilityFilter } from '../actions';
import { State, VisibilityFilterType } from '../types';

type Props = {
  filter: VisibilityFilterType;
};

const FilterLink: React.FC<Props> = ({ filter, children }) => {
  const state = useReduxState<State>();
  const active = filter === state.visibilityFilter;
  const setVisibilityFilter = useSetVisibilityFilter();
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

#### .src/components/Footer.tsx

Because we rely on type checking,
it's OK to pass strings to filter prop to FilterLink.

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

#### .src/components/AddTodo.tsx

This is a bit modified from the original example
to use controlled form with `useState`.

```jsx
import * as React from 'react';
import { useState } from 'react';

import { useAddTodo } from '../actions';

const AddTodo = () => {
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

### Online demo

Please visit the [codesandbox](https://codesandbox.io/s/github/dai-shi/reactive-react-redux/tree/master/examples/11_todolist) and run the example in your browser.

The source code can also be found [here](https://github.com/dai-shi/reactive-react-redux/tree/master/examples/11_todolist).

### For more information

I didn't explain internal details about reactive-react-redux in this post.
Please visit [the GitHub repo](https://github.com/dai-shi/reactive-react-redux)
to see more information which includes
[the list of previous blog posts](https://github.com/dai-shi/reactive-react-redux#blogs).
