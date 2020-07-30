
在了解工作线程的具体用法之前，有必要先想想：工作线程解决了什么问题？

工作线程主要解决的是cpu密集型场景下的问题，由于node只有单个主线程的特性，导致在执行高cpu运算任务时，会有以下的问题：

1. 计算任务阻塞主线程，导致无法响应新的请求
2. 只能单核执行，无法充分利用多核cpu

而工作线程通过开启在主线程中开启新的线程单独执行计算任务，避免了阻塞整个事件循环，使主线程仍然可以继续处理后续的请求。

并且由于是新的线程，可以在其他cpu核心上执行，使得单个进程可以更充分的利用多核cpu。


## 用法介绍

### 生成工作线程

```js
const { Worker } = require('worker_threads');
let worker = new Worker('工作线程的js文件路径', { });
```

### 线程通信
```js
const { parentPort } = require('worker_threads');
//计算结果 res
parentPort.postMessage(res); //向主线程返回结果
```

其他用法见 [node文档](http://nodejs.cn/api/worker_threads.html#worker_threads_class_messageport)

## 示例

下面是一个阻塞主线程以及使用工作线程优化的示例。[示例代码](https://github.com/hhgfy/demos/tree/master/node/%E5%B7%A5%E4%BD%9C%E7%BA%BF%E7%A8%8Bworker_threads)

使用koa框架启动了一个简单的http服务，包含以下三个api

1. `/test` ：用于测试主线程是否被阻塞，正常情况应当立即返回

2. `/fib`：用暴力方式计算斐波那契数，随着输入的n增大，运算量呈指数增长，返回计算结果

3. `/asyncFib`：将计算交由工作线程执行

```js
//app.js
const { Worker } = require('worker_threads');

const Koa = require('koa');
const Router = require('koa-router') 

const app = new Koa();
const router = new Router(); // 实例化路由
app.use(router.routes());

router.get('/test', async (ctx, next) => {
  ctx.body = 'test';
});

router.get('/fib',async (ctx,next)=>{
  let n = ctx.query.n;
  ctx.body = fib(n);
})

router.get('/asyncFib',async (ctx,next)=>{
  let n = ctx.query.n;
  ctx.body = await asyncFib(n);
})

app.listen(3000);
console.log('http://127.0.0.1:3000');

function fib (n) {
  if (n === 1 || n === 2) return 1;
  return fib(n - 1) + fib(n - 2);
}

async function asyncFib (n) {
  let worker = new Worker('./fib.js', { workerData: n });
  return new Promise((resolve) => {
    worker.on('message', (val) => {
      resolve(val); //接收工作线程计算完毕后返回的结果
    });
  });
}
```

- 工作线程运行的代码
```js
// fib.js
const { workerData, parentPort } = require('worker_threads');
let num = workerData;//获取参数
let res = fib(num);
parentPort.postMessage(res); //向主线程返回结果

function fib (n) {
  if (n === 1 || n === 2) return 1;
  return fib(n - 1) + fib(n - 2);
}
```

## 测试


### 测试阻塞主线程
1. 访问 `http://127.0.0.1:3000/test`， 立刻返回响应test
2. 访问 `http://127.0.0.1:3000/fib?n=44`，处于pending状态，在我的测试中约10秒后返回
3. 再次访问 `http://127.0.0.1:3000/test`，也处于peding状态，要等到`/fib`请求结束后才能收到响应

可以看到，请求2不仅自己执行的慢，还影响到了后续请求，令其他请求都要等待它执行完成后才能处理。

### 测试使用工作线程
1. 访问 `http://127.0.0.1:3000/test`， 立刻返回响应test
2. 访问 `http://127.0.0.1:3000/asyncFib?n=44`，处于pending状态，也是约10秒后返回
3. 再次访问 `http://127.0.0.1:3000/test`，立刻返回响应test


请求2不再阻塞主线程

---
## 如何理解

就表现来说，加上了工作线程，请求就突然不被阻塞了。要如何理解这个现象？


首先要明确一点，在优化前后斐波那契数的计算量是固定的，该做的计算并不会凭空消失。工作线程是通过在另一个cpu核心上运行单独线程进行计算，从而实现不阻塞主线程。

我们都知道node.js 可以非阻塞的执行io操作，可以将工作线程与它进行类比，便于理解。


举个栗子：假设有这么一个服务，它以http接口的形式提供了一个计算斐波那契数的api `http://api.test.com/fib?n=40` ，再由我们的服务去调用它，这个流程与工作线程执行流程的对照如下：

调用外部的计算服务 | 使用工作线程
---|---
发起http请求，交给`io线程`，注册一个回调等待响应 <br><br>`request(url, (res)=>{})`| 生成`工作线程`，监听message事件等待响应 <br><br>`worker.on('message',(res)=>{})`
继续响应其他请求 | 继续响应其他请求
远程的计算服务获取参数n，**用远程服务器的cpu** 计算斐波那契数 | 工作线程获取参数n，**用本机的其他空闲cpu核心** 计算斐波那契数
返回结果，执行回调 | 返回结果，执行回调


可以看出，除了不阻塞主线程这一个好处之外，工作线程给node.js提供了单进程情况下使用多核cpu的能力，可以承载更多的计算量。



