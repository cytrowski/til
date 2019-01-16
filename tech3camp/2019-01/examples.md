# Bulletproof ducks
## Redux meets TypeScript

Bartosz Cytrowski / 2019-01-15

[@cytrowski](https://twitter.com/cytrowski)

#### Let's take a basic reducer for simple counter app:

```js
// Action types
const INCREMENT = "counter/INCREMENT";
const DECREMENT = "counter/DECREMENT";
const RESET = "counter/RESET";

// Action creators
export const increment = payload => ({
  type: INCREMENT,
  payload
});

export const decrement = payload => ({
  type: DECREMENT,
  payload
});

export const reset = payload => ({
  type: RESET,
  payload
});

// Initial state
const initialState = {
  value: 0
};

// Reducer
export default (state = initialState, action) => {
  switch (action.type) {
    case INCREMENT:
      return {
        ...state,
        value: state.value + action.payload.delta
      };
    case DECREMENT:
      return {
        ...state,
        value: state.value - action.payload.delta
      };
    case RESET:
      return initialState;
    default:
      return state;
  }
};
```

#### Let's assume we are alergic to `switch` statements and we want to make it smaller:

```js
// ...

// LOOK MAAA - new function in the scope :D
// we have to give it a name :( NAMING IS HARD.
const handleIncrement = (state, payload) => ({
  ...state,
  value: state.value + payload.delta
});

// Reducer
export default (state = initialState, action) => {
  switch (action.type) {
    case INCREMENT:
      return handleIncrement(state, action.payload); // Well, this case is at least a little bit shorter
    case DECREMENT:
      return {
        ...state,
        value: state.value - action.payload.delta
      };
    case RESET:
      return initialState;
    default:
      return state;
  }
};
```

#### Let's apply the previous pattern for all `case` branches:

```js
// ...

const handleIncrement = (state, payload) => ({
  // NAMES :O
  ...state,
  value: state.value + payload.delta
});

const handleDecrement = (state, payload) => ({
  // NAMES :O
  ...state,
  value: state.value - payload.delta
});

const handleReset = () => initialState; // NAMES :O

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  switch (action.type) {
    case INCREMENT:
      return handleIncrement(state, action.payload); // MUCH CHARACTERS :<
    case DECREMENT:
      return handleDecrement(state, action.payload); // BIG DISLIKE :<
    case RESET:
      return handleReset(state, action.payload);
    default:
      return identity(state, action.payload);
  }
};
```

#### What if we group handlers within object and name them after related actions?

```js
// ...
const handlers = {
  INCREMENT: (state, payload) => ({
    ...state,
    value: state.value + payload.delta
  }),
  DECREMENT: (state, payload) => ({
    ...state,
    value: state.value - payload.delta
  }),
  RESET: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  switch (action.type) {
    case INCREMENT:
      return handlers.INCREMENT(state, action.payload);
    case DECREMENT:
      return handlers.DECREMENT(state, action.payload);
    case RESET:
      return handlers.RESET(state, action.payload);
    default:
      return identity(state, action.payload);
  }
};
```

#### We can do better with dynamic property names - STEP 1

```js
// ...
const handlers = {
  [INCREMENT]: (state, payload) => ({
    ...state,
    value: state.value + payload.delta
  }),
  [DECREMENT]: (state, payload) => ({
    ...state,
    value: state.value - payload.delta
  }),
  [RESET]: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  switch (action.type) {
    case INCREMENT:
      return handlers[INCREMENT](state, action.payload);
    case DECREMENT:
      return handlers[DECREMENT](state, action.payload);
    case RESET:
      return handlers[RESET](state, action.payload);
    default:
      return identity(state, action.payload);
  }
};
```

#### ... and STEP 2 (but still not perfect)

```js
// ...
const handlers = {
  [INCREMENT]: (state, payload) => ({
    ...state, // <- I HATE WE HAVE TO REMEMBER ABOUT THIS EVERY TIME
    value: state.value + payload.delta
  }),
  [DECREMENT]: (state, payload) => ({
    ...state, // <- I HATE WE HAVE TO REMEMBER ABOUT THIS EVERY TIME
    value: state.value - payload.delta
  }),
  [RESET]: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) =>
  (handlers[action.type] || identity)(state, action.payload);
```

#### ... remove duplication :)

```js
// ...
const handlers = {
  [INCREMENT]: (state, payload) => ({
    value: state.value + payload.delta
  }),
  [DECREMENT]: (state, payload) => ({
    value: state.value - payload.delta
  }),
  [RESET]: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  const result = (handlers[action.type] || identity)(state, action.payload);
  return result === state || result === initialState
    ? result
    : { ...state, ...result }; // <- NOW WE HAVE IT IN ONE PLACE
};
```

#### ... almost perfect - let's add some destructuring to the mixture :D

```js
// ...

// NOW THAT FEELS SHORT :D

const handlers = {
  [INCREMENT]: ({ value }, { delta }) => ({
    value: value + delta
  }),
  [DECREMENT]: ({ value }, { delta }) => ({
    value: value - delta
  }),
  [RESET]: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  const result = (handlers[action.type] || identity)(state, action.payload);
  return result === state || result === initialState
    ? result
    : { ...state, ...result };
};
```

#### OK - What about the guys at the top of this file - we almost forgot about them:

```js
// Action types
const INCREMENT = "counter/INCREMENT";
const DECREMENT = "counter/DECREMENT";
const RESET = "counter/RESET";

// Action creators
export const increment = payload => ({
  type: INCREMENT,
  payload
});

export const decrement = payload => ({
  type: DECREMENT,
  payload
});

export const reset = payload => ({
  type: RESET,
  payload
});

// Initial state
const initialState = {
  value: 0
};

const handlers = {
  [INCREMENT]: ({ value }, { delta }) => ({
    value: value + delta
  }),
  [DECREMENT]: ({ value }, { delta }) => ({
    value: value - delta
  }),
  [RESET]: () => initialState
};

const identity = x => x;

// Reducer
export default (state = initialState, action) => {
  const result = (handlers[action.type] || identity)(state, action.payload);
  return result === state || result === initialState
    ? result
    : { ...state, ...result };
};
```

#### If we look at INCREMENT action we can find its name in 5 places:

```js
// Action types
const INCREMENT = "counter/INCREMENT"; // 1 and 2

// ...

// Action creators
export const increment = payload => ({
  // 3
  type: INCREMENT, // 4
  payload
});

// ...

const handlers = {
  [INCREMENT]: ({ value }, { delta }) => ({
    // 5
    value: value + delta
  }),
  [DECREMENT]: ({ value }, { delta }) => ({
    value: value - delta
  }),
  [RESET]: () => initialState
};
```

#### What if I tell you we can shorten it to only 2 occurences? Just like that:

```js
import { makeReduxDuck } from "teedux";

// Initial state
const initialState = {
  value: 0
};

const duck = makeReduxDuck("counter", initialState);

export const increment = duck.defineAction(
  "INCREMENT",
  ({ value }, { delta }) => ({
    value: value + delta
  })
);

// ...

// Reducer
export default duck.getReducer();
```

#### Which with TypeScript looks like that:

```js
import { makeReduxDuck } from "teedux";

interface IState {
  // Look Maaa - we have an Interface here
  value: number;
}

// Initial state
const initialState: IState = {
  value: 0
};

const duck = makeReduxDuck("counter", initialState);

export const increment = duck.defineAction<{ delta: number}>( // One simple generic :)
  "INCREMENT",
  ({ value }, { delta }) => ({
    value: value + delta
  })
);


// ...

// Reducer
export default duck.getReducer();
```

#### You may think about how to use it with combineReducers - well :)

```js
import { createStore, combineReducers } from "redux";
import { composeWithDevTools } from "redux-devtools-extension";

import counter from "./ducks/counter";

const reducer = combineReducers({
  counter
});

export type TRootState = ReturnType<typeof reducer>; // <- LOOK HERE

const store = createStore(reducer, composeWithDevTools());

export default store;
```

#### How to make a connectable HOC?

```js
import { connect } from "react-redux";
import { increment, decrement, reset } from "../../state/ducks/counter";
import { TRootState } from "../../state/store"; // TS here

const mapStateToProps = (state: TRootState) => ({ // ... here
  counterValue: state.counter.value
});

const mapDispatchToProps = {
  increment,
  decrement,
  reset
};

// ... and here :)
export type TConnectableProps = 
  ReturnType<typeof mapStateToProps> &
  typeof mapDispatchToProps; 

export const Connectable = connect(
  mapStateToProps,
  mapDispatchToProps
);
```

#### Look at the component:

```jsx
import React, { Component, ComponentType } from "react";

import { Connectable, TConnectableProps } from "./Connectable.hoc";

interface IOwnProps {}

type TProps = IOwnProps & TConnectableProps;

class App extends Component<TProps> {
  render() {
    return (
      <div>
        <p>Counter value is: {this.props.counterValue}</p>
        <p>
          <button
            onClick={() => this.props.increment({ delta: 10 })}
          >
            Increment
          </button>
        </p>
      </div>
    );
  }
}

export default Connectable(App) as ComponentType<IOwnProps>;

```
