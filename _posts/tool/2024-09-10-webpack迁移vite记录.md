---
title: webpack迁移vite记录
date: 2024-09-10
categories: [tool]
tags: [vite,react,工具]
description: React + Webpack迁移 Vite 的过程记录
---

## 1. 背景
由于老项目使用webpack打包，前端路由path估计100多个，冷启动时间太长，开发环境启动估计要4-5分钟，等待时间实在太长，再加上热更新也不是很好用，强行刷新页面等待的时间也感觉有点长，遂顺应迭代需求，升级一波vite以改善开发体验，由于迁移过程细节繁多，只拧部分重点展示具体代码


## 2. 迁移目标
由于迁移 vite 不可避免的要升级 node 版本(18+)，所以需要整体升级npm包版本，也要保证升级npm包版本后对代码进行对应的优化调整，整体目标如下：

- 升级 webpack => vite
- 升级 npm 包版本
- 切换包管理工具 npm => pnpm
- 升级路由到 react-router v6
- 升级 antd => antd v5
- 升级 moment => dayjs
- 升级 @ant-design/charts
- 升级 eslint,prettier,lint-staged,husky,stylelint,commitlint


## 3. 迁移方案
考虑到在原有仓库直接升级会有一定风险，不利于迁移对比，所以升级采用先搭建好vite开发环境，然后将代码拷贝进去新vite开发环境的方案
项目里面堆积了不少无用的npm包，所以为了依赖间接性，会进行依赖清理，减少不必要的依赖
由于eslint9 和 eslint8 整体区别很大，eslint8开发周期足够长，加上 eslint9 有一些或多或少的问题，遂考虑暂时升级到eslint8


## 4. 迁移Vite
迁移vite主要考虑以下功能

- 代理转发
- 开发环境默认使用https协议
- 支持qiankun(使用v3-beta会报错, 暂时使用v2.10.16)
- 继续使用 CROSS_ENV 定义环境
- 为打包后的文件提供传统浏览器兼容性支持
- 支持less
- 支持alias

```js
import path from 'path';
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import legacy from '@vitejs/plugin-legacy';
import basicSsl from '@vitejs/plugin-basic-ssl';
import { legacyQiankun } from 'vite-plugin-legacy-qiankun';

export default defineConfig({
    base: '/',
    server: {
        https: true,
        host: 'dev-new-eds.gaodun.com',
        port: 3000,
    },
    plugins: [
        react(),
        legacy({
            targets: ['defaults', 'not IE 11', '> 1%', 'not dead'],
        }),
        basicSsl(),
        legacyQiankun({
            name: 'eds2',
            devSandbox: true,
        }),
    ],
    define: { 'process.env.CROSS_ENV': JSON.stringify(process.env.CROSS_ENV) },
    resolve: {
        alias: [
            { find: '@', replacement: path.resolve(__dirname, 'src') },
            // 解决antd中less ~ 别名引入报错问题
            { find: /^~/, replacement: '' },
        ],
    },
    css: {
        preprocessorOptions: {
            less: {
                math: 'always',
                modifyVars: {
                    'primary-color': '#1DA57A',
                    'link-color': '#1DA57A',
                    'border-radius-base': '2px',
                },
                javascriptEnabled: true,
            },
        },
    },
    build: {
        rollupOptions: {
            output: {
                manualChunks: {
                    antd: ['antd'],
                },
            },
        },
    },
});
```


## 5. 升级React-Router v6
Router-Router5 到 Router-Router6的变动还是挺多的，具体到我们项目的变更如下：

- Switch 组件 => Routes 组件
- component 和 render 属性替换为 element
- 底座模式改变，v5匹配 `/` 对应的组件，v6 由于默认是精确匹配，所以需要配置为 `/*` 对应底座组件
- 路由对应的懒加载组件需要使用 Suspense 组件进行包裹
- 嵌套路由行为改变
- v6中不再需要使用 exact 显式的指定精确匹配，默认就是精确匹配
- useHistory => useNavigate
- Redirect => Navigate
- withRouter => useLocation,useNavigate,useParams

总体入口调整如下：

```js
// React-Router 5
const Container = () => {
  return (
    <BrowserRouter basename={window.__POWERED_BY_QIANKUN__ ? '/system/eds2' : '/'}>
      <Switch>
          <Route path={'/'} component={Index} />
          <Route path={'/login'} component={Login} />
          {reportRouters?.map(item => {
              return <Route key={item.path} path={item.path} component={item.component} />;
          })}
      </Switch>
    </BrowserRouter>
  )
}

// React-Router 6
const Container = () => {
  const baseRoutes = [
    {
        path: '/*',
        element: React.lazy(() => import('@/containers/Index')),
    },
  ];

  return (
    <BrowserRouter basename={window.__POWERED_BY_QIANKUN__ ? '/system/eds2' : '/'}>
      <Routes>
          {baseRoutes.map(item => (
              <Route
                  key={item.path}
                  path={item.path}
                  element={
                      <React.Suspense fallback={<Spin spinning={true} />}>
                          <item.element key={item.path} />
                      </React.Suspense>
                  }
              />
          ))}
          <Route path="/login" element={<Login />} />
          {reportRouters?.map(item => {
              return <Route key={item.path} path={item.path} element={item.element} />;
          })}
      </Routes>
  </BrowserRouter>
  )
}
```

嵌套路由行为改变

```js
const Parent = () => (
  <div>
    <h1>Parent</h1>
    <Outlet />
  </div>
);

// React-Router 5
const Container = () => {
  return (
    <Switch>
      <Route path="/parent" component={Parent} />
      <Route path="/parent/child" component={Child} />
    </Switch>
  )
}

// React-Router 6
const Container = () => {
  return (
    <Route path="parent" element={<Parent />}>
      <Route path="child" element={<Child />} />
    </Route>
  )
}
```


## 6. 升级Antd
Antd v4 到 Antd v5 的重量级改变是主题系统，v5 选择了 CSS-in-JS 样式方案，Token 系统取代 LESS 变量

我们项目的Antd细节调整如下(部分省略)：

- Modal prop visible => open, footerStyle => styles
- Drawer prop visible => open
- Select prop dropdownMatchSelectWidth => popupMatchSelectWidth, bordered => variant
- Tabs TabPane => Tabs items 配置
- Dropdown prop overlay => menu
- RangePicker 调整
- form.validateFields 与 form.setFieldsValue 配合使用的行为调整，需要调整 form.validateFields 的时机


## 7. 升级husky

- husky install => husky,husky init
- 无需使用 shell 脚本执行 husky.sh 文件

```bash
# husky v8
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx commitlint --edit $1 --config .commitlintrc.js

# husky v9
npx commitlint --edit $1 --config .commitlintrc.js
```

## 8. 结语
* 介绍了 webpack 升级 vite 的迁移过程，整体迁移顺畅
* 在 vite 中使用 qiankun v3-beta 暂时会有问题，遂使用 qiankun v2.10.16
* antd form.validateFields 和 form.setFieldsValue 配合使用可能会遇到问题，需要调整事件循环任务优先级
* 遇到某些难解决的问题，可能需要避免使用 React18 严格模式