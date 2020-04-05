# Hot Module Replacement

Hot module replacement (also known as Hot Reloading)

## How HMR works internally

All the source code is bundled into a single `modules.js` file which looks like this:

```js
// modules.js
export const modules = {
  'src/index.js': [
    // module function for `src/index.js`
    (exports, require, module) => {
      const _add = require('./add')
      console.log(_add.add(1, 2))
    },
    // dependencyMap for `src/index.js`
    {
      './add': 'src/add.js',
    },
  ],
  'src/add.js': [
    // module function for `src/add.js`
    (exports, require, module) => {
      exports.add = (a, b) => a + b
    },
    // dependencyMap for `src/add.js`
    {},
  ],
}
```

Having the source in this format is useful because now modules can be replaced like this

```js
function hmrUpdate({ id, content }) {
  modules[id][0] = (exports, require) => eval(content)
}

// updates the content of `src/index.js`
hmrUpdate({
  id: 'src/index.js',
  content: '\nconst _add = require("./add")\nconsole.log(_add.add(2,3))',
})
```

## Using Hot Module Replacement

### Accept other files

When `fileA` is accepts `fileB` and `fileB` changes, the accept callback function is invoked. It is responsible for updating the `fileA`. `fileA` will not be re-executed.

Example:

```js
// state.js
export const state = { count: 0 }

// render.js
export const render = state => {
  document.body.innerHtml = `count is ${state.count}`
}

// index.js
import { state } from './state'
import { render } from './render'

const component = {
  state,
  render,
}

setInterval(() => {
  state.count++
  component.render(state)
}, 1000)

if (module.hot) {
  module.hot.accept('./state', () => {
    component.state = require('./state').state
  })
  module.hot.accept('./render', () => {
    component.render = require('./render').render
  })
}
```

When we change `state.js`, we require the new state object and assign it to `component.state`.

When we change `render.js`, we require the new render function and assign it to `component.render`.

In both cases `index.js` itself will not be re-executed. But because `component.state`/`component.render` (which we just updated) is used inside the interval it will use the updated state object/the updated render function there.

Updating the render function preserves the state. If the interval has already run 5 times, `state.count` will be `6` and after changing the render function, `state.count` will still be `6`.

### Accept itself

When a file is self-accepting and that file changes, first the content of the module will be updated and then the module will be re-executed.

Example:

```js
// index.js
import { add } from './add.js'

console.log(add(1, 2))

if (module.hot) {
  module.hot.accept('.', () => {}) // self-accept hmr
}
```

If we replace `console.log(add(1, 2))` with `console.log(add(3, 4))`, `index.js` will be updated and then required and run from top to bottom.

<!-- Some transform functions automatically insert hmr code. -->
