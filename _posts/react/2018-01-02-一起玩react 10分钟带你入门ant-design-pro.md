---
title: 一起玩react 10分钟带你入门ant-design-pro
date: 2018-01-02
categories: [react]
tags: [react]
description: 前端ant-design-pro框架的一些尝试和记录
---

## 1.前言
这篇文章主要根据自己最近一个月从学习react到最近实际使用ant-design-pro，谈一谈自己的使用心得，个人见解有误的地方望大家指正！

## 2.为什么要选择ant-design-pro？
其实我来目前公司之前，公司前端技术栈是vue+vuex+elementui+axios，但是奈何公司前端利用vue做出来东西表现确实一般，更重要的是代码有点乱，用我自己的话说就是野路子太多（当然野路子多会很方便），所以发挥空间很大，所以代码会因人而异变得不规范起来，增加后来的人维护成本，基于我之前angular1,angular2-4（angular5已经发布，和angular4一样主要优化编译速度和编译后的文件大小）的使用经验，综合考虑一下我还是支持使用react+redux的前端技术栈的，angular2的问题在于我当时使用angular-cli进行开发的时候，mock机制不够友好，以至于我对前端开发mock的认知是前端mock只能模拟get请求（也就是从js或者json文件里面读数据），直到我接触了ant-design-pro集成的mock机制，发现在前端也是完全可以利用内存实现post请求的，然后就是碰巧蚂蚁技术体验部基于react+redux的架构出了一套最佳实战--dva,然后ant-design-pro就是基于dva-cli进行改进的，dva让单向数据流变得非常清晰，用model驱动view的思想贯穿始终，这就让开发规范很多（当然也就限制了你的开发野路子）

