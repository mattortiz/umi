---
translateHelp: true
---

# How to speed up compilation


If you encounter problems such as slow compilation, slow incremental compilation, memory explosion, OOM, etc., you can try the following methods.

## Configure `nodeModulesTransform` as `{ type:'none' }`

> Requires Umi 3.1.

Umi compiles files under node\_modules by default, which not only brings some benefits, but also adds extra compilation time. If you don't want the files under node\_modules to be compiled with babel, you can reduce the compilation time by 40% to 60% through the following configuration.

```js
export default {
  nodeModulesTransform: {
    type: 'none',
    exclude: [],
  },
}
```

## View package structure

When executing ʻumi dev` or ʻumi build`, increase the environment variable ʻANALYZE=1` to view the dependency ratio of the product.

<img src="https://img.alicdn.com/tfs/TB1P_iYDQL0gK0jSZFAXXcA9pXa-2432-1276.png" width="600" />

note:

* ʻUmi dev` can be modified and viewed in real time, but some development dependencies will be introduced, please ignore

## Configure externals

For some large-scale dependencies, such as chart libraries, antd, etc., you can try to import related umd files through the configuration of externals to reduce compilation consumption.

For example, react and react-dom:

```js
export default {
  // Configuration external
  externals: {
    'react': 'window.React',
    'react-dom': 'window.ReactDOM',
  },

  // Import scripts from external libraries
  // Differentiate development and production, use different products
  scripts: process.env.NODE_ENV === 'development' ? [
    'https://gw.alipayobjects.com/os/lib/react/16.13.1/umd/react.development.js',
    'https://gw.alipayobjects.com/os/lib/react-dom/16.13.1/umd/react-dom.development.js',
  ] : [
    'https://gw.alipayobjects.com/os/lib/react/16.13.1/umd/react.production.min.js',
    'https://gw.alipayobjects.com/os/lib/react-dom/16.13.1/umd/react-dom.production.min.js',
  ],
}
```
note:

1. If you want to support IE10 or below, external react also needs to introduce additional patches, such as `https://gw.alipayobjects.com/os/lib/alipay/react16-map-set-polyfill/1.0.2/dist/react16 -map-set-polyfill.min.js`
2. If antd is external, additional moment, react and react-dom need to be external at the same time, and introduced before antd

## Reduce patch size

Umi will include patches for the following browsers and their versions by default,

```
chrome: 49,
firefox: 64,
safari: 10,
edge: 13,
ios: 10,
```

Choosing the right browser version can reduce a lot of size. For example, if it is configured as follows, it is expected to reduce the size of 90K (not gzip after compression).

```js
export default {
  targets: {
    chrome: 79,
    firefox: false,
    safari: false,
    edge: false,
    ios: false,
  },
}
```

note:

* Setting the browser to false will not include his patch

## Adjust splitChunks strategy to reduce overall size

If dynamicImport is turned on, and the product is very large, and each export file contains the same dependencies, such as antd, you can try to adjust the extraction strategy of common dependencies through the splitChunks configuration.

such as:

```js
export default {
  chunks: ['vendors', 'umi'],
  chainWebpack: function (config, { webpack }) {
    config.merge({
      optimization: {
        minimize: true,
        splitChunks: {
          chunks: 'all',
          minSize: 30000,
          minChunks: 3,
          automaticNameDelimiter: '.',
          cacheGroups: {
            vendor: {
              name: 'vendors',
              test({ resource }) {
                return /[\\/]node_modules[\\/]/.test(resource);
              },
              priority: 10,
            },
          },
        },
      }
    });
  },
}
```

## Set the upper limit of memory through NODE\_OPTIONS

If OOM occurs, you can also try to solve it by increasing the upper memory limit. For example, `NODE_OPTIONS=--max_old_space_size=4096` is set to 4G.

## Adjust SourceMap generation method

If dev time is slow or incremental compilation is slow after modifying the code, you can try to disable SourceMap generation or adjust to a lower cost generation method.

```js
// disable sourcemap
export default {
  devtool: false,
};
```

or,

```js
// Use the lowest cost sourcemap generation method, the default is cheap-module-source-map
export default {
  devtool: 'eval',
};
```

## monaco-editor editor package

For editor packaging, the following configuration is recommended to avoid build errors:

```js
export default {
  chainWebpack: (config) => {
    config.plugin('monaco-editor-webpack-plugin').use(
      // More configuration https://github.com/Microsoft/monaco-editor-webpack-plugin#options
      new MonacoWebpackPlugin(),
    );
    config
    .plugin('d1-ignore')
      .use(
        // eslint-disable-next-line
        require('webpack/lib/IgnorePlugin'), [
          /^((fs)|(path)|(os)|(crypto)|(source-map-support))$/, /vs(\/|\\)language(\/|\\)typescript(\/|\\)lib/
        ]
      )
    .end()
    .plugin('d1-replace')
      .use(
        // eslint-disable-next-line
        require('webpack/lib/ContextReplacementPlugin'),
        [
          /monaco-editor(\\|\/)esm(\\|\/)vs(\\|\/)editor(\\|\/)common(\\|\/)services/,
          __dirname,
        ]
      )
    return config;
  }
}
```

## Replace compressor with esbuild

> Experimental function, there may be holes, but the effect is outstanding.

Taking a project that relies on full antd and bizcharts as an example, the test was carried out on the basis of disabling Babel cache and Terser cache, and the effect is shown in the figure:

![](https://gw.alipayobjects.com/zos/antfincdn/NRwBU0Kgui/7f24657c-ec26-420b-b956-14ae4e3de0a3.png)

Install dependencies first,

```bash
$ yarn add @umijs/plugin-esbuild
```

Then turn it on in the configuration,

```js
export default {
  esbuild: {},
}
```

## No compression

> Not recommended, use in emergency situations.

Compression time accounts for most of the slow compilation, so if you do not compress during compilation, you can save a lot of time and memory consumption, but the size will increase a lot. Compression can be skipped through the environment variable `COMPRESS=none`.

```bash
$ COMPRESS=none umi build
```
