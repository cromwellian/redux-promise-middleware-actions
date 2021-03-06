# redux-promise-middleware-actions

[![Build Status](https://img.shields.io/travis/omichelsen/redux-promise-middleware-actions/master.svg)](https://travis-ci.org/omichelsen/redux-promise-middleware-actions)
[![Coverage Status](https://coveralls.io/repos/omichelsen/redux-promise-middleware-actions/badge.svg?branch=master&service=github)](https://coveralls.io/github/omichelsen/redux-promise-middleware-actions?branch=master)
[![npm bundle size (minified + gzip)](https://img.shields.io/bundlephobia/minzip/redux-promise-middleware-actions.svg)](https://bundlephobia.com/result?p=redux-promise-middleware-actions)

Create Redux actions with a `type` and `payload` in a standardized way. Inspired by [redux-actions](https://www.npmjs.com/package/redux-actions) but simpler and with special support for asynchronous actions (promises).

Has no dependencies and is tiny (~680 bytes gzipped). First class TypeScript support.

Works with [redux-promise-middleware](https://www.npmjs.com/package/redux-promise-middleware) to handle asynchronous actions by dispatching `pending`, `fulfilled` and `rejected` events based on the state of the input promise.

Goals of this library:

* Reference action creators directly - no need to maintain an action type enum/list
* Automatically generate actions for pending, fulfilled and rejected outcomes of a promise payload
* Have statically typed access to all action types - no need to manually add a type suffix like "_PENDING"
* TypeScript support so asynchronous actions can't be confused for normal synchronous actions

Note: If you are using TypeScript this library requires TypeScript 3. For TypeScript 2 use version 1 of this library.

## Installation

You need to install this library as well as [redux-promise-middleware](https://www.npmjs.com/package/redux-promise-middleware).

```
npm install redux-promise-middleware-actions redux-promise-middleware
```

Include redux-promise-middleware when you create your store:

```js
import promiseMiddleware from 'redux-promise-middleware';

composeStoreWithMiddleware = applyMiddleware(
  promiseMiddleware(),
)(createStore);
```

## Usage

### Synchronous action

Synchronous actions works exactly like redux-actions. You supply a function that returns whatever payload the action should have (if any).

```js
import { createAction } from 'redux-promise-middleware-actions';

export const foo = createAction('FOO', (num) => num);

dispatch(foo(5)); // { type: 'FOO', payload: 5 }
```

When handling the action in a reducer, you simply cast the action function to a string to return the type. This ensures type safety (no spelling errors) and you can use code navigation to find all uses of an action.

```js
const fooType = String(foo); // 'FOO'
```

### Asynchronous action

When you create an asynchronous action you need to return a promise payload. If your action is called `FOO` the following events will be dispatched:

1. `FOO_PENDING` is dispatched immediately
2. `FOO_FULFILLED` is dispatched when the promise is resolved
    * ... or `FOO_REJECTED` is dispatched instead if the promise is rejected

```js
import { createAsyncAction } from 'redux-promise-middleware-actions';

export const fetchData = createAsyncAction('FETCH_DATA', async () => {
  const res = await fetch(...);
  return res.json();
});

dispatch(fetchData()); // { type: 'FETCH_DATA_PENDING' }
```

An async action function has three properties to access the possible outcome actions: `pending`, `fulfilled` and `rejected`. You can dispatch them directly (in tests etc.):

```js
dispatch(fetchData.pending());          // { type: 'FETCH_DATA_PENDING' }
dispacth(fetchData.fulfilled(payload)); // { type: 'FETCH_DATA_FULFILLED', payload: ... }
dispacth(fetchData.rejected(err));      // { type: 'FETCH_DATA_REJECTED', payload: err, error: true }
```

But normally you only need them when you are writing reducers:

```js
case String(fetchData.pending):   // 'FETCH_DATA_PENDING'
case String(fetchData.fulfilled): // 'FETCH_DATA_FULFILLED'
case String(fetchData.rejected):  // 'FETCH_DATA_REJECTED'
```

Note that if you try and use the base function in a reducer, an error will be thrown to ensure you are not listening for an action that will never happen:

```js
case String(fetchData): // throws an error
```

### Async reducer

You can now handle the different events in your reducer by referencing the possible outcome states:

```js
import { fetchData } from './actions';

export default (state, action) => {
  switch (action.type) {
    case String(fetchData.pending):
      return {
        ...state,
        pending: true,
      };
    case String(fetchData.fulfilled):
      return {
        ...state,
        data: action.payload,
        error: undefined,
        pending: false,
      };
    case String(fetchData.rejected):
      return {
        ...state,
        error: action.payload,
        pending: false,
      };
    default:
      return state;
  }
};
```

#### Async reducer helper

It can get tedious writing the same reducer for every single async action so we've included a simple reducer that does the same as the example above:

```js
import { asyncReducer } from 'redux-promise-middleware-actions';
import { fetchData } from './actions';

export default asyncReducer(fetchData);
```

You can also combine it with an existing reducer:

```js
import { asyncReducer } from 'redux-promise-middleware-actions';
import { fetchData } from './actions';

const fetchReducer = asyncReducer(fetchData);

export default (state, action) => {
  const newState = fetchReducer(state, action);

  switch (action.type) {
    case 'SOME_OTHER_ACTION':
      return { ... };
    default:
      return newState;
  }
};
```

### Metadata

You can add metadata to any action by supplying an additional metadata creator function. The metadata creator will receive the same arguments as the payload creator:

#### `createAction(type, payloadCreator, metadataCreator)`

```js
export const foo = createAction(
  'FOO',
  (num) => num,
  (num) => num + num
);

dispatch(foo(5)); // { type: 'FOO', meta: 10, payload: 5 }
```

#### `createAsyncAction(type, payloadCreator, metadataCreator)`

```js
export const fetchData = createAsyncAction(
  'FETCH_DATA',
  (n: number) => fetch(...),
  (n: number) => ({ n })
);

dispatch(fetchData(42));
// { type: 'FETCH_DATA_PENDING', meta: { n: 42 } }
// { type: 'FETCH_DATA_FULFILLED', meta: { n: 42 }, payload: Promise<...> }
// { type: 'FETCH_DATA_REJECTED', meta: { n: 42 }, payload: Error(...) }
```
