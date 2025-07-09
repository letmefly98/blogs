# 记一次全栈开发经历

最近项目中需要

## 后端脚手架搭建

采用 `Koa` 服务，想全用 `typescript` 编写。最好有热更新方便调试，

### 开发时

```bash
npm i koa @koa/router
npm i -D typescript tsx @types/node @types/koa @types/koa__router
```

#### 基础代码

```ts
// src/server.ts
import Router from '@koa/router';
import Koa from 'koa';

const API_PROT = 4002;
const app = new Koa();
const router = new Router();

app.use(router.routes());
app.use(router.allowedMethods());

app.listen(API_PROT, () => {
  console.log(`server is running on prot: ${API_PROT}`);
});
```

运行 `tsx ./src/server.ts` 无误即完成了最基本的后端服务。

#### 热更新

安装 `nodemon` 后，将 `tsx ./src/server.ts` 写到 `nodemon.json` 配置中。

```json
// nodemon.json
{
  "watch": ["src"],
  "ext": "ts,cjs,mjs,js,json",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "tsx ./src/server.ts"
}
```

再运行 `nodemon` 命令，即可实现热更新。每次修改代码后，服务会自动重启。

```json
// package.json
{
  "scripts": {
    "dev": "nodemon",
  }
}
```

### 编写接口

```ts
// src/apis/login/index.ts
interface HttpReqLogin {
  txt?: string;
  error?: string;
}
interface HttpResLogin {
  msg: string;
}
const hello: Middleware = async (ctx, next) => {
  const props = <HttpReqLogin>ctx.request.query;
  if (props.error) {
    ctx.throw(new Error(props.error));
    return;
  }
  const result: HttpResLogin = { msg: `hello world: ${props.txt || ''}` }
  ctx.success(result);
  await next();
};
export default login;
```

```ts
// src/server.ts
import login from './apis/login/index.ts';

router.get('/login', login);
app.use(router.routes());
```

至此，在浏览器中访问 `http://localhost:4002/login` 有正确内容，你则完成了接口开发。

#### 中间件

比如统一接口返回的结构、于一处增加权限验证、打印每次请求出入参等，都可以通过中间件来实现。

```ts
// src/middleware/log-params-and-return-han.ts
// 中间件：打印所有接口的出入参
interface Options {}
export function logParamsAndReturnHandler(opt?: Options): Middleware {
  return async (ctx, next) => {
    console.log('入参：', ctx.request.body || ctx.request.query);
    await next();
    console.log('出参：', ctx.body);
  }
}
```

`Koa` 是洋葱模型的设计策略，所以需要把控 `next` 函数的调用顺序。

```ts
// 伪代码
function mid1(ctx, next) {
  console.log('1a');
  await next();
  console.log('1b');
}
function mid2(ctx, next) {
  console.log('2a');
  await next();
  console.log('2b');
}

// 打印为：1a 2a 2b 1b
app.use(mid1);
app.use(mid2); 

 // 打印为：2a 1a 1b 2b
app.use(mid2);
app.use(mid1);
```

因此，最早的 `before` 和最晚的 `after` 需排在前面。

比如我就是这样 `body-parse(before)` `handle-error(after)` `add-custom-context-methods(before)` `add-trace-id(before+after)` `use-common-return(after)` 最后 `register-apis`

### 打包

我采用的 `vite` 来打包。减少一些兼容 `cjs` 与 `esm` 、摇树优化等方面处理。

```ts
// vite.config.ts
import type { UserConfig } from 'vite';

export default (): UserConfig => {
  return {
    build: {
      ssr: true,
      lib: {
        entry: 'src/server.ts',
        formats: ['cjs'],
      },
    },
    ssr: {
      noExternal: true,
    },
  };
};
```

运行 `vite build` 即可得到 `./dist/server.cjs` 打包结果文件。

使用 `node ./dist/server.cjs` 即可启动服务。

## 部署问题

使用两个 docker 当然可以，但我更倾向于一个 docker 然后内部用 `supervisord` 或 `pm2` 来管理两个进程。

比如网页使用 `8080` 默认端口，后端使用 `4002` 端口，然后 `nginx` 代理 `/f-api` 到 `localhost:4002` 上。

那么 `{domain}/f-api/hello` 即访问到后端接口上了。

```dockerfile
## Dockerfile
FROM node:20.18.0-alpine

RUN apk add nginx supervisor

RUN mkdir -p /var/log/supervisor
RUN mkdir -p /etc/supervisor/conf.d

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

WORKDIR /usr/share/nginx/html
COPY . .
RUN mv -f nginx.conf /etc/nginx
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

```conf
## supervisord.conf
[supervisord]
nodaemon=true

[program:node]
command=node server.cjs
autostart=true
autorestart=true
environment=PORT=4002
stderr_logfile=/var/log/node.err.log
stdout_logfile=/var/log/node.out.log

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
stderr_logfile=/var/log/nginx.err.log
stdout_logfile=/var/log/nginx.out.log
```
