---
title: vue3结合openapi的前端架构小记
date: 2023-11-8
categories: [vue]
tags: [前端架构,vue3,vite,pnpm,swagger,openapi,docker]
description: 使用vue3 + ts + pinia + openapi + vue-query + pnpm + docker搭建B端管理系统，h5前端架构的一些尝试和记录
---

## 1.引言
开发中，我们是否经常遇到以下痛点：
* 项目越大，启动和热更新越来越慢，启动都要花个3-5分钟以上
* 没有类型保障，接口返回的Object不拿到真实数据都不知道有哪些字段，接手别人js项目(无类型)很痛苦
* 需要手动写很多request函数去调用api，手动书写各种判断枚举值
* 缺乏代码格式化，代码错误检查，git commit规范
* npm包管理问题，比如：多版本的npm包冲突、npm包依赖嵌套、npm僵尸包、npm依赖包平铺到nodule_modules首层
* 手动变更接口的loading状态、手动管理modal的visible状态
* 很多热门的开源chatgpt产品: dify、fastgpt，他们都用很新的前端技术，但是仍然是大批量的手写request函数，手写各种枚举，以及interface，很痛苦

此前端架构优势以及展望如下：
* 支持自动根据openapi生成api request函数、类型、枚举等, [openapi数据格式参考](https://srv-demo-docker.onrender.com/openapi)
* 支持前端工程化，完美的ts开发体验，ts + eslint + tslint + prettier + commitlint + husky
* 支持前端容器化(需要安装docker环境)，跨环境运行
* 同步接口请求状态，实现自动loading
* 支持接口联动，方便跨父子组件刷新相关联的接口
* 支持容器化变量注入，无需前端配置文件写死，方便通过 k8s 动态注入
#### 基于以上痛点，我整合了一些开源技术搭了一套脚手架供自己使用，并分享给大家学习，如果对你有帮助请在[github](https://github.com/rookie-luochao/create-vite-app-cli)上面给我一个star🙏🙏🙏
#### 俗话说王婆卖瓜，自卖自夸，各位大佬轻喷！！！
#### openapi 规范文档对于前端来说，绝对是超级省事的，必须安排起来！！！
#### 很多细节没有在文章中提及！！！
## 2.脚手架核心技术
* 打包编译 - [vite](https://github.com/vitejs/vite)
* 包管理 - [pnpm](https://github.com/pnpm/pnpm)
* 编程语言 - [typescript](https://github.com/microsoft/TypeScript)
* 前端框架 - [vue3](https://github.com/vuejs/core)
* 路由 - [vue-router4](https://github.com/vuejs/router)
* UI组件库 - [element-plus](https://github.com/element-plus/element-plus)
* 全局数据共享 - [pinia](https://github.com/vuejs/pinia)
* 自动生成api - [openapi](https://github.com/chenshuai2144/openapi2typescript)
* 网络请求 - [axios](https://github.com/axios/axios)
* 数据请求利器 - [vue-query](https://github.com/TanStack/query/tree/main/packages/vue-query)
* 通用hook - [vueuse](https://github.com/vueuse/vueuse)
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
// HelloGet是一个基于axios的promise请求
export async function HelloGet(
  // 叠加生成的Param类型 (非body参数swagger默认没有生成对象)
  params: Api.HelloGetParams,
  options?: { [key: string]: any },
) {
  return request<Api.HelloResp>("/demo-docker/api/v1/hello", {
    method: "GET",
    params: {
      ...params,
    },
    ...(options || {}),
  });
}

// 自动调用接口获取数据
const name = ref("zhangsan");
const { data, isPending, refetch } = useQuery({
  queryKey: ["helloGet", name],
  queryFn: () => HelloGet({ name: name.value || "" }),
});

// HelloPost是一个基于axios的promise请求
export async function HelloPost(body: Api.HelloPostParam, options?: { [key: string]: any }) {
  return request<Api.HelloResp>("/demo-docker/api/v1/hello", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    data: body,
    ...(options || {}),
  });
}

// 提交编辑数据
const queryClient = useQueryClient();
const userStore = useUserStore();
const { mutate, isPending } = useMutation({
  mutationFn: HelloPost,
  onSuccess: (res) => {
    // 第一种刷新方式：修改store
    userStore.updateUserInfo({ name: res.data });
    // 第二种刷新方式：通过清除vue-query缓存key
    queryClient.invalidateQueries({ queryKey: ["helloGet"] });
  },
});

mutate({ name: "lisi" });
```

## 4.技术说明
* 自动生成api request函数(openapi):&ensp;后端接入apenapi后，前端可以根据openapi文件自动生成api request，后端通常使用[swagger](https://swagger.io)转换成openapi规范供前端使用
* 通用hook(vueuse):&ensp;一个hook工具库，就是hook增强，该库可以依据个人喜好选择是否使用
* 前端日志(sentry):&ensp;暂时未集成，需要进一步调研实用性和可用性

## 5.前端架构源码
[点此查看前端架构源码](https://github.com/rookie-luochao/create-vite-app-cli)