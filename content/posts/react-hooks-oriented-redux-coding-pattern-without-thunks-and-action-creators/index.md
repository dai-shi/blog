---
title: "React hooks-oriented Redux coding pattern without thunks and action creators"
subtitle: "With TypeScript"
date: 2019-08-12T10:00:00+09:00
tags: ["react", "hooks", "redux", "global-state", "async"]
---

### Motivation

I love Redux. But that doesn't mean I like all parts of Redux eco system.
Some people dislike Redux because of its boilerplate code. That's sad.
Boilerplate code is not from the Redux core, but from the eco system.
Don't get me wrong. Best practices are nice and I think
the recent work of Redux Starter Kit is great. (Claps to Mark)

I think I have my own understanding of how to use Redux with React.
It may not be common and probably it will never be the mainstream.
I understand Redux is useful and tuned for larger applications.
What I have in mind is the usage for smaller apps and for beginners.

For smaller apps and for beginners, there seems to be several hurdles.
The first one for me was `mapStateToProps`.
I developed
[reactive-react-redux](https://github.com/dai-shi/reactive-react-redux)
to solve it. 
It provides super simple `useTrackedState`.
It was developed before the Redux hooks API is available.
Now, `useSelector` from the new hooks API is so nice.
It's much less ugly than `mapStateToProps`.
Note that `useTrackedState` is still easier,
because it doesn't require memoization for optimization.

Another hurdle for me is async actions.
I generally like the middleware system of Redux and the elegance
of the implementation of redux-thunk.
But, I find some difficulties with it. Basically, it's too flexible.
It's like exposing the middleware system to userland, to some extent.
Just like people misuse selectors having heavy computation,
people misuse thunks, or overuse them.
redux-observable and redux-saga seem to provide better abstraction
but they are complex systems. They would fit with larger apps.

So, in this post, I would like to show example code as an alternative pattern.
It doesn't use middleware, but React custom hooks.
Here're some points in this pattern.

- No async libraries (Run async tasks outside of Redux)
- No action creators (Define action types in TypeScript)

Without a word, let's dive into the code.

(By the way, yet another hurdle for me is `combineReducers`,
but it's out of the scope of this post.)

### Example

The example to use is [Async Actions](https://redux.js.org/advanced/async-actions) in the official Redux Advanced Tutorial.

### Code

##### Folder structure

```yaml
- src/
  - index.tsx
  - store/
    - actions.ts
    - reducers.ts
  - hooks/
    - useSelectSubreddit.ts
    - useInvalidateSubreddit.ts
    - useFetchPostsIfNeeded.ts
  - components/
    - App.tsx
    - Picker.tsx
    - Posts.tsx
```

##### ./src/index.tsx

```jsx
import * as React from 'react';
import { render } from 'react-dom';
import { Provider } from 'react-redux';

import rootReducer from './store/reducers';
import App from './components/App';

const store = createStore(rootReducer);

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('app'),
);
```

This is the entry point. Nothing special in this file.

##### src/store/actions.ts

```typescript
export type Post = {
  id: string;
  title: string;
};

export type SubredditPosts = {
  isFetching: boolean;
  didInvalidate: boolean;
  items: Post[];
  lastUpdated?: number;
};

export type PostsBySubreddit = {
  [subreddit: string]: SubredditPosts;
};

export type SelectedSubreddit = string;

export type State = {
  selectedSubreddit: SelectedSubreddit;
  postsBySubreddit: PostsBySubreddit;
};

type SelectSubredditAction = {
  type: 'SELECT_SUBREDDIT';
  subreddit: string;
};

type InvalidateSubredditAction = {
  type: 'INVALIDATE_SUBREDDIT';
  subreddit: string;
};

type RequestPostsAction = {
  type: 'REQUEST_POSTS';
  subreddit: string;
};

type ReceivePostsAction = {
  type: 'RECEIVE_POSTS';
  subreddit: string;
  posts: Post[];
  receivedAt: number;
};

export type Action =
  | SelectSubredditAction
  | InvalidateSubredditAction
  | RequestPostsAction
  | ReceivePostsAction;
```

This defines `State` and `Action` types.
No action constants and no action creators are defined.

##### src/store/reducers.ts

```typescript
import { combineReducers } from 'redux';
import {
  SubredditPosts,
  SelectedSubreddit,
  PostsBySubreddit,
  State,
  Action,
} from './actions';

const selectedSubreddit = (
  state: SelectedSubreddit = 'reactjs',
  action: Action,
): SelectedSubreddit => {
  switch (action.type) {
    case 'SELECT_SUBREDDIT':
      return action.subreddit;
    default:
      return state;
  }
};

const posts = (state: SubredditPosts = {
  isFetching: false,
  didInvalidate: false,
  items: [],
}, action: Action): SubredditPosts => {
  switch (action.type) {
    case 'INVALIDATE_SUBREDDIT':
      return {
        ...state,
        didInvalidate: true,
      };
    case 'REQUEST_POSTS':
      return {
        ...state,
        isFetching: true,
        didInvalidate: false,
      };
    case 'RECEIVE_POSTS':
      return {
        ...state,
        isFetching: false,
        didInvalidate: false,
        items: action.posts,
        lastUpdated: action.receivedAt,
      };
    default:
      return state;
  }
};

const postsBySubreddit = (
  state: PostsBySubreddit = {},
  action: Action,
): PostsBySubreddit => {
  switch (action.type) {
    case 'INVALIDATE_SUBREDDIT':
    case 'RECEIVE_POSTS':
    case 'REQUEST_POSTS':
      return {
        ...state,
        [action.subreddit]: posts(state[action.subreddit], action),
      };
    default:
      return state;
  }
};

const rootReducer = combineReducers<State>({
  postsBySubreddit,
  selectedSubreddit,
});

export default rootReducer;
```

This is a normal reducer file with type annotations.
Note that we don't use any explicit and implicit `any`.

##### src/hooks/useSelectSubreddit.ts

```typescript
import { useCallback } from 'react';
import { useDispatch } from 'react-redux';

import { Action } from '../store/actions';

const useSelectSubreddit = () => {
  const dispatch = useDispatch<Action>();
  const selectSubreddit = useCallback((subreddit: string) => {
    dispatch({
      type: 'SELECT_SUBREDDIT',
      subreddit,
    });
  }, [dispatch]);
  return selectSubreddit;
};

export default useSelectSubreddit;
```

This is something instead of an action creator.
It's a hook to return a callback function
that creates and dispatches an action.
Let's call it "an action hook."
This one is a sync action hook.

##### src/hooks/useInvalidateSubreddit.ts

```typescript
import { useCallback } from 'react';
import { useDispatch } from 'react-redux';

import { Action } from '../store/actions';

const useInvalidateSubreddit = () => {
  const dispatch = useDispatch<Action>();
  const invalidateSubreddit = useCallback((subreddit: string) => {
    dispatch({
      type: 'INVALIDATE_SUBREDDIT',
      subreddit,
    });
  }, [dispatch]);
  return invalidateSubreddit;
};

export default useInvalidateSubreddit;
```

This is another sync action hook.

##### src/hooks/useFetchPostsIfNeeded.ts

```typescript
import { useCallback } from 'react';
import { useDispatch, useStore } from 'react-redux';

import { Action, State, Post } from '../store/actions';

const shouldFetchPosts = (state: State, subreddit: string) => {
  const posts = state.postsBySubreddit[subreddit];
  if (!posts) {
    return true;
  }
  if (posts.isFetching) {
    return false;
  }
  return posts.didInvalidate;
};

const extractPosts = (json: unknown): Post[] | null => {
  try {
    const posts: Post[] = (json as {
      data: {
        children: {
          data: {
            id: string;
            title: string;
          };
        }[];
      };
    }).data.children.map(child => child.data);
    // type check
    if (posts.every(post => (
      typeof post.id === 'string' && typeof post.title === 'string'
    ))) {
      return posts;
    }
    return null;
  } catch (e) {
    return null;
  }
};

const useFetchPostsIfNeeded = () => {
  const dispatch = useDispatch<Action>();
  const store = useStore<State>();
  const fetchPostsIfNeeded = useCallback(async (subreddit: string) => {
    if (!shouldFetchPosts(store.getState(), subreddit)) {
      return;
    }
    dispatch({
      type: 'REQUEST_POSTS',
      subreddit,
    });
    const response = await fetch(`https://www.reddit.com/r/${subreddit}.json`);
    const json = await response.json();
    const posts = extractPosts(json);
    if (!posts) throw new Error('unexpected json format');
    dispatch({
      type: 'RECEIVE_POSTS',
      subreddit,
      posts,
      receivedAt: Date.now(),
    });
  }, [dispatch, store]);
  return fetchPostsIfNeeded;
};

