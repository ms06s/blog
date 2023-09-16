---
author: rs305
pubDatetime: 2023-09-16
title: koa 简单搭建 mock 服务
description: koa 简单搭建 mock 服务
featured: true
tags:
  - mock
  - nodejs
  - koa
---

# Mock 客户端简单使用

## 简单使用

```js
// src/_mock/index.js
import Mock from "mockjs";

Mock.mock("/api/test", "get", () => {
  return {
    errno: 0,
    data: {
      name: "rs305",
    },
  };
});
```

引入组件

```jsx
import React from "react";
import { useEffect } from "react";
import "../_mock/index";

const MockDemo = () => {
  useEffect(() => {
    fetch("/api/test")
      .then(res => res.json())
      .then(data => console.log("featch data", data));
  }, []);
};

export default MockDemo;
```

此时开启预览服务

![image-20230916130824652](/src/content/blog/koa%20简单搭建%20mock%20服务/img/image-20230916130824652.png)
查看 network 面板，发现有正确请求

![image-20230916131021030](/src/content/blog/koa%20简单搭建%20mock%20服务/img/image-20230916131021030.png)

原因：mock 原理为劫持 XMLHttpRequest 请求，fetch 不能劫持

那么我们引入 axios 重新编写代码

```jsx
useEffect(() => {
  axios.get("/api/test").then(({ data }) => console.log(data));
}, []);
```

正确劫持

![image-20230916131544216](/src/content/blog/koa%20简单搭建%20mock%20服务/img/image-20230916131544216.png)

## 关于客户端使用 mock 的一些局限性

- mock.js 只能劫持 XMLHttpRequest 请求，不能劫持 fetch
- 生产环境(上线时)需要注释掉，否则线上请求也会被劫持
- 包体积过大

# 使用 Koa 简单搭建服务

## 定义 mock 数据格式

定义 mock 数据格式

```js
// mock/question.js
const Mock = require("mockjs");

const Random = Mock.Random;

module.exports = [
  {
    url: "/api/question/:id",
    method: "get",
    response() {
      return {
        errno: 0,
        data: {
          id: Random.id(),
          title: Random.ctitle(),
        },
      };
    },
  },
  {
    url: "/api/question",
    // post 请求，使用 postman 进行校验
    method: "post",
    response() {
      return {
        errno: 0,
        data: {
          id: Random.id(),
        },
      };
    },
  },
];
```

```js
// mock/test.js
const Mock = require("mockjs");

const Random = Mock.Random;

module.exports = [
  {
    url: "/api/test",
    method: "get",
    response() {
      return {
        errno: 0,
        data: {
          name: Random.cname(),
        },
      };
    },
  },
];
```

整合

```js
// mock/index.js
const test = require("./test");
const question = require("./question");

const mockList = [...test, ...question];

module.exports = mockList;
```

## 创建 koa 服务

```js
const Koa = require("koa");
const Router = require("koa-router");
const mockList = require("./mock");

const app = new Koa();
const router = new Router();

// 模拟真实网络请求，1s 返回数据
async function getRes(fn) {
  return new Promise(resolve => {
    setTimeout(() => {
      const res = fn();
      resolve(res);
    }, 1000);
  });
}

// 注册路由
mockList.forEach(item => {
  const { url, method, response } = item;
  router[method](url, async ctx => {
    const res = await getRes(response);
    ctx.body = res;
  });
});

app.use(router.routes());
app.listen(3001);
```

nodemon 开启服务之后，即可访问对应请求

# 解决跨域问题

```js
// vite.config
server: {
  proxy: {
    '/api': {
      target: 'http://localhost.com:3001',
    },
  },
}
```