## 3.技术组成？
关于[ant-design-pro](https://pro.ant.design/index-cn)的基本的学习（如何安装启动等等）请直接参照官网，技术组成主要是react+redux+dva+antd+fetch+roadhog，dva把react+redux放了出来，使用者可以自由选择react和redux的版本，dva虽然在源码包index.js里面导出了fetch，但是你不想使用fetch库，想换成其他库也是可以的（笔者就是换用了axios，因为笔者觉得axios后期扩展性更好），roadhog主要是基于webpack实现的封装

## 4.推荐开发套路？
使用ant-design-pro，建议分layout就行开发，分layout进行开发有很多好处，比如可以复用公共的界面和状态，处理权限路由会更加方便，划分路由会思路更加清晰等等。首先开始做一个项目或者阅读一个项目之前你首先应该阅读的代码应该是路由和项目启动文件（纯属我的认知），然后根据路由再去寻找对应的模块，贴第一个ant-design-pro项目的路由配置：
```js
import dynamic from 'dva/dynamic'; // 异步路由

// wrapper of dynamic
const dynamicWrapper = (app, models, component) => dynamic({
  app,
  models: () => models.map(m => import(`../models/${m}.js`)),
  component,
});

export const getNavData = app => [
  {
    component: dynamicWrapper(app, ['app', 'home', 'taxConsole', 'login'], () => import('../layouts/BasicLayout')),
    layout: 'BasicLayout',
    name: '首页',
    path: '/',
    children: [
      {
        name: '客户列表',
        icon: '',
        path: '/',
        component: dynamicWrapper(app, [], () => import('../routes/Home')),
      },
      {
        name: '客户详情',
        path: 'customer-detail/:id',
        component: dynamicWrapper(app, ['customerDetail'], () => import('../routes/CustomerDetail')),
      },
      {
        name: '展开详情',
        component: dynamicWrapper(app, ['dataDialog'], () => import('../routes/DataDialog')),
        path: 'dataDialog/:id',
      },
    ],
  },
  {
    component: dynamicWrapper(app, ['app'], () => import('../layouts/EmptyLayout')),
    layout: 'EmptyLayout',
    name: '登录',
    path: '',
    children: [{
      name: '用户',
      icon: 'user',
      path: 'user',
      children: [{
        name: '欢迎页',
        component: dynamicWrapper(app, ['login'], () => import('../routes/Welcome')),
        path: 'welcome',
      }, {
        name: '注册',
        component: dynamicWrapper(app, ['register'], () => import('../routes/Register')),
        path: 'register',
      }, {
        component: dynamicWrapper(app, ['login'], () => import('../routes/Login')),
        path: 'login',
        name: '登录',
      }, {
        name: '个人中心',
        path: 'pensonal-center',
        component: dynamicWrapper(app, ['home', 'login', 'center'], () => import('../routes/PersonalCenter')),
      }, {
        name: '找回密码',
        path: 'reset-password',
        component: dynamicWrapper(app, ['resetPassword'], () => import('../routes/ResetPassword')),
      }],
    }],
  },
];
```

业务代码一般放到src/routes文件夹里面，业务代码一般都是会连接models里面的state，也就是有状态的组件，业务组件的入口一般会和路由配置文件的component属性进行对应，纯组件一般放到src/components文件夹里面。纯组件一般都是可以复用的组件，和antd库里的组件一样，不应该连接models里面的state，可以被复用。其他的如http请求放在src/services文件夹，工具函数放在src/utils文件夹等等。所以大体上的思路是：
1. 在路由配置文件（src/common文件夹下）不同layout下新增路由。
2. 在src/routes文件夹下面新增路由组件（容器组件），路由组件里面编写模块主界面，编写事件处理代码
3. 在容器组件触发model层里面action修改state。
4. state改变会触发每一个连接改state的容器组件进行重新渲染。
所以说推荐处理数据的逻辑都放在model层。当然容器组件可以有自己的临时state的，这些state的修改可以在容器组件`this.setState({stateKey: stateValue})`，容器组件自己的state和models文件夹里面的state区别在于：容器组件的state是有生命周期的，容器组件被销毁，state也就不存在了，而models里面的state是在整个项目运行期间一直存在于内存当中的。

## 5.你应该要弄清楚的几个问题？
1. react的生命周期？
2. 纯组件和非纯组件的区别？
3. React.pureComponent和React.component的区别？
4. react定义组件的方式以及它们之间的区别？
5. react为什么要手动绑定this？
6. react触发渲染的方式和区别？
7. 如何监听数据源的变化？
8. 如何同时进行多异步请求？
9. 基于fetch如何进行超时处理？
10. 如何进行性能优化？
11. 如何加快首屏渲染速度？
弄清楚这些问题能帮你避免很多坑，当然希望每个人都带着这些问题去学习，因为自己寻找怎么解决问题的过程也是学习的过程

## 6.具体开发过程？
下面我以具体如何初始化一个表格和修改表格内容为栗子讲解一下业务模块的具体实现过程
首先看下表格渲染结果
![](https://img-blog.csdn.net/20180102142219048?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVvMTA1NTEyMDIwNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
1.首先在routes文件夹下面建立容器文件
```js
import React, { Component } from 'react';
import { connect } from 'dva';
import { Card, Divider, Row, Col, Input, Button, Table, Pagination, notification } from 'antd';
import moment from 'moment';
import FilterBar from 'components/FilterBar';
import PageHeaderLayout from '../../layouts/PageHeaderLayout';
import styles from './list.less';

const { Search } = Input;

function showTotal(total) {
  return `Total ${total} items`;
}
// 连接model层的state数据，然后通过this.props.state名(namespace)访问model层的state数据
@connect(state => ({
  client: state.clientManager,
  app: state.app,
}))
export default class ClientManager extends Component {
  state = {
    pagination: {
      defaultCurrent: 1,
      defaultPageSize: 10,
      showSizeChanger: true,
      showQuickJumper: true,
    },
  };

  componentDidMount() {
    const { dispatch } = this.props;
    const { defaultCurrent, defaultPageSize } = this.state.pagination;
    const { currentUser, currentInst } = this.props.app;
    const params = {
      skip: defaultCurrent - 1,
      limit: defaultPageSize,
      sort: 'createAt',
      where: { instid: currentUser.id, ownerid: currentInst.id },
    };
    // 触发action触发model层的state的初始化
    dispatch({
      type: 'clientManager/queryList',
      payload: params,
    });
  }
  // 处理表格的关注和取消关注功能
  clientLove = (client) => {
    this.props.dispatch({
      type: 'clientManager/loveClient',
      payload: { client }
    });
  }
  // 分页查询
  paging = (page, pageSize) => {
    const { dispatch } = this.props;
    const { currentUser, currentInst } = this.props.app;
    const params = {
      skip: page - 1,
      limit: pageSize,
      sort: 'createAt',
      where: { instid: currentUser.id, ownerid: currentInst.id },
    };
    dispatch({
      type: 'clientManager/queryList',
      payload: params,
    });
  }

  // 传递给Table组件前的数据处理，数据处理全部放在此处，不要放到表格配置中去，表格配置只有展示相关
  tableSourceDataHandle = (data) => {
    return data.map((item, index) => {
      item.clientLove = this.clientLove;
      item.date = moment(item.date).format('YYYY-MM-DD');
      return item;
    });
  }
  
  render() {
    const { gridInfo, data, tableLoading, userTotal, fiterbarOptions } = this.props.client;
    const tableSourceData = this.tableSourceDataHandle(data)
    const { selectedRowKeys } = this.state;
    const rowSelection = {
      selectedRowKeys,
      onChange: this.onSelectChange,
    };
    return (
      <PageHeaderLayout title="客户管理">
        <Card bordered={false}>
          <FilterBar title="筛选数据" dataSource={fiterbarOptions} filterClick={this.handleFilterClick} refreshClick={this.handleRefreshClick} />
          <Divider />
          <Row className={styles.toolbar} type="flex" justify="space-between" align="middle">
            <Col className={styles.alignCenter}>
              <div className={styles.searchWrap}>
                <Search
                  size="large"
                  placeholder="输入关键词"
                  onSearch={value => console.log(value)}
                />
              </div>
              <div className={styles.info}>选中<span>{selectedRowKeys.length}</span>家<span>|</span>共<span>{tableSourceData.length}</span>家</div>
            </Col>
            <Col>
              <Button type="primary" size="large" icon="user" className={styles.btn} onClick={this.handleAssignManager}>分配负责人</Button>
              <Button type="primary" size="large" icon="download" className={styles.btn} onClick={this.handleInfoDownload}>下载客户资料</Button>
              <Button type="primary" size="large" icon="close" className={styles.btn} onClick={this.handleDeleteClient}>删除客户</Button>
              <Button type="primary" size="large" icon="form" onClick={this.handleEdit}>查看编辑</Button>
            </Col>
          </Row>
          <Table
            rowKey={(record) => record.id}
            loading={tableLoading}
            dataSource={tableSourceData}
            columns={gridInfo.columns}
            pagination={false}
            rowSelection={rowSelection}
          />
          <div className={styles.paginationWrap}>
            <Pagination
              className={styles.pagination}
              total={userTotal}
              {...this.state.pagination}
              showTotal={showTotal}
              onChange={this.paging}
              onShowSizeChange={this.paging}
            />
          </div>
        </Card>
      </PageHeaderLayout>
    );
  }
}
```
2.在容器组件的componentDidMount钩子里面使用dispatch触发models里面的action进行state数据的初始化

3.调用model的effects方法(相当于redux的middleware)，在effects里面可以调用services方法进行异步请求
```js
import { notification } from 'antd';
import { clientManagerListGridInfo, dataHandler } from '../config/grids/index';
import { queryClientList, updateClientLove, queryFilterOptions } from '../services/clientManager';

export default {
  namespace: 'clientManager', //model的state名字，匹配action行为的type属性前缀
  state: {
    gridInfo: clientManagerListGridInfo,
    data: [],
    copyData: [],
    tableLoading: false,
    userTotal: 0,
    fiterbarOptions: [],
  },
  reducers: {
    updateState(state, { payload }) {
      return { 
        ...state,
        ...payload,
      }
    }
  },
  effects: {
    *queryList({ payload }, { call, put, select }) {
	  // effects里面触发action的方法是yield put
      yield put({
        type: 'updateState',
        payload: {
          tableLoading: true,
        },
      });
      // 同时进行多个异步http调用，当最慢的http调用完成后得到返回结果，程序继续向下执行，相当于Promise.all方法
      // effects里面调用services的方法是yield call
      const [clientsResponse, filtersResponse] = yield [
        call(queryClientList, payload),
        call(queryFilterOptions)
      ]
      let data;
      let pageinfo;
      if (clientsResponse.success) {
        pageinfo = clientsResponse.pageinfo;
        // effects里面获取state的方法是yield select
        const columns = yield select(state => state.clientManager.gridInfo.columns)
        data = yield dataHandler(clientsResponse.data, columns)
      }
      // 根据表格数据生成表格上面的过滤条配置
      let fiterbarOptions;
      if (filtersResponse.success) {
        fiterbarOptions = filtersResponse.data;
        const counts = [data.length, 0, 0, 0, 0, 0];
        // 像一般纳税人这种判断条件建议写成配置文件
        data.forEach((item) => {
          if (item.taxtypename === "一般纳税人") {
            counts[1] += 1;
          } else if (item.taxtypename === "小规模纳税人") {
            counts[2] += 1;
          }
          if (item.taxManager === "无") {
            counts[3] += 1;
          }
          if (item.accountManager === "无") {
            counts[4] += 1;
          }
          if (item.love) {
            counts[5] += 1;
          }
        });
        for (let i = 0, length = fiterbarOptions.length; i < length; i += 1) {
          fiterbarOptions[i].count = counts[i];
        }
      }
      // 拿到http请求数据后触发reducer修改state的数据
      yield put({
        type: 'updateState',
        payload: {
          data,
          copyData: [...data],
          userTotal: pageinfo.total,
          clientListHttpParams: payload,
          tableLoading: false,
          fiterbarOptions,
        },
      });
    },
    *loveClient({ payload }, { call, put, select }) {
      if (payload.client) {
        yield put({
          type: 'updateState',
          payload: {
            tableLoading: true,
          },
        });
        const response = yield call(updateClientLove, { client: payload.client });
        if (response.success) {
          const clientListHttpParams = yield select(state => state.clientManager.clientListHttpParams);
          yield put({
            type: 'queryList',
            payload: clientListHttpParams,
          });
        }
        yield put({
          type: 'updateState',
          payload: {
            tableLoading: false,
          },
        });
      }
    },
  }
}
```

4.执行service方法
```js
import request from '../utils/request';

export async function queryClientList(params) {
  return request('/api/biz/md/bizMdClient/search', {
    method: 'POST',
    data: params || {}
  })
}

export async function updateClientLove(params) {
  return request('/api/biz/md/bizMdClient/love', {
    method: 'POST', data: params || {}
  })
}

export async function queryFilterOptions(params) {
  return request('/api/biz/md/bizMdClient/queryFilterOptions', {
    method: 'POST', data: params || {}
  })
}
```

request方法是基于axios封装的进行http请求的工具函数，下面是request.js的实现：
```js
import axios from 'axios';
import { notification } from 'antd';

function checkStatus(response) {
  if (response.status >= 200 && response.status < 300) {
    const { data: result } = response;
    if (result.status) {
      if (result.status >= 200 && result.status < 300) {
        result.success = true; //eslint-disable-line
        return result;
      } else {
        const error = new Error(result);
        result.success = false;
        error.result = result;
        throw error;
      }
    }
  }
  const error = new Error(response.statusText);
  error.response = response;
  throw error;
}

/**
 * Requests a URL, returning a promise.
 *
 * @param  {string} url       The URL we want to request
 * @param  {object} [options] The options we want to pass to "axios"
 * @return {object}           An object containing either "data" or "err"
 */
export default function request(url, options) {
  const defaultOptions = {
    credentials: 'include',
  };
  const newOptions = { ...defaultOptions, ...options };
  if (newOptions.method === 'POST' || newOptions.method === 'PUT') {
    newOptions.headers = {
      Accept: 'application/json',
      'Content-Type': 'application/json; charset=utf-8',
      ...newOptions.headers,
    };
  }

  return axios.create().request({
    url,
    method: options && options.method ? options.method : 'get',
    timeout: 15000, //http请求超时时间
    ...newOptions,
  })
    .then(checkStatus)
    .catch((error) => {
      if (error.code) {
        notification.error({
          message: error.name,
          description: error.message,
        });
      }
      // http请求超时处理
      if ('stack' in error && 'message' in error) {
        const { message } = error;
        if (~message.indexOf('timeout')) {
          notification.error({
            message: `请求错误: ${url}`,
            description: '很抱歉您的请求已经超时了，请稍后再试！',
          });
        } else {
          notification.error({
            message: `请求错误: ${url}`,
            description: error.message,
          });
        }
      }
      const result = { success: false };
      return result;
    });
}
```

5.mock机制（其实也是个node服务）捕获service的http请求，对http请求进行处理，也就是传入request方法的第一个参数url要和.roadhogrc.mock.js文件设置的捕获url一致，下面展示.roadhogrc.mock.js的配置：
```js
import mockjs from 'mockjs';
import { format, delay } from 'roadhog-api-doc';

import user from './mock/user';
import client from './mock/client';
import app from './mock/app';
import members from './mock/members';

// 是否禁用代理
const noProxy = process.env.NO_PROXY === 'true';

const proxy = Object.assign({}, user, client, app, members)

export default noProxy ? {} : delay(proxy, 1000);
```

这里mock/client文件是对我上面展示的表格进行逻辑处理的代码，以下展示client.js代码：
```js
import Mock from 'mockjs';

const { Random } = Mock;

const userListLength = Random.integer(10, 100);
const userList = [];
// 生成表格mock数据
for (let i = 0; i < userListLength; i += 1) {
  userList.push({
    id: i,
    love: Math.random(0, 1) > 0.5,
    taxNo: Random.integer(4e14, 5e14),
    taxtype: i % 3 === 0 ? 'small' : i % 2 === 0 ? 'personal' : 'normal',
    name: `深圳市丝悦化妆品有限公司${i + 1}`,
    no: i % 2 === 0 ? `B${i}` : `T${i}`,
    taxZone: Math.random(0, 1) > 0.5 ? '广东' : '深圳',
    key: Math.random(0, 1) > 0.5,
    sysVerify: Math.random(0, 1) > 0.5,
    taxManager: 'lane',
    accountManager: 'lane',
    date: Random.datetime('yyyy-MM-dd HH:mm:ss'),
  })
}

export default {
  [`POST /api/biz/md/bizMdClient/search`](req, res, u, b) { // 分页查询处理
    const body = (b && b.body) || req.body;
    const { skip, limit } = body;
    const start = skip * limit; // 数据开始索引
    const end = start + (limit * 1); // 数据结束索引
    const dataSource = userList.slice(start, end); // 要返回的数据
    const result = {
      code: 200,
      data: dataSource,
      // 分页信息
      pageinfo: {
        "total": userList.length,
        "pageindex": skip,
        "pagesize": limit,
      }
    };
    if (res && res.json) {
      res.json(result);
    } else {
      return result;
    }
  },
  [`POST /api/biz/md/bizMdClient/love`](req, res) {
    const { client } = req.body
    for (let i = 0, length = userList.length; i < length; i += 1) {
      if (client.id === userList[i].id) {
        userList[i].love = !userList[i].love;
        break;
      }
    }
    res.send({ code: 200 }).end()
  },
  [`POST /api/biz/md/bizMdClient/queryFilterOptions`](req, res) {
    const dataSource = [
      {
        title: '全部',
      },
      {
        title: '一般',
      },
      {
        title: '小规模',
      },
      {
        title: '税务待分配',
      },
      {
        title: '账务待分配',
      },
      {
        title: '我的关注',
      },
    ];
    const result = {
      code: 200,
      data: dataSource,
    };
    if (res && res.json) {
      res.json(result);
    } else {
      return result;
    }
  }
}
```
6.拿到异步请求数据，触发reducer修改state的数据（唯一修改model层state数据的方式就是触发reducer）
7.state数据变化，触发所有连接此state的容器组件调用render方法就行重新渲染 
基本上大体的开发流程就是上面说的这几步

## 7.补充说明？
如何模拟请求不同服务器接口，利用roadhog会很简单，因为roadhog已经提供了代理的接口，你可以在roadhogrc文件里面进行配置，一下展示笔者的配置：
```js
const path = require('path');

const svgSpriteDirs = [
  path.resolve(__dirname, 'src/svg/'),
  require.resolve('antd').replace(/index\.js$/, '')
]

export default {
  "entry": "src/index.js",
  "svgSpriteLoaderDirs": svgSpriteDirs,
  "extraBabelPlugins": [
    "transform-runtime",
    "transform-decorators-legacy",
    "transform-class-properties",
    ["import", { "libraryName": "antd", "libraryDirectory": "es", "style": true }]
  ],
  "env": {
    "development": {
      "extraBabelPlugins": [
        "dva-hmr"
      ]
    }
  },
  proxy: { // 该属性下进行url代理的配置
    "/api/ftm": {
      "target": "http://120.79.88.200:8761/",
      "changeOrigin": true,
    },
    "/api/md": {
      "target": "http://120.79.88.200:8761/",
      "changeOrigin": true,
    },
    "/api/bill": {
      "target": "http://120.79.88.200:8761/",
      "changeOrigin": true,
    },
  },
  "externals": {
  },
  "ignoreMomentLocale": true,
  "theme": "./src/theme.js"
}
```

当然真实的代理是需要后端支持的，笔者前端项目下有一个基于express的node微后台，负责登录验证和接口代理，使用`http-proxy-middleware` 进行代理，以下是node后台的入口文件（有兴趣的可以看，没有兴趣看也不要紧，可以寻求后端童鞋帮助）：
```js
const express = require('express');
const path = require('path');
const favicon = require('serve-favicon');
const cookieParser = require('cookie-parser');
const compression = require('compression');
const history = require('connect-history-api-fallback');

const passport = require('./passport');
const proxys = require('./routes/proxys');
const users = require('./routes/users');
const config = require('./config');

const app = express();

// for gzip
app.use(compression());

// 配置日志
require('./logger')(app);

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

// uncomment after placing your favicon in /public
app.use(favicon(path.join(__dirname, 'public', 'logo.png')));
// app.use(bodyParser.json());
// app.use(bodyParser.urlencoded({ extended: true }));
app.use(cookieParser());
// 配合前端使用history
// 设置路由

app.use('/api', proxys);
app.use('/users', users);
app.use(history());
app.use(express.static(path.join(__dirname, 'public')));
app.use(require('express-session')({ secret: 'Gi6zDvtS5!AC4hV13Wv@H9kl5^@ItBl0', resave: true, saveUninitialized: true }));

// 加载配置
config(app);

// 设置passport
passport(app);


// catch 404 and forward to error handler
app.use((req, res, next) => {
  const err = new Error('Not Found');
  err.status = 404;
  next(err);
});

// error handler
app.use((err, req, res) => {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```

下面是代理的处理文件：
```js
/**
 * 反向代理
 */
const express = require('express');

const router = express.Router();
const proxy = require('http-proxy-middleware');

// 代理服务
router.use('/', proxy({
  target: 'http://120.79.88.200:8761',
  changeOrigin: true,
  onProxyReq(proxyReq, req) {
    if (req.user && req.user.accessToken) { proxyReq.setHeader('Authorization', `Bearer ${req.user.accessToken}`); }
  },
  ws: true,
}));

module.exports = router;
```

关于代码中用到的库，比如mockjs，qs，lodash，roadhog等等请自行到github进行搜索学习，关于项目如何进行性能优化且听下次分解，如果有不对的地方欢迎指正，非常感谢！