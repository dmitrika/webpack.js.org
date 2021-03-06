---
title: SplitChunksPlugin
contributors:
  - sokra
  - jeremenichelli
  - chrisdothtml
  - EugeneHlushko
  - byzyk
  - madhavarshney
related:
  - title: webpack's automatic deduplication algorithm example
    url: https://github.com/webpack/webpack/blob/master/examples/many-pages/README.md
  - title: "webpack 4: Code Splitting, chunk graph and the splitChunks optimization"
    url: https://medium.com/webpack/webpack-4-code-splitting-chunk-graph-and-the-splitchunks-optimization-be739a861366
---

Originally, chunks (and modules imported inside them) were connected by a parent-child relationship in the internal webpack graph. The `CommonsChunkPlugin` was used to avoid duplicated dependencies across them, but further optimizations were not possible

Since webpack v4, the `CommonsChunkPlugin` was removed in favor of `optimization.splitChunks`.


## Defaults

Out of the box `SplitChunksPlugin` should work great for most users.

By default it only affects on-demand chunks because changing initial chunks would affect the script tags the HTML file should include to run the project.

webpack will automatically split chunks based on these conditions:

* New chunk can be shared OR modules are from the `node_modules` folder
* New chunk would be bigger than 30kb (before min+gz)
* Maximum number of parallel requests when loading chunks on demand would be lower or equal to 5
* Maximum number of parallel requests at initial page load would be lower or equal to 3

When trying to fulfill the last two conditions, bigger chunks are preferred.

## Configuration

For developers that want to have more control over this functionality, webpack provides a set of options to better fit your needs. If you're changing the configuration, it's a good idea to measure the impact of your changes to ensure there's a real benefit.

W> Default configuration was chosen to fit web performance best practices but the optimum strategy for your project might defer depending on the nature of it.

## `optimization.splitChunks`

This configuration object represents the default behavior of the `SplitChunksPlugin`.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async',
      minSize: 30000,
      minChunks: 1,
      maxAsyncRequests: 5,
      maxInitialRequests: 3,
      automaticNameDelimiter: '~',
      name: true,
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

### `splitChunks.automaticNameDelimiter`

`string`

By default webpack will generate names using origin and name of the chunk (e.g. `vendors~main.js`). This option lets you specify the delimiter to use for the generated names.


### `splitChunks.chunks`

`function` `string`

This indicates which chunks will be selected for optimization. If a string is provided, possible values are `all`, `async`, and `initial`. Providing `all` can be particularly powerful because it means that chunks can be shared even between async and non-async chunks.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // include all types of chunks
      chunks: 'all'
    }
  }
};
```

Alternatively, you can provide a function for more control. The return value will indicate whether to include each chunk.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks (chunk) {
        // exclude `my-excluded-chunk`
        return chunk.name !== 'my-excluded-chunk';
      }
    }
  }
};
```

T> You can combine this configuration with the [HtmlWebpackPlugin](/plugins/html-webpack-plugin/). It will inject all the generated vendor chunks for you.

### `splitChunks.maxAsyncRequests`

`number`

Maximum number of parallel requests when on-demand loading.

### `splitChunks.maxInitialRequests`

`number`

Maximum number of parallel requests at an entry point.

### `splitChunks.minChunks`

`number`

Minimum number of chunks that must share a module before splitting.

### `splitChunks.minSize`

`number`

Minimum size for a chunk to be generated.

### `splitChunks.name`

`boolean: true` `function` `string`

The name of the split chunk. Providing `true` will automatically generate a name based on chunks and cache group key. Providing a string or function will allow you to use a custom name. If the name matches an entry point name, the entry point will be removed.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      name (module) {
        // generate a chunk name...
        return; //...
      }
    }
  }
};
```

W> When assigning equal names to different split chunks, all vendor modules are placed into a single shared chunk, though it's not recommend since it can result in more code downloaded.

### `splitChunks.cacheGroups`

Cache groups can inherit and/or override any options from `splitChunks.*`; but `test`, `priority` and `reuseExistingChunk` can only be configured on cache group level. To disable any of the default cache groups, set them to `false`.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        default: false
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.priority`

`number`

A module can belong to multiple cache groups. The optimization will prefer the cache group with a higher `priority`. The default groups have a negative priority to allow custom groups to take higher priority (default value is `0` for custom groups).

#### `splitChunks.cacheGroups.reuseExistingChunk`

`boolean`

If the current chunk contains modules already split out from the main bundle, it will be reused instead of a new one being generated. This can impact the resulting file name of the chunk.

#### `splitChunks.cacheGroups.test`

`function` `RegExp` `string`

Controls which modules are selected by this cache group. Omitting it selects all modules. It can match the absolute module resource path or chunk names. When a chunk name is matched, all modules in the chunk are selected.

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      test (chunks) {
        //...
        return true;
      }
    }
  }
};
```

## Examples

### Defaults: Example 1

```js
// index.js

import('./a'); // dynamic import
```

```js
// a.js
import 'react';

//...
```

**Result:** A separate chunk would be created containing `react`. At the import call this chunk is loaded in parallel to the original chunk containing `./a`.

Why:

* Condition 1: The chunk contains modules from `node_modules`
* Condition 2: `react` is bigger than 30kb
* Condition 3: Number of parallel requests at the import call is 2
* Condition 4: Doesn't affect request at initial page load

What's the reasoning behind this? `react` probably won't change as often as your application code. By moving it into a separate chunk this chunk can be cached separately from your app code (assuming you are using chunkhash, records, Cache-Control or other long term cache approach).

### Defaults: Example 2

```js
// entry.js

// dynamic imports
import('./a');
import('./b');
```

```js
// a.js
import './helpers'; // helpers is 40kb in size

//...
```

```js
// b.js
import './helpers';
import './more-helpers'; // more-helpers is also 40kb in size

//...
```

**Result:** A separate chunk would be created containing `./helpers` and all dependencies of it. At the import calls this chunk is loaded in parallel to the original chunks.

Why:

* Condition 1: The chunk is shared between both import calls
* Condition 2: `helpers` is bigger than 30kb
* Condition 3: Number of parallel requests at the import calls is 2
* Condition 4: Doesn't affect request at initial page load

Putting the content of `helpers` into each chunk will result into its code being downloaded twice. By using a separate chunk this will only happen once. We pay the cost of an additional request, which could be considered a tradeoff. That's why there is a minimum size of 30kb.

### Split Chunks: Example 1

Create a `commons` chunk, which includes all code shared between entry points.

__webpack.config.js__


```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          name: 'commons',
          chunks: 'initial',
          minChunks: 2
        }
      }
    }
  }
};
```

W> This configuration can enlarge your initial bundles, it is recommended to use dynamic imports when a module is not immediately needed.

### Split Chunks: Example 2

Create a `vendors` chunk, which includes all code from `node_modules` in the whole application.

__webpack.config.js__


```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        commons: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  }
};
```

W> This might result in a large chunk containing all external packages. It is recommended to only include your core frameworks and utilities and dynamically load the rest of the dependencies.
