---
id: rtk-query
title: RTK Query
sidebar_label: RTK Query Quick Start
hide_title: true
---

# RTK Query Quick Start

:::tip What You'll Learn

- How to set up and use Redux Toolkit's "RTK Query" data fetching functionality

:::

:::info Prerequisites

- Understanding of [Redux terms and concepts](https://redux.js.org/tutorials/fundamentals/part-2-concepts-data-flow)

:::

## Introduction

Welcome to the Redux Toolkit Query tutorial! **This tutorial will briefly introduce you to Redux Toolkit's "RTK Query" data fetching capability and teach you how to start using it correctly**.

RTK Query is an advanced data fetching and caching tool, designed to simplify common cases for loading data in a web application. RTK Query itself is built on top of the Redux Toolkit core, and leverages RTK's APIs like [`createSlice`](../api/createSlice.mdx) and [`createAsyncThunk`](../api/createAsyncThunk.mdx) to implement its capabilities.

RTK Query is included in the `@reduxjs/toolkit` package as an additional addon. You are not required to use the RTK Query APIs when you use Redux Toolkit, but we think many users will benefit from RTK Query's data fetching and caching in their apps.

### How to Read This Tutorial

For this tutorial, we assume that you're using Redux Toolkit with React, but you can also use it with other UI layers as well. The examples are based on [a typical Create-React-App folder structure](https://create-react-app.dev/docs/folder-structure) where all the application code is in a `src`, but the patterns can be adapted to whatever project or folder setup you're using.

## Setting up your store and API service

To see how RTK Query works, let's walk through a basic usage example. For this example, we'll assume you're using React and want to make use of RTK Query's auto-generated React hooks.

### Create an API service

First, we'll create a service definition that queries the publicly available [PokeAPI](https://pokeapi.co/).

```ts title="src/services/pokemon.ts"
// Need to use the React-specific entry point to import createApi
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/react'

// highlight-start
// Define a service using a base URL and expected endpoints
export const pokemonApi = createApi({
  reducerPath: 'pokemonApi',
  baseQuery: fetchBaseQuery({ baseUrl: 'https://pokeapi.co/api/v2/' }),
  endpoints: (builder) => ({
    getPokemonByName: builder.query({
      query: (name: string) => `pokemon/${name}`,
    }),
  }),
})
//highlight-end

// highlight-start
// Export hooks for usage in functional components, which are
// auto-generated based on the defined endpoints
export const { useGetPokemonByNameQuery } = pokemonApi
// highlight-end
```

With RTK Query, you usually define your entire API definition in one place. This is most likely different from what you see with other libraries such as `swr` or `react-query`, and there are several reasons for that. Our perspective is that it's _much_ easier to keep track of how requests, cache invalidation, and general app configuration behave when they're all in one central location in comparison to having X number of custom hooks in different files throughout your application.

### Add the service to your store

An RTK service generates a "slice reducer" that should be included in the Redux root reducer, and a custom middleware that handles the data fetching. Both need to be added to the Redux store.

```ts title="src/store.ts"
import { configureStore } from '@reduxjs/toolkit'
// Or from '@reduxjs/toolkit/query/react'
import { setupListeners } from '@reduxjs/toolkit/query'
import { pokemonApi } from './services/pokemon'

export const store = configureStore({
  reducer: {
    // highlight-start
    // Add the generated reducer as a specific top-level slice
    [pokemonApi.reducerPath]: pokemonApi.reducer,
    // highlight-end
  },
  // highlight-start
  // Adding the api middleware enables caching, invalidation, polling,
  // and other useful features of `rtk-query`.
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(pokemonApi.middleware),
  // highlight-end
})

// optional, but required for refetchOnFocus/refetchOnReconnect behaviors
// see `setupListeners` docs - takes an optional callback as the 2nd arg for customization
setupListeners(store.dispatch)
```

### Wrap your application with the `Provider`

If you haven't already done so, follow the standard pattern for providing the Redux store to the rest of your React application component tree:

```ts title="src/index.tsx"
import * as React from 'react'
import { render } from 'react-dom'
import { Provider } from 'react-redux'

import App from './App'
import { store } from './store'

const rootElement = document.getElementById('root')
render(
  <Provider store={store}>
    <App />
  </Provider>,
  rootElement
)
```

## Use the query in a component

Once a service has been defined, you can import the hooks to make a request.

```ts title="src/App.tsx"
import * as React from 'react'
// highlight-next-line
import { useGetPokemonByNameQuery } from './services/pokemon'

export default function App() {
  // highlight-start
  // Using a query hook automatically fetches data and returns query values
  const { data, error, isLoading } = useGetPokemonByNameQuery('bulbasaur')
  // highlight-end

  return (
    <div className="App">
      {error ? (
        <>Oh no, there was an error</>
      ) : isLoading ? (
        <>Loading...</>
      ) : data ? (
        <>
          <h3>{data.species.name}</h3>
          <img src={data.sprites.front_shiny} alt={data.species.name} />
        </>
      ) : null}
    </div>
  )
}
```

When making a request, you're able to track the state in several ways. You can always check `data`, `status`, and `error` to determine the right UI to render. In addition, `useQuery` also provides utility booleans like `isLoading`, `isFetching`, `isSuccess`, and `isError` for the latest request.

#### Basic Example

<iframe
  src="https://codesandbox.io/embed/getting-started-basic-17n8h?fontsize=12&hidenavigation=1&theme=dark"
  style={{ width: '100%', height: '500px', border: 0, borderRadius: '4px', overflow: 'hidden' }}
  title="rtk-query-getting-started-basic"
  allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
  sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
></iframe>

Okay, that's interesting... but what if you wanted to show multiple pokemon at the same time? What happens if multiple components load the same pokemon?

#### Advanced example

RTK Query ensures that any component that subscribes to the same query will always use the same data. RTK Query automatically de-dupes requests so you don't have to worry about checking in-flight requests and performance optimizations on your end. Let's evaluate the sandbox below - make sure to check the Network panel in your browser's dev tools. You will see 3 requests, even though there are 4 subscribed components - `bulbasaur` only makes one request, and the loading state is synchronized between the two components. For fun, try changing the value of the dropdown from `Off` to `1s` to see this behavior continue when a query is re-ran.

<iframe
  src="https://codesandbox.io/embed/getting-started-advanced-8tx2b?file=/src/App.tsx?fontsize=12&hidenavigation=1&theme=dark"
  style={{ width: '100%', height: '600px', border: 0, borderRadius: '4px', overflow: 'hidden' }}
  title="rtk-query-getting-started-advanced"
  allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
  sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
></iframe>