# React integration of JSS

[![Gitter](https://badges.gitter.im/JoinChat.svg)](https://gitter.im/cssinjs/lobby)

React-JSS provides components for [JSS](https://github.com/cssinjs/jss) as a layer of abstraction. JSS and [presets](https://github.com/cssinjs/jss-preset-default) are already built in! Try it out on [webpackbin](https://www.webpackbin.com/bins/-Kn90iijPuAJO48ItgF-).

The benefits are:

- Theming support out of the box.
- Lazy evaluation - sheet is created only when component will mount.
- Auto attach/detach - sheet will be rendered to the DOM when component is about to mount and will be removed when no element needs it.
- A Style Sheet gets shared between all elements.

## Table of Contents

* [Install](#install)
* [Usage](#usage)
  * [Basic](#basic)
  * [Theming](#theming)
  * [Server-side rendering](#server-side-rendering)
  * [Reuse same StyleSheet in different Components](#reuse-same-stylesheet-in-different-components)
  * [The classNames helper](#the-classnames-helper)
  * [The inner component](#the-inner-component)
  * [Custom setup](#custom-setup)
  * [Decorators](#decorators)
* [Contributing](#contributing)
* [License](#license)

## Install

```
npm install --save react-jss
```

## Usage

React-JSS wraps your component with an [higher-order component](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750). It injects `classes` and `sheet` into props, where `sheet` is a [JSS StyleSheet](https://github.com/cssinjs/jss) instance. It can act both as a simple wrapping function and as a [ES7 decorator](https://github.com/wycats/javascript-decorators)

JSS class names are scoped by default, you will need to reach into `props.classes` to get the generated class names.

### Example

Try it out on [webpackbin](https://www.webpackbin.com/bins/-Kn90iijPuAJO48ItgF-).

```javascript
import React from 'react'
import injectSheet from 'react-jss'

const styles = {
  button: {
    background: props => props.color
  },
  label: {
    fontWeight: 'bold'
  }
}

const Button = ({classes, children}) => (
  <button className={classes.button}>
    <span className={classes.label}>
      {children}
    </span>
  </button>
)

export default injectSheet(styles)(Button)
```

### Theming

The idea is that you define theme, wrap your application with `ThemeProvider` and pass the `theme` to `ThemeProvider`. ThemeProvider will pass it over `context` to your styles creator function and to your props. After that you may change your theme, and all your components will get new theme automatically.

Under the hood `react-jss` uses unified CSSinJS `theming` solution for React. You can find [detailed docs in its repo](https://github.com/iamstarkov/theming).

Using `ThemeProvider`:

* It has `theme` prop which should be an `object` or `function`:
  * If it is an `Object` and used in a root `ThemeProvider` then it's intact and being passed down the react tree.
  * If it is `Object` and used in a nested `ThemeProvider` then it's being merged with theme from a parent `ThemeProvider` and passed down the react tree.
  * If it is `Function` and used in a nested `ThemeProvider` then it's being applied to the theme from a parent `ThemeProvider`. If result is an `Object` it will be passed down the react tree, throws otherwise.
* `ThemeProvider` as every other component can render only single child, because it uses `React.Children.only` in render and throws otherwise.
* [Read more about `ThemeProvider` in `theming`'s documentation.](https://github.com/iamstarkov/theming#themeprovider)

```javascript
import React from 'react'
import {ThemeProvider} from 'react-jss'
import {Button} from './components'

const Button = ({classes, children}) => (
  <button className={classes.button}>
    <span className={classes.label}>
      {children}
    </span>
  </button>
)

const StyledButton = injectSheet((theme) => ({
  button: {
    background: theme.colorPrimary
  },
  label: {
    fontWeight: 'bold'
  }
}))(Button)

const theme = {
  colorPrimary: 'green'
}

const App = () => (
  <ThemeProvider theme={theme}>
    <StyledButton>I am a button with green background</StyledButton>
  </ThemeProvider>
)
```

In case you need to access the theme but not render any CSS, you can also use `withTheme`. It is a Higher-order Component factory which takes a `React.Component` and maps the theme object from context to props. [Read more about `withTheme` in `theming`'s documentation.](https://github.com/iamstarkov/theming#withthemecomponent)

```javascript
import React from 'react'
import injectSheet, {withTheme} from 'react-jss'

const Button = withTheme(({theme}) => (
  <button>I can access {theme.colorPrimary}</button>
))
```

### Server-side rendering

```javascript
import {renderToString} from 'react-dom/server'
import {JssProvider, SheetsRegistry} from 'react-jss'
import MyApp from './MyApp'

export default function render(req, res) {
  const sheets = new SheetsRegistry()

  const body = renderToString(
    <JssProvider registry={sheets}>
      <MyApp />
    </JssProvider>
  )

  // Any instances of `injectSheet` within `<MyApp />` will have gotten sheets
  // from `context` and added their Style Sheets to it by now.

  return res.send(renderToString(
    <html>
      <head>
        <style type="text/css">
          {sheets.toString()}
        </style>
      </head>
      <body>
        {body}
      </body>
    </html>
  ))
}
```

### Reuse same StyleSheet in different Components

Sometimes you may need to reuse the same Style Sheet in different components without generating new styles for each. You can pass a `StyleSheet` instance to the `injectSheet` function instead of `styles` object.

```javascript
import React, {Component} from 'react'
import injectSheet, {jss} from 'react-jss'

const sheet = jss.createStyleSheet({
  button: {
    color: 'red'
  }
})

@injectSheet(sheet)
class Button extends Component {
  render() {
    const {classes} = this.props
    return <button className={classes.button}></button>
  }
}
```

### The classNames helper

You can use [classNames](https://github.com/JedWatson/classnames) together with JSS same way you do it with global CSS.

```javascript
import classNames from 'classnames'

const Component = ({classes, children, isActive}) => (
  <div
    className={classNames({
      [classes.normal]: true,
      [classes.active]: isActive
    })}>
    {children}
  </div>
)
```

### The inner component

```es6
const InnerComponent = () => null
const StyledComponent = injectSheet(styles, InnerComponent)
console.log(StyledComponent.InnerComponent) // Prints out the inner component.
```

### Custom setup

If you want to specify a JSS version and plugins to use, you should create your [own Jss instance](https://github.com/cssinjs/jss/blob/master/docs/js-api.md#create-an-own-jss-instance), [setup plugins](https://github.com/cssinjs/jss/blob/master/docs/setup.md#setup-with-plugins) and pass it to `JssProvider`.

```javascript
import {create as createJss} from 'jss'
import {JssProvider} from 'react-jss'
import vendorPrefixer from 'jss-vendor-prefixer'

const jss = createJss()
jss.use(vendorPrefixer())

const Component = () => (
  <JssProvider jss={jss}>
    <App />
  </JssProvider>
)
```

You can also access the Jss instance being used by default.

```javascript
import {jss} from 'react-jss'
```

### Decorators

_Beware that [decorators are stage-2 proposal](https://tc39.github.io/proposal-decorators/), so there are [no guarantees that decorators will make its way into language specification](https://tc39.github.io/process-document/). Do not use it in production. Use it at your own risk and only if you know what you are doing._

You will need [babel-plugin-transform-decorators-legacy](https://github.com/loganfsmyth/babel-plugin-transform-decorators-legacy).

```javascript
import React, {Component} from 'react'
import injectSheet from 'react-jss'

const styles = {
  button: {
    backgroundColor: 'yellow'
  },
  label: {
    fontWeight: 'bold'
  }
}

@injectSheet(styles)
export default class Button extends Component {
  render() {
    const {classes, children} = this.props
    return (
      <button className={classes.button}>
        <span className={classes.label}>
          {children}
        </span>
      </button>
    )
  }
}
```

## Contributing

See our [contribution guidelines](./contributing.md).

## License

MIT
