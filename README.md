# @avaragado/xstateful-react

> Use [`@avaragado/xstateful`](https://github.com/avaragado/xstateful) with React, accessing states and activities from multiple statecharts anywhere in your app

See https://codesandbox.io/s/6lyq0yl4rz for a full example, simulating UK-style traffic lights and a pedestrian crossing (the source is in the `examples/pelican` directory).

## Features

-   **Provider/consumer model** Call a function to create React components from an `XStateful` instance. Add the provider near the app root, and use consumers lower down the tree to access machine state and extended state.
-   **Specialised consumer components** Use declarative components that render based on machine activities, machine state, or any function.
-   **Familiar props** Consumer components accept `component` prop, `render` prop, function-as-child, or child nodes, very similar to `react-router`.
-   **Lifecycle helper** A special component lets you "set up" and "tear down" a machine on mount/unmount if you need to.

### Requirements

-   React 16.3+ (uses React's new "context" functionality)

## Why?

The [`@avaragado/xstateful`](https://github.com/avaragado/xstateful) package is a self-contained interpreter for statecharts, wrapping `xstate` and adding support for reducers, extended state, and time-based events. `xstateful` itself doesn't render anything.

`xstateful-react` works with `xstateful` to let you render React components based on your statechart, and trigger state transitions.

## Terminology

Terms are as used in `xstateful`, plus common terms in React development such as _render props_, _function-as-child_, and the provider/consumer pattern of React Context.

## Installation

```bash
$ yarn add xstate @avaragado/xstateful @avaragado/xstateful-react
$ # or
$ npm install xstate @avaragado/xstateful @avaragado/xstateful-react
```

## Getting started

In summary:

1.  Use `xstateful` to create an `XStateful` instance (either instantiating `XStateful` directly, or by calling `createStatefulMachine`).
1.  Call the `xstateful-react` function `createReactMachine`, passing your `XStateful` instance. This function returns a number of components (a _provider_ and several _consumers_).
1.  Add the provider component somewhere near the root of your app.
1.  Add the consumer components as necessary as direct or indirect descendants of the provider.
1.  Add the control component if needed to set up/tear down your machine at the right places in your app.

### Example

Use one module to export an `XStateful` instance and the React components generated by `xstateful-react`. You can test the `XStateful` instance in isolation, without worrying about particular rendering environments.

```js
// my-react-machine.js

import { Machine } from 'xstate';
import { createStatefulMachine } from '@avaragado/xstateful';
import { createReactMachine } from '@avaragado/xstateful-react';

const machine = Machine({
    // xstate machine configuration
});
const reducer = ... // if needed
const extstate = ... // if needed

export const xsf = createStatefulMachine({ machine, reducer, extstate });

export default createReactMachine(xsf);
```

Near the root of the render tree, add the provider component (it accepts no props). It doesn't have to look exactly like this: as long as the provider is an ancestor of all the consumers, it's fine.

The object returned by `createReactMachine` is a general-purpose consumer component, described in full below. It has properties for the other, more specialised consumers, plus a property for the provider, and one for the control component. This structure is intended to "read well" to aid understanding when inspecting a component tree.

```js
// my-root.jsx

import React from 'react';
import ReactDOM from 'react-dom';

import MyApp from './my-app';
import MyReactMachine from './my-react-machine';

ReactDOM.render(
    <MyReactMachine.Provider>
        <MyApp />
    </MyReactMachine.Provider>,
    document.getElementById('root'),
);
```

Lower down the render tree, render the consumer components. They all accept `component`, `render` and `children` props that work almost identically to `react-router`. They also nest, with predictable results. Full descriptions of each component are below – this example component just shows a few options.

```js
// my-random-component.jsx

import React from 'react';

import MyReactMachine from './my-react-machine';

const MyRandomComponent = () => (
    <div>
        <p>Blah...</p>

        {/* the main component renders according to a function of state and extended state */}
        <MyReactMachine
            cond={({ state, extstate }) =>
                state.value === 'boink' && extstate.foo === 'bar'
            }
        >
            <p>Rendered if cond evaluates to true</p>
        </MyReactMachine>

        {/* the .Activity component renders according to current machine activities */}
        <MyReactMachine.Activity is="pending">
            Loading...
        </MyReactMachine.Activity>
        <MyReactMachine.Activity is={['error', 'timeout']}>
            Something went wrong
        </MyReactMachine.Activity>

        {/* the .State component renders according to current machine state value */}
        <MyReactMachine.State is="a.b.c"> ... </MyReactMachine.State>
        <MyReactMachine.State is={['a.b.c', 'a.b.d']}>...</MyReactMachine.State>
    </div>
);

export default MyRandomComponent;
```

Some `XStateful` machines may need special "set up" and "tear down" behaviour. For example, consider a machine that sends an event on a periodic timer when it's in a particular state. This machine keeps sending that event even when it's not mounted in the component tree, for as long as it remains in this state. (This is because the `XStateful` instance is decoupled from React.)

This might be what's needed for an app: every app is different. Some apps might instead want to "power down" a machine when it's not mounted. Use the `Control` component to do this.

```js
// my-other-component.jsx

import React from 'react';

import MyReactMachine from './my-react-machine';

const MyOtherComponent = () => (
    <MyReactMachine.Control
        onDidMount={({ transition }) => transition('POWER_ON');}
        onWillUnmount={({ transition }) => transition('POWER_OFF');}
    >
        ...other components, including consumer components from MyReactMachine
    </MyReactMachine.Control
);

export default MyOtherComponent;
```

## Reference

### Module exports

```js
import { createReactMachine } from '@avaragado/xstateful-react';
```

### `createReactMachine` function

#### `createReactMachine(XStateful) => React.ComponentType`

Returns a general-purpose React consumer component (let's call it `Machine`) tied to the input `XStateful` instance. `Machine` holds other components, as properties `Activity`, `State`, `Provider` and `Control`.

### `Machine`

This is a React component that gives access to values from `Machine.Provider` rendered higher in the tree. Use these values, through props, to determine whether to render other components.

`Machine` props:

-   `cond?: boolean | (({ state, extstate }) => boolean)`
-   `component?: React.ComponentType<{ state, extstate, init, transition, setExtState }>`
-   `render?: ({ state, extstate, init, transition, setExtState }) => React.Node`
-   `children?: React.Node | ({ state, extstate, init, transition, setExtState, match }) => React.Node`

The arguments/props `state`, `extstate`, `init`, `transition` and `setExtState` correspond to the `XStateful` instance: `state` contains machine state, `extstate` contains extended state, `init` is a function to initialise or reset the machine, `transition` is a function to send an event to the machine, and `setExtState` is a function to update extended state.

If more than one of `component`, `render` and `children` are specified, `component` takes precedence over `render`, and `render` takes precedence over `children`.

The boolean value from `cond` (which defaults to `true` if `cond` is omitted), combined with `component`/`render`/`children`, define what's rendered.

-   With `component`:
    -   When `cond` is true, creates a React element from that component, passing the props `state`, `extstate`, `init`, `transition` and `setExtState`, and renders that.
    -   When `cond` is false, renders `null`.
-   With `render`:
    -   When `cond` is true, calls the render prop function passing a single object arg `{ state, extstate, init, transition, setExtState }` and renders the result.
    -   When `cond` is false, renders `null`.
-   With `children` nodes (not function-as-child form):
    -   When `cond` is true, renders the children.
    -   When `cond` is false, renders `null`.
-   With `children` function:
    -   Calls the function, passing a single object arg `{ state, extstate, init, transition, setExtState, match }`, where `match` is the boolean result of `cond`, and renders the result.

Examples:

```js
<Machine cond={other_value_in_scope}>
    <p>Rendered only if cond value is true</p>
</Machine>

<Machine cond={mc => mc.extstate.foo === 123}>
    <p>Rendered only if cond evaluates to true</p>
</Machine>

<Machine cond={mc => mc.extstate.foo === 123}>
    {({ match }) => (
        <p>Rendered always, cond result is in match</p>
    )}
</Machine>

<Machine
    cond={mc => mc.state.actions.length === 0}
    component={RenderedOnlyIfCondTrue}
/>

<Machine
    cond={mc => otherfunction(mc, othervalue)}
    render={
        ({ state }) => (<p>rendered only if cond true</p>)
    }
/>

<Machine>
    {({ state, extstate, init, transition, setExtState }) => {
        // render something!
    }}
</Machine>
```

### `Machine.Activity`

This is a React component that sprinkles some sugar over `Machine`, focusing on the activities emitted by the statechart.

`Machine.Activity` props:

-   `is?: string | Array<string> | (({ [activity: string]: boolean }) => boolean)`
-   `not?: string | Array<string> | (({ [activity: string]: boolean }) => boolean)`
-   `component?: React.ComponentType<{ state, extstate, init, transition, setExtState }>`
-   `render?: ({ state, extstate, init, transition, setExtState }) => React.Node`
-   `children?: React.Node | ({ state, extstate, init, transition, setExtState, match }) => React.Node`

The `is` and `not` props check against the statechart's current activities.

-   `is="foo"` matches if the string is a current activity.
-   `not="foo"` matches if the string is not a current activity.
-   `is={['foo', 'bar']}` matches if any of the array members is a current activity.
-   `not={['foo', 'bar']}` matches if none of the array members is a current activity.
-   `is={myFunction}` matches if the function, when passed an object `{ [activity: string]: boolean }` describing the statechart's current activities, returns true.
-   `not={myFunction}` matches if the function, when passed an object `{ [activity: string]: boolean }` describing the statechart's current activities, returns false.

The boolean result of the match feeds in to the `component`, `render` and `children` props as described above for `Machine`.

Examples:

```js
<Machine.Activity is="buzzing">
    <p>Power is on!</p>
</Machine.Activity>

<Machine.Activity
    not={['fizzing', 'buzzing']}
    component={MostlyHarmless}
/>

<Machine.Activity is="open">
    {({ match }) => (
        <p>match is true if activity "open" is current, false otherwise</p>
    )}
</Machine.Activity>
```

### `Machine.State`

This is a React component that sprinkles some sugar over `Machine`, focusing on the current state(s) of the statechart.

`Machine.State` props:

-   `is?: string | Array<string> | (({ [activity: string]: boolean }) => boolean)`
-   `not?: string | Array<string> | (({ [activity: string]: boolean }) => boolean)`
-   `component?: React.ComponentType<{ state, extstate, init, transition, setExtState }>`
-   `render?: ({ state, extstate, init, transition, setExtState }) => React.Node`
-   `children?: React.Node | ({ state, extstate, init, transition, setExtState, match }) => React.Node`

The `is` and `not` props check against the statechart's current states.

-   `is="foo"` matches if the string is a current state.
-   `not="foo"` matches if the string is not a current state.
-   `is={['foo', 'bar']}` matches if any of the array members is a current state.
-   `not={['foo', 'bar']}` matches if none of the array members is a current state.
-   `is={myFunction}` matches if the function, when passed an `xstate` state value, returns true.
-   `not={myFunction}` matches if the function, when passed an `xstate` state value, returns false.

The state check uses the `xstate` utility `matchesState`, and supports parallel and nested states. For example, if the statechart is currently in states `a.b.c` and `a.b.d`, then a check `is="a.b"` will match.

The boolean result of the match feeds in to the `component`, `render` and `children` props as described above for `Machine`.

Examples:

```js
<Machine.State is="idle">
    <p>Waiting for input</p>
</Machine.State>

<Machine.State
    not={['a.b', 'a.c']}
    render={({ exstate }) => (
        <Something val={extstate.foo} />
    )}
/>

<Machine.State is={someComplexFunctionOfStateValue}>
    {({ match }) => (
        <p>match is boolean result of function</p>
    )}
</Machine.State>
```

### `Machine.Provider`

This React component holds the link to the `XStateful` instance for the statechart. It provides current machine state and extended state values, and functions to send events, to all related consumer components (`Machine`, `Machine.Activity`, `Machine.State`) rendered as descendants in the render tree.

No props.

### `Machine.Control`

This React component includes two props that map to React lifecycle methods. Use these props to initialise and/or send events to the statechart when the React component mounts and/or unmounts. Not all apps need to use it.

`Machine.Control` props:

-   `onDidMount?: ({ state, extstate, init, transition, setExtState }) => void`
-   `onWillUnmount?: ({ state, extstate, init, transition, setExtState }) => void`
-   `children: React.Node`

The arguments/props `state`, `extstate`, `init`, `transition` and `setExtState` correspond to the `XStateful` instance: `state` contains machine state, `extstate` contains extended state, `init` is a function to initialise or reset the machine, `transition` is a function to send an event to the machine, and `setExtState` is a function to update extended state.

The component always renders its children.

The component calls the `onDidMount` function in its own `componentDidMount` lifecycle method, and `onWillUnmount` in its own `componentWillUnmount` lifecycle method.

Examples:

```js
<Machine.Control onDidMount={({ init }) => init()}>
    ...
</Machine.Control>

<Machine.Control
    onDidMount={({ transition }) => transition('POWER_ON')}
    onWillUnmount={({ transition }) => transition('POWER_OFF')}
>
    ...
</Machine.Control>
```

## Meta

### Inspiration

-   [`xstate`](https://github.com/davidkpiano/xstate), by David Khourshid
-   [`react-finite-machine`](https://github.com/derek-duncan/react-finite-machine), by Derek Duncan
-   [`react-automata`](https://github.com/MicheleBertoli/react-automata), by Michele Bertoli

### Maintainer

David Smith (@avaragado)

### Contribute

Bug reports, feature requests and PRs are gratefully received. [Add an issue](https://github.com/avaragado/xstateful-react/issues/new) or submit a PR.

Please note that this project is released with a [Contributor Code of Conduct](code-of-conduct.md). By participating in this project you agree to abide by its terms.

### Contributors

Thanks goes to these wonderful people ([emoji key](https://github.com/kentcdodds/all-contributors#emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore -->
| [<img src="https://avatars2.githubusercontent.com/u/886041?v=4" width="100px;"/><br /><sub><b>David Smith</b></sub>](http://avaragado.org)<br />[📖](https://github.com/avaragado/xstateful-react/commits?author=avaragado "Documentation") [💻](https://github.com/avaragado/xstateful-react/commits?author=avaragado "Code") [⚠️](https://github.com/avaragado/xstateful-react/commits?author=avaragado "Tests") | [<img src="https://avatars0.githubusercontent.com/u/4660659?v=4" width="100px;"/><br /><sub><b>ShMcK</b></sub>](http://medium.com/@shmck)<br />[📖](https://github.com/avaragado/xstateful-react/commits?author=ShMcK "Documentation") |
| :---: | :---: |

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/kentcdodds/all-contributors) specification. Contributions of any kind welcome!

### Developer notes

The `package.json` file contains all the usual scripts for linting, testing, building and releasing.

Buzzwords: prettier, eslint, flow, flow-typed, babel, jest, rollup, react.

#### Branches and merging

When merging to master **Squash and Merge**.

In the commit message, follow [conventional-changelog-standard conventions](https://github.com/bcoe/conventional-changelog-standard/blob/master/convention.md)

#### Releasing

When ready to release to npm:

1.  `git checkout master`
1.  `git pull origin master`
1.  `yarn release:dryrun`
1.  `yarn release`
1.  Engage pre-publication paranoia
1.  `git push --follow-tags origin master`
1.  `npm publish` - not yarn here as yarn doesn't seem to respect publishConfig

### Licence

[MIT](LICENSE.txt) © David Smith