export default useFetchPostsIfNeeded;
```

This one is an async action hook.
This is the logic that would be written with thunk.
There are a few important points in this file.

- Because this is not middleware, we don't have direct access to the state.
  It uses `useStore`, which is something we shouldn't misuse.
  This is the biggest caveat in this entire pattern.
- `extractPosts` is a kind of type guard to test json from the network.
- We don't implement error handing as is [in the original tutorial](https://redux.js.org/advanced/async-actions#note-on-error-handling).

##### src/components/App.tsx

```tsx
import * as React from 'react';
import { useCallback, useEffect } from 'react';
import { useSelector } from 'react-redux';

import { State, SelectedSubreddit } from '../store/actions';
import useSelectSubreddit from '../hooks/useSelectSubreddit';
import useFetchPostsIfNeeded from '../hooks/useFetchPostsIfNeeded';
import useInvalidateSubreddit from '../hooks/useInvalidateSubreddit';

import Picker from './Picker';
import Posts from './Posts';

const App: React.FC = () => {
  const selectedSubreddit = useSelector((state: State) => state.selectedSubreddit);
  const postsBySubreddit = useSelector((state: State) => state.postsBySubreddit);
  const {
    isFetching,
    items: posts,
    lastUpdated,
  } = postsBySubreddit[selectedSubreddit] || {
    isFetching: true,
    items: [],
    lastUpdated: undefined,
  };

  const fetchPostsIfNeeded = useFetchPostsIfNeeded();
  useEffect(() => {
    fetchPostsIfNeeded(selectedSubreddit);
  }, [fetchPostsIfNeeded, selectedSubreddit]);

  const selectSubreddit = useSelectSubreddit();
  const handleChange = useCallback((nextSubreddit: SelectedSubreddit) => {
    selectSubreddit(nextSubreddit);
  }, [selectSubreddit]);

  const invalidateSubreddit = useInvalidateSubreddit();
  const handleRefreshClick = (e: React.MouseEvent) => {
    e.preventDefault();
    invalidateSubreddit(selectedSubreddit);
    fetchPostsIfNeeded(selectedSubreddit);
  };

  const isEmpty = posts.length === 0;
  return (
    <div>
      <Picker
        value={selectedSubreddit}
        onChange={handleChange}
        options={['reactjs', 'frontend']}
      />
      <p>
        {lastUpdated && (
          <span>
            Last updated at {new Date(lastUpdated).toLocaleTimeString()}.
            {' '}
          </span>
        )}
        {!isFetching && (
          <button type="button" onClick={handleRefreshClick}>
            Refresh
          </button>
        )}
      </p>
      {isEmpty && isFetching && <h2>Loading...</h2>}
      {isEmpty && !isFetching && <h2>Empty.</h2>}
      {!isEmpty && (
        <div style={{ opacity: isFetching ? 0.5 : 1 }}>
          <Posts posts={posts} />
        </div>
      )}
    </div>
  );
};

