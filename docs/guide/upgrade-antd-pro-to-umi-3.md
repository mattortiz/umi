---
nav:
  title: Guide
translateHelp: true
---

# Upgrade Ant Design Pro to Umi 3


Migrate to Umi 3 in three steps, less than 10 minutes to complete the migration：

1. **Dependency Processing**
2. **Configuration layer migration**
3. **Code layer modification**

### Dependency processing

The `package.json` of the project needs to upgrade umi and replace the corresponding umi plugin.

```diff
{
  "dependencies": {
-   "dva": "^2.6.0-beta.16",
  },
  "devDependencies": {
-   "umi": "^2.13.0",
-   "umi-types": "^0.5.9"
-   "umi-plugin-react": "^1.14.10",
-   "umi-plugin-ga": "^1.1.3",
-   "umi-plugin-pro": "^1.0.2",
-   "umi-plugin-antd-icon-config": "^1.0.2",
-   "umi-plugin-antd-theme": "^1.0.1",
-   "umi-plugin-pro-block": "^1.3.2",
+   "umi": "^3.0.0",
+   "@umijs/preset-react": "^1.2.2"
  }
}
```

Execute `npm install` to reinstall dependencies.

### Configuration layer migration

According to [Umi 3 Configuration](../config), there are **modified configuration items** as follows: `config/config.ts`:

```typescript
import { defineConfig, utils } from 'umi';

const { winPath } = utils;

export default defineConfig({
  // Automatically mount the umi plugin through package.json, no need to mount again
  // plugins: [],
  
  antd: {},
  dva: {
    hmr: true,
  },
  locale: {
    default: 'zh-CN',
    baseNavigator: true,
  },
  dynamicImport: {
    // 无需 level, webpackChunkName 配置
    // loadingComponent: './components/PageLoading/index'
    loading: '@/components/PageLoading/index',
  },
  // 暂时关闭
  pwa: false,
  lessLoader: { javascriptEnabled: true },
  cssLoader: {
    // 这里的 modules 可以接受 getLocalIdent
    modules: {
      getLocalIdent:(
        context: {
          resourcePath: string;
        },
        _: string,
        localName: string,
      ) => {
        if (
          context.resourcePath.includes('node_modules') ||
          context.resourcePath.includes('ant.design.pro.less') ||
          context.resourcePath.includes('global.less')
        ) {
          return localName;
        }
        const match = context.resourcePath.match(/src(.*)/);
        if (match && match[1]) {
          const antdProPath = match[1].replace('.less', '');
          const arr = winPath(antdProPath)
            .split('/')
            .map((a: string) => a.replace(/([A-Z])/g, '-$1'))
            .map((a: string) => a.toLowerCase());
          return `antd-pro${arr.join('-')}-${localName}`.replace(/--/g, '-');
        }
        return localName;
      },
    }
  }
})
```

### 代码层修改

Umi 3 增加 `import from umi`，常用的模块、工具可直接从 `umi` 中导入：

```diff
- import Link from 'umi/link';
- import { connect } from 'dva';
- import { getLocale, setLocale, formatMessage } from 'umi-plugin-react/locale';
+ import {
+   Link,
+   connect,
+   getLocale,
+   setLocale,
+   formatMessage,
+ } from 'umi';
```

**注意：**不建议直接使用 formatMessage，推荐大家使用 [useIntl](/zh-CN/plugins/plugin-locale#useintl) 或者 [injectIntl](https://github.com/formatjs/formatjs/blob/main/website/docs/react-intl/api.md#injectintl-hoc)，可以实现同样的功能。

路由跳转使用 `history`：

```diff
- import { router } from 'umi';
+ import { history } from 'umi';

- router.push()
+ history.push()
```

第三步完成后，执行下 `npm run start`，访问 [http://localhost:8000](http://localhost:8000)，能访问则表示迁移完成。

![](https://gw.alipayobjects.com/zos/antfincdn/MysqNKCYyc/ae1d7e2a-3b6e-49d8-8c0a-c306840932f6.png)

> 更多迁移细节见 [PR](https://github.com/ant-design/ant-design-pro/pull/6039)。
