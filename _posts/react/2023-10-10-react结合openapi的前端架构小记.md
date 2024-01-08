---
title: react结合openapi的前端架构小记
date: 2023-10-10
categories: [react]
tags: [前端架构,react,vite,pnpm,swagger,openapi,docker]
description: 使用react + ts + openapi + react-query + docker搭建B端管理系统，h5前端架构的一些尝试和记录
---

## 1.引言
开发中，我们是否经常遇到以下痛点：
* 项目越大，启动和热更新越来越慢，启动都要花个3-5分钟以上
* 没有类型，接口返回的Object不拿到真实数据都不知道有哪些字段
* 需要手动写很多request函数去调用api，手动书写各种判断枚举值
* 缺乏代码格式化，代码错误检查，git commit规范
* 难以维护的css代码和文件，js里面书写编写css时没有提示，js里面无法使用css高级用法
* 数据流要么太死板，对ts支持很差(dva)，要么太灵活(mobx)
* 重度依赖redux，需要写很多模板文件
* npm包管理问题，比如：多版本的npm包冲突、npm包依赖嵌套、npm僵尸包、npm依赖包平铺到nodule_modules首层
* 手动变更接口的loading状态、手动管理modal的visible状态
* 页面经常因为js错误导致白屏，体验很差
* 很多热门的开源chatgpt产品: dify、fastgpt，他们都用很新的前端技术，但是仍然是大批量的手写request函数，手写各种枚举，以及interface，很痛苦

此前端架构优势以及展望如下：
* 支持自动根据openapi生成api request函数、类型、枚举等, [openapi数据格式参考](https://srv-demo-docker.onrender.com/openapi)
* 支持前端工程化，完美的ts开发体验，ts + eslint + tslint + prettier + commitlint + husky
* 支持前端容器化(需要安装docker环境)，跨环境运行
* 同步接口请求状态，实现自动loading
* 支持接口联动，方便跨父子组件刷新相关联的接口
* 支持容器化变量注入，无需前端配置文件写死，方便通过 k8s 动态注入
* 后续支持更好用的modal、form
* 此脚手架最佳实战参考[rookie-luochao/react](https://github.com/rookie-luochao/react)，持续更新中

#### 基于以上痛点，我整合了一些开源技术搭了一套脚手架供自己使用，并分享给大家学习，如果对你有帮助请在[github](https://github.com/rookie-luochao/create-vite-app-cli)上面给我一个star🙏🙏🙏
#### 俗话说王婆卖瓜，自买自夸，各位大佬轻喷！！！
#### openapi 规范文档对于前端来说，绝对是超级省事的，必须安排起来！！！
#### 很多细节没有在文章中提及，关于为什么不用nextjs, 为什么用ts都会有自己的理解！！！

## 2.脚手架核心技术
* 打包编译 - [vite](https://github.com/vitejs/vite)
* 包管理 - [pnpm](https://github.com/pnpm/pnpm)
* 编程语言 - [typescript](https://github.com/microsoft/TypeScript)
* 前端框架 - [react](https://github.com/facebook/react)
* 路由 - [react-router](https://github.com/remix-run/react-router)
* UI组件库 - [antd](https://github.com/ant-design/ant-design)
* cssinjs(不考虑性能开销) - [emotion](https://github.com/emotion-js/emotion)
* 全局数据共享 - [zustand](https://github.com/pmndrs/zustand)
* 自动生成api - [openapi](https://github.com/chenshuai2144/openapi2typescript)
* 网络请求 - [axios](https://github.com/axios/axios)
* 数据请求利器 - [react-query](https://github.com/TanStack/query)
* 通用hook(可不用) - [ahooks](https://github.com/alibaba/hooks)
* 错误边界 - [react-error-boundary](https://github.com/bvaughn/react-error-boundary)
* 前端日志(暂未集成) - [sentry-javascript](https://github.com/getsentry/sentry-javascript)
* hack - [babel](https://github.com/babel/babel)
* 代码检查 - [eslint](https://github.com/eslint/eslint)
* ts代码检查插件 - [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint)
* 代码美化 - [prettier](https://github.com/prettier/prettier)
* git钩子 - [husky](https://github.com/typicode/husky)
* commit格式化 -[commitlint](https://github.com/conventional-changelog/commitlint)

## 2.自动基于后端openapi文件生成request函数
```js
// src/core/openapi/index.ts

// 示例代码
generateService({
  // openapi地址
  schemaPath: `${appConfig.baseURL}/${urlPath}`,
  // 文件生成目录
  serversPath: "./src",
  // 自定义网络请求函数路径
  requestImportStatement: `/// <reference types="./typings.d.ts" />\nimport request from "@request"`,
  // 代码组织命名空间, 例如：Api
  namespace: "Api",
});
```

## 3.调用接口示例

```js
// HelloGet是一个基于axios的promise请求（自动生成）
export async function HelloGet(
  // 叠加生成的Param类型 (非body参数swagger默认没有生成对象)
  params: Api.HelloGetParams,
  options?: { [key: string]: any },
) {
  return request<Api.HelloResp>('/gin-demo-server/api/v1/hello', {
    method: 'GET',
    params: {
      ...params,
    },
    ...(options || {}),
  });
}

// 自动调用接口获取数据
const { data, isLoading } = useQuery({
  queryKey: ["hello", name],
  queryFn: () => {
    return HelloGet({ name: name });
  },
});

// HelloPost是一个基于axios的promise请求（自动生成）
export async function HelloPost(body: Api.HelloPostParam, options?: { [key: string]: any }) {
  return request<Api.HelloResp>('/gin-demo-server/api/v1/hello', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    data: body,
    ...(options || {}),
  });
}

// 提交编辑数据
const { mutate, isLoading } = useMutation({
  mutationFn: HelloPost,
  onSuccess(data) {
    setName(data?.data || "");
  },
  onError() {
    // 清除queryKey为hello的接口数据缓存，自动重新获取接口数据
    queryClient.invalidateQueries({ queryKey: ['hello'] });
  }
})

mutate({ name: "lisi" });
```

## 4.技术说明
* 自动生成api request函数(openapi):&ensp;后端接入apenapi后，前端可以根据openapi文件自动生成api request，后端通常使用[swagger](https://swagger.io)转换成openapi规范供前端使用
* UI组件库(ant-design):&ensp;开箱即用，省心省力。没有选择[headless-ui](https://headlessui.com)，还没有看到成熟的方案([chakra-ui](https://chakra-ui.com)使用成本也很高)，封装成本高，会一直持续关注
* 通用hook(ahooks):&ensp;一个hook工具库，没有什么特别的亮点，就是hook增强，该库可以依据个人喜好选择是否使用
* 路由(react-router-dom):&ensp;自身默认支持错误边界功能，我觉得react-error-boundary更好用点，所以用hack绕过了react-router-dom的错误边界(ps: 暂时不支持参数禁用错误边界)，react-router-dom官方没有提供prop禁用默认的错误边界
* 前端日志(sentry):&ensp;暂时未集成，需要进一步调研实用性和可用性

## 5.前端架构源码
[点此查看前端架构源码](https://github.com/rookie-luochao/create-vite-app-cli)