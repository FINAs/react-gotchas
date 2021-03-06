# `Reselect`

## Input selectors should be pure

Consider the following:

```javascript
const getFooNames = state => Object.keys(state.foo);
const getMappedBar = state => state.bar.map(x => x * 2);
const getFilteredBaz = state => state.baz.filter(x => !!x);
const nullCheckedQuux = state => state.quux || {};
```

In all these cases, the input selector is returning a new object:

```javascript
getFooNames(state) === getFooNames(state) // returns false
```

The same input is not leading to the same output; hence, the function is not pure.

What are the consequences? Both have to do with performance:

1. Unnecessary rerenders: if such an input selector is used in `mapStateToProps()`, props will have changed every time `mapStateToProps()` is invoked.
2. Loss of caching: Reselect selectors have a cache depth of 1 by default. If such an input selector is used, the Reselect selector will cache miss every single time.

Generally, this leads to the frustrating scenario where branch A of state changes, and a component connected to branch B rerenders.

## Function signature

Both input and composed selectors should ALWAYS have the function signature `(state)` or `(state, props)`. This is because selectors should be composable. Imagine if you had the following selectors:

```javascript
const getFoo = state => state.foo;
const getBar = props => props.bar;

const getFooAndBar = createSelector(
  [getFoo, getBar],
  (foo, bar) => ({ foo, bar })
);
```

What is the signature of `getFooAndBar()` in this case? Composed selectors have the same function signature as their children... but in this case, the two signatures are mutually exclusive! Both `getFoo()` and `getBar()` will attempt to operate on the first argument passed to `getFooAndBar()`, and because `getFooAndBar()` is most likely passed `state` in `mapStateToProps()`, `getBar()` may not return the expected value.

Instead:

```javascript
const getFoo = state => state.foo;
const getBar = (_, props) => props.bar; // _ just a suggestion

getFooAndBar(state, props) // works as intended
```

## Prop selectors

Only access props in selectors when absolutely necessary. Because selectors can be composed, excessive usage of props makes it hard to track down the dependencies of a selector. Consider the following:

```javascript
const getFlight = (state, props) => state.entities.flights[props.flightId];
const getLineItem = (state, props) => state.entities.lineItems[props.lineItemId];
const getTactic = (state, props) => state.entities.tactics[props.tacticId];

const getStuff = createSelector(
  [getFlight, getLineItem, getTactic],
  (flight, lineItem, tactic) => ({ flight, lineItem, tactic })
);
```

The `getStuff()` selector takes `(state, props)` as arguments. But now there is the implicit requirement that `props.flightId`, `props.lineItemId`, and `props.tacticId` all exist in the component that uses `getStuff()` in `mapStateToProps()`. Combined with the fact that Reselect selectors themselves can be used to compose other selectors, the dependency chain gets messy real fast.

## Do not return functions in selectors

Sometimes it might seem convenient to have a selector return a function, because the functionality required has many data dependencies. Then, the dependencies are passed to the function returned by the selector:

```javascript
const getStuff = createSelector(
  [getFoo],
  foo => props => doSomething(foo, props),
);
```

This is an antipattern, because Reselect will only cache the creation of the function (`foo => someFn`). The returned function itself (`props => doSomething(foo, props)`) will not be cached. Because the returned function will ostensibly be used in `mapStateToProps`, this will have dire performance implications.

See [`mapStateToProps()` Props should be connected in a pure way](https://github.com/timhwang21/react-gotchas/blob/master/mapStateToProps.md#props-should-be-connected-in-a-pure-way) for more information.

Instead,the entirety of `props => doSomething(foo, props)` should be wrapped in a selector, along with all of its dependencies. **The entire point of Reselect is to cache function calls based on their arguments; if Reselect is not exposed to all the arguments, the caching will fail.** To avoid passing Reselect an unwieldy number of imput selectors, selector composition can be used.
