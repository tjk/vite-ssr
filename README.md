<p align="center">
  <img width="180" src="./logo.svg" alt="Vite SSR logo">
</p>

# Vite SSR

Simple yet powerlful Server Side Rendering for Vite 2 in Node.js.

- ⚡ Lightning Fast HMR (powered by Vite, even in SSR mode).
- 💁‍♂️ Consistent DX experience abstracting most of the SSR complexity.
- 🔍 Small library, unopinionated about your page routing and API logic.
- 🔥 Fast and SEO friendly thanks to SSR, with SPA takeover for snappy UX.
- 🧱 Compatible with Vite's plugin ecosystem such as file-based routing, PWA, etc.

Start a new SSR project right away using Vue, filesystem routes, page layouts, icons auto-import and more with [Vitesse SSR template](https://github.com/frandiox/vitesse-ssr-template). See [live demo](https://vitesse-ssr.vercel.app/).

Vite SSR can be deployed to any Node.js environment, including serverless platforms like Vercel or Netlify. It can create pages dynamically from a cloud function and cache the result at the edge network for subsequent requests, effectively behaving as statically generated pages with no cost.

See [Vitedge](https://github.com/frandiox/vitedge) for SSR in Cloudflare Workers.

## Installation

Create a normal [Vite](https://vitejs.dev/guide/) project for Vue or React.

```sh
yarn create @vitejs/app my-app --template [vue|vue-ts|react|react-ts]
```

Then, add `vite-ssr` with your package manager (direct dependency) and your framework router.

```sh
# For Vue
yarn add vite-ssr vue@3 vue-router@4 @vueuse/head

# For React
yarn add vite-ssr react@16 react-router-dom@5
```

## Usage

Add Vite SSR plugin to your Vite config file (see [`vite.config.js`](./examples/vue/vite.config.js) for a full example).

```js
// vite.config.js
import vue from '@vitejs/plugin-vue'
import viteSSR from 'vite-ssr/plugin.js'
// import reactRefresh from '@vitejs/plugin-react-refresh'

export default {
  plugins: [
    viteSSR(),
    vue(), // reactRefresh()
  ],
}
```

Then, simply import the main Vite SSR handler in your main entry file as follows. See full examples for [Vue](./examples/vue/src/main.js) and [React](./examples/react/src/main.jsx).

```js
import App from './App' // Vue or React main app
import routes from './routes'
import viteSSR from 'vite-ssr'

export default viteSSR(App, { routes }, (context) => {
  /* custom logic */
  /* const { app, router, initialState, ... } = context */
})
```

That's right, in Vite SSR **there's only 1 single entry file** by default 🎉. It will take care of providing your code with the right environment.

If you need conditional logic that should only run in either client or server, use Vite's `import.meta.env.SSR` boolean variable and the tree-shaking will do the rest.

<details><summary>Separate entry files option</summary>
<p>

Even though Vite SSR uses 1 single entry file by default, thus abstracting complexity from your app, you can still have separate entry files for client and server if you need more flexibility. This can happen when building a library on top of Vite SSR, for example.

Simply provide the entry file for the client in `index.html` (as you would normally do in an SPA) and pass the entry file for the server as a CLI flag: `vite-ssr [dev|build] --ssr <path/to/entry-server>`.

Then, import the main SSR handlers for the entry files from `vite-ssr/vue/entry-client` and `vite-ssr/vue/entry-server` instead. Use `vite-ssr/react/*` for React.

</p>
</details>

## SSR initial state and data fetching

The SSR initial state is the application data that is serialized as part of the server-rendered HTML for later hydration in the browser. This data is normally gathered using fetch or DB requests from your API code.

Vite SSR initial state consists of a plain JS object that is passed to your application and can be modified at will during SSR. This object will be serialized and later hydrated automatically in the browser, and passed to your app again so you can use it as a data source.

```js
export default viteSSR(App, { routes }, ({ initialState }) => {
  if (import.meta.env.SSR) {
    // Write in server
    initialState.myData = 'DB/API data'
  } else {
    // Read in browser
    console.log(initialState.myData) // => 'DB/API data'
  }

  // Provide the initial state to your stores, components, etc. as you prefer.
})
```

<details><summary>Initial state in Vue</summary>
<p>

Vue has multiple ways to provide the initial state to Vite SSR:

- Calling your API before entering a route (Router's `beforeEach` or `beforeEnter`) and populate `route.meta.state`. Vite SSR will get the first route's state and use it as the SSR initial state. See a full example [here](./examples/vue/src/main.js).

```js
routes.forEach((route) => {
  // Get meta.state as page props
  route.props = (route) => route.meta.state
})

export default viteSSR(App, { routes }, async ({ app }) => {
  router.beforEach((to, from, next) => {
    if (to.meta.state) {
      return next() // Already has state
    }

    const response = await fetch('my/api/data')

    // This will modify initialState
    to.meta.state = await response.json()

    next()
  })
})
```

- Using Vue's `serverPrefetch` to call your API from any component and save the result in the SSR initial state. See a full example [here](./examples/vue/src/pages/Homepage.vue).

```js
// Main
export default viteSSR(App, { routes }, ({ app, initialState }) => {
  // You can pass it to your state management, if you like that
  const store = createStore({ state: initialState /* ... */ })
  app.use(store)

  // Or provide it to child components
  app.provide('initial-state', initialState)
})

// Page Component
export default {
  async serverPrefetch() {
    await this.fetchMyData()
  },
  async beforeMount() {
    await this.fetchMyData()
  },
  methods: {
    async fetchMyData() {
      const data = await (await fetch('my/api/data')).json()
      const store = useStore()
      store.commit('myData', data)
    },
  },
}
```

</p>
</details>

<details><summary>Initial state in React</summary>
<p>

React and its router don't provide yet any mechanism to allow easy data prefetch in SSR (perhaps Suspense or Server Components will make it possible when they are stable).

Therefore, the only way to add initial state is modifying or returning it from the main Vite SSR handler. See [`main.jsx`](./examples/react/src/main.jsx) for an example.

```jsx
export default viteSSR(App, { routes }, ({ url, initialState }) => {
  if (!initialState.myData) {
    initialState.myData = await fetchMyData({ url })
  }

  myStore.dispatch({ type: 'ADD_INITIAL_STATE', initialState })
})
```

</p>
</details>

## Head tags and global attributes

<details><summary>Vue Head</summary>
<p>

Install [`@vueuse/head`](https://github.com/vueuse/head) as follows:

```js
import { createHead } from '@vueuse/head'

export default viteSSR(App, { routes }, ({ app }) => {
  const head = createHead()
  app.use(head)

  return { head }
})

// In your components:
// import { useHead } from '@vueuse/head'
// ... useHead({ ... })
```

</p>
</details>

<details><summary>React Helmet</summary>
<p>

Use [`react-helmet-async`](https://github.com/staylor/react-helmet-async) from your components (similar usage to `react-helmet`).

</p>
</details>

## Development

There are two ways to run the app locally for development:

- SPA mode: `vite dev` command runs Vite directly without any SSR.
- SSR mode: `vite-ssr dev` command spins up a local SSR server. It supports similar attributes to Vite CLI, e.g. `vite-ssr --port 1337 --open`.

SPA mode will be slightly faster but the SSR one will have closer behavior to a production environment.

## Production

Run `vite-ssr build` for buildling your app. This will create 2 builds (client and server) that you can import and use from your Node backend. See an Express.js example server [here](./examples/node-server/index.js), or a serverless function deployed to Vercel [here](https://github.com/frandiox/vitesse-ssr-template/blob/master/serverless/api/index.js).

## References

The following projects served as learning material to develop this tool:

- [@tbgse](https://github.com/tbgse)'s [vue3-vite-ssr-example](https://github.com/tbgse/vue3-vite-ssr-example/)

## Todos

- [x] TypeScript
- [x] Make `src/main.js` file name configurable
- [x] Support build options as CLI flags (`--ssr entry-file` supported)
- [x] Support React
- [x] SSR dev-server
- [x] Make SSR dev-server similar to Vite's dev-server (options, terminal output)
- [ ] Research if `vite-ssr` CLI logic can be moved to the plugin in Vite 2 to use `vite` command instead.
- [x] Docs