export default App;
```

This is a root component or a container component.
Unfortunately, the code looks like boilerplate.
But, it should be mostly the same with a normal React app.
I think the second caveat in this pattern is requiring
the `useCallback` hook.

##### src/components/Picker.tsx

```tsx
import * as React from 'react';

const Picker: React.FC<{
  value: string;
  onChange: (value: string) => void;
  options: string[];
}> = ({ value, onChange, options }) => (
  <span>
    <h1>{value}</h1>
    <select
      onChange={e => onChange(e.target.value)}
      value={value}
    >
      {options.map(option => (
        <option value={option} key={option}>
          {option}
        </option>
      ))}
    </select>
  </span>
);

export default Picker;
```

This is a stateless component.
Nothing is changed except for type annotations.

##### src/components/Posts.tsx

```tsx
import * as React from 'react';

const Posts: React.FC<{
  posts: {
    id: string;
    title: string;
  }[];
}> = ({ posts }) => (
  <ul>
    {posts.map(post => (
      <li key={post.id}>{post.title}</li>
    ))}
  </ul>
);

export default Posts;
```

This is another stateless component.
We could import `Post` from `actions.ts`.

That's everything. We are all set.

### Demo

<iframe src="https://codesandbox.io/embed/github/dai-shi/reactive-react-redux/tree/master/examples/12_async?fontsize=14" title="reactive-react-redux-example" allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

[Source code in the repo](https://github.com/dai-shi/reactive-react-redux/tree/master/examples/12_async)

Notice this code is based on
[reactive-react-redux](https://github.com/dai-shi/reactive-react-redux)
instead of react-redux. reactive-react-redux has a compatible hooks API
with react-redux, except for `useStore`.
In this demo, `useStore` is implemented with another context.

### Closing notes

This coding pattern may not be new, and I'm sure somebody else
has already tried it out.
However, it makes more sense with React hooks and TypeScript.
It can eliminate some boilerplate code.
This example uses `isFetching` flag to show a loading status,
but that will change with React Suspense.
This pattern should ease the transition to React Suspense.
