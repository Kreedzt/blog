---
author: "Kreedzt"
title: "Rx.js 入门"
date: "2023-02-01"
description: "该文章内容为 https://www.youtube.com/channel/UCVyRiMvfUNMA1UPlDPzG5Ow 的翻译"
tags: ["web", "javascript"]
---

# Rx.js 入门

> ReactiveX 通常用来解决异步的繁琐操作, ReactiveX 中所有数据都看作"流", 也是"流式编程"的集大成者, 并运用 ReactiveX 的操作符我们可以轻松联动多个异步操作之间的关系

## 起手文件配置

项目结构

```
|-src/
|---code.ts
|-index.html
|-package.json
|-tsconfig.json
|-webpack.config.js
```

`package.json`:

```json
{
  "name": "rxjs-learn",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "start": "webpack-dev-server --mode development"
  },
  "dependencies": {
    "rxjs": "^6.4.0",
    "rxjs-compat": "^6.4.0",
    "ts-loader": "^5.3.3",
    "typescript": "^3.4.3",
    "webpack": "^4.30.0",
    "webpack-dev-server": "^3.3.1"
  },
  "devDependencies": {
    "webpack-cli": "^3.3.0"
  }
}
```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "outDir": "./dist/",
    "noImplicitAny": true,
    "module": "es2015",
    "moduleResolution": "node",
    "sourceMap": true,
    "target": "es6",
    "typeRoots": ["node_modules/@types"],
    "lib": ["es2017", "dom"]
  }
}
```

`webpack.config.js`:

```js
const path = require("path");

module.exports = {
  entry: "./src/code.ts",
  devtool: "inline-source-map",
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: [".ts", ".js", ".tsx"],
  },
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
};
```

`index.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Learn rxjs</title>
    <style>
      body {
        font-family: "Arial";
        background: #ececec;
      }
      ul {
        list-style-type: none;
        padding: 20px;
      }
      li {
        padding: 20px;
        background: white;
        margin-bottom: 5px;
      }
    </style>
  </head>
  <body>
    <ul id="output"></ul>

    <script src="/bundle.js"></script>
  </body>
</html>
```

`src/code.ts`:

```typescript
import * as Rx from "rxjs/Observable";

console.log(Rx);
```

## 教程

### 从 Observable 开始

修改 `code.ts` 文件为:

```typescript
import { Observable } from "rxjs/Observable";

var observable = Observable.create((observer: any) => {
  observer.next("Hey guys!");
});

observable.subscribe((x: any) => console.log(x));
```

浏览器控制台输出"Hey guys!".

#### 理解 Observable

1. Observable.create 方法创建一个可观察对象, 接受一个函数, 参数类型为 Observer
2. Observable 属于不订阅(subscribe), 不执行的原则
3. Observable 的 complete 方法调用即表示'完成'状态, 该方法不能传递参数,
4. 每个可观察对象都有一个 subscribe 订阅方法, 可接受 1-3 个函数作为参数, 也可接受一个对象: 格式为

```typescript
// 对象作为参数的格式
observable.subscribe({
  next: (nextVal) => console.log("got value " + nextVal),
  error: (err) => console.error("something wrong occurred: " + err),
  complete: () => console.log("done"),
});
// 函数作为参数的格式 -- 主要使用此格式
observable.subscribe(
  (next) => console.log(next),
  (error) => console.error(error),
  () => console.log("complete")
);
```

初看 observable 可能难以理解, 我们不妨拿 ~Promise~ 来作为对比:

```typescript
// 示例Promise代码
function createPromise = (success) => {
	return new Promise((res, rej) => {
    if (success) {
      res('success');
    } else {
    	rej('error');
    }
  });
}

createPromise
  .then(complete => console.log('complete'))
  .catch(error => console.error('error'));

// 示例Observable代码
import { Observable } from "rxjs/Observable";

var observable = Observable.create((observer: any) => {
  observer.next("Hey guys!");
  observer.complete();
  observer.error('error!');
});

observable.subscribe(
  (x: string) => console.log(x),
  (err: any) => console.error(err),
  () => console.log('complete')
);
```

|                | Promise                                            | Observable                                                                |
| -------------- | -------------------------------------------------- | ------------------------------------------------------------------------- |
| 创建           | 接受一个函数, 2 个参数, 分别实现 complete 和 error | 接受一个函数, 一个参数, 该参数有 next, complete 和 error 3 个函数分别派发 |
| 订阅           | then 接受一个成功函数, catch 接收一个失败函数      | subscribe 可接受 3 个函数, 分别接收 next/error 和 complete 事件           |
| 是否可多次派发 | 否                                                 | 是                                                                        |

至此可以 **暂时** 理解 `Observable` 为 `Promise` 的加强版, `Observable` 可以通过调用 `observer.next()` 来派发多次值, 而 `Promise` 的状态一旦改变一次就不可修改,而 `observer.complete()` 状态才是确定状态完成, `complete` 调用后 `observer` 的状态才不可修改

#### 添加 DOM 元素以优雅的展示效果

修改 `code.ts` 的 `subscribe` 代码, 改变展示效果:

```typescript
// ...
observable.subscribe((x: string) => addItem(x));

// 添加DOM元素以便优雅的展示效果
function addItem(val: string) {
  var node = document.createElement("li");
  var textNode = document.createTextNode(val);
  node.appendChild(textNode);
  document.getElementById("output").appendChild(node);
}
```

#### observer 的 next 和 complete

上面对比说过, `next` 可多次调用, `complete` 调用后就不可再次派发了, 现在我们修改 `code.ts` 代码, 来展示 `next` 的多次调用和 `complete` 的效果:

```typescript
// ts导入Observer类型方便代码提示
var observable = Observable.create((observer: Observer<string>) => {
  observer.next("Hey guys!");
  observer.next("how are you?");
  observer.complete();
  console.log("just test this code ");
  observer.next("This will not send");
});

observable.subscribe(
  (x: string) => addItem(x),
  (error: string) => addItem(error),
  () => addItem("Completed")
);
```

页面中发现在 `complete()` 前的 `next()` 代码可以执行, `complete()` 后面的 `next()` 没有执行, 但是 `console.log()` 代码依旧执行了.
也就是说, **仅仅不能派发值了, 其他代码依旧可以执行**.

#### observer 与定时器

除了 `subscribe()` 方法可以订阅 observer 外, 我们还可以使用 `subscribe()` 返回的实例调用 `unsubscribe()` 取消订阅, 以便内存及时释放

修改代码, 我们把 observer 与定时器组合使用

```typescript
var observable = Observable.create((observer: Observer<string>) => {
  try {
    observer.next("Hey guys!");
    observer.next("how are you?");
    setInterval(() => {
      observer.next("I am good");
    }, 2000);
  } catch (err) {
    observer.error(err);
  }
});

var observer = observable.subscribe(
  (x: string) => addItem(x),
  (error: string) => addItem(error),
  () => addItem("Completed")
);

setTimeout(() => {
  observer.unsubscribe();
}, 6001);
```

因为是 2 秒触发一次'i am goood', 我们在 6.001 秒后取消了订阅, 所以该字符串只添加了 3 次

我们可以订阅多次 observer, 就像调用多次 `Promise` 的订阅一样

```typescript
var observer2 = observable.subscribe(
  (x: string) => addItem(x),
  (error: string) => addItem(error),
  () => addItem("Completed")
);
```

我们发现间歇定时器只有 1 个销毁了, 我们订阅 2 次产生了 2 个定时器, observer2 一直在派发值
我们当然可以在 `setTimeout` 中对 observer2 取消订阅, 但是 observer 实例存在一个 add()方法, 可以把第 2 个 observer 实例添加在一个实例上

```typescript
observer.add(observer2);

setTimeout(() => {
  observer.unsubscribe();
}, 6001);
```

这样取消订阅时候, 定时器不再触发

---

重改代码, 实现 2 个 observer 并列运行

```typescript
var observer = observable.subscribe(
  (x: string) => addItem(x),
  (error: string) => addItem(error),
  () => addItem("Completed")
);

setTimeout(() => {
  var observer2 = observable.subscribe((x: string) =>
    addItem("Subscriber 2: " + x)
  );
}, 1000);
```

通常我们不需要定时器执行前的代码重复执行

```typescript
try {
  // 仅需要执行一次 ↓
  observer.next("Hey guys!");
  observer.next("how are you?");
  // ----
  setInterval(() => {
    observer.next("I am good");
  }, 2000);
} catch (err) {
  observer.error(err);
}
```

完成这个需求, 可以引入 rxjs 的 share 方法:

```typescript
import "rxjs/add/operator/share"; // 引入share

var observable = Observable.create((observer: Observer<string>) => {
  try {
    observer.next("Hey guys!");
    observer.next("how are you?");
    setInterval(() => {
      observer.next("I am good");
    }, 2000);
  } catch (err) {
    observer.error(err);
  }
}).share(); // 调用share
```

我们查看[share()的文档]():

![rxjs-share](/rxjs-share.png)

share()使得 observable 变为多可播源, 每次 subscribe 此 observable 不会重新创建新的订阅流程
我们用最容易理解的方式来注释以下代码:

```typescript
var observable = Observable.create((observer: Observer<string>) => {
  try {
    observer.next("Hey guys!");
    observer.next("how are you?");
    // 以上代码在1秒内必定执行完成, 所以observer2订阅时候, 仅订阅到了setInterval内部的代码
    setInterval(() => {
      observer.next("I am good");
    }, 2000);
  } catch (err) {
    observer.error(err);
  }
}).share();

// 订阅立即执行, observerable不会由于第二次订阅而重新执行
var observer1 = observable.subscribe(
  (x: string) => addItem(x),
  (error: string) => addItem(error),
  () => addItem("Completed")
);

setTimeout(() => {
  var observer2 = observable.subscribe((x: string) =>
    addItem("Subscriber 2: " + x)
  );
}, 1000);
// 1秒后订阅observable, 此时observer已经派发出了2个next(), 仅订阅后续的派发, 因为share()包装后的observable不会再次执行里面的代码
```

流程:
![rxjs-流程1](/rxjs-流程1.png)

---

#### fromEvent

fromEvent 可以让我们对一个 DOM 元素和定义的事件来创建 observable
导入 fromEvent, 订阅事件:

```typescript
import { fromEvent } from "rxjs/observable/fromEvent";

var observable = fromEvent(document, "mousemove");

setTimeout(() => {
  var subscription = observable.subscribe((x: any) => addItem(x));
}, 2000);
```

鼠标移入页面 2 秒后, 执行了订阅代码

### 使用 Subject

Subject 适用于不确定派发时机的情况, 可以先订阅, 后派发, 例子如下:

```typescript
import { Subject } from "rxjs/Subject";

var subject = new Subject();

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

subject.next("The first thing has been sent");
```

与 Observable 同样的取消订阅方式:

```typescript
import { Subject } from "rxjs/Subject";

var subject = new Subject();

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

subject.next("The first thing has been sent");

var observer2 = subject.subscribe((data) => addItem("Observer 2: " + data));

subject.next("The second thing has been sent");
subject.next("The third thing has been sent");

observer2.unsubscribe();

subject.next("A final thing has been set");
```

#### BehaviorSubject

BehaviorSubject 接收一个初始派发值, 并且可以回放最后一次的派发值给下一订阅者

```typescript
import { BehaviorSubject } from "rxjs/BehaviorSubject";

// 初次输出First
var subject = new BehaviorSubject("First");

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

subject.next("The first thing has been sent");

subject.next("...Obserber 2 is about subscribe");

// 上行代码observer2也接收到
var observer2 = subject.subscribe((data) => addItem("Observer 2: " + data));
```

#### ReplaySubject

ReplaySubject 接收一个参数时, 参数作为回放给下一派发者的次数:

```typescript
import { ReplaySubject } from "rxjs/ReplaySubject";

var subject = new ReplaySubject(2);

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

subject.next("The first thing has been sent");
// observer2从此处开始接收
subject.next("Another thing has been sent");
subject.next("...Obserber 2 is about subscribe...");

var observer2 = subject.subscribe((data) => addItem("Observer 2: " + data));
```

ReplaySubject 接收 2 个参数时, 第二个参数作为回放时间:

```typescript
import { ReplaySubject } from "rxjs/ReplaySubject";

// 回放时间为200ms
var subject = new ReplaySubject(30, 200);

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

var i = 1;
var int = setInterval(() => subject.next(i++), 100);

setTimeout(() => {
  // 由于回放时间为200ms, 所以接收从300ms(500 - 200)开始的值
  var observer2 = subject.subscribe((data) => addItem("Observer 2: " + data));
}, 500);
```

此时 observer2 从 3 开始输出, 若修改 200 为 500 即可使接收所有值

#### AsyncSubject

AsyncSubject 的 subscription 仅派发 complete 前的最后一个 next

```typescript
import { AsyncSubject } from "rxjs/AsyncSubject";

var subject = new AsyncSubject();

subject.subscribe(
  (data) => addItem("Observer 1: " + data),
  (err) => addItem(err),
  () => addItem("Observer 1 Completed")
);

var i = 1;
var int = setInterval(() => subject.next(i++), 100);

setTimeout(() => {
  var observer2 = subject.subscribe((data) => addItem("Observer 2: " + data));
  subject.complete(); // 标识已完成: 若注释此代码, 订阅不会触发
}, 500);
```

我们尝试注释 subject.complete()代码, 订阅不会执行, 页面空无一物

### 操作符

操作符被调用时，它们不会**改变**已经存在的 Observable 实例。相反，它们返回一个**新的** Observable ，它的 subscription 逻辑基于第一个 Observable 。
操作符可以处理派发的数据, 也可以处理 Observable 之间的关系.

#### merge: 合并多个 Observable

就如单词意思一样, merge 操作符可以合并多个 Observable 为一个新的 Observable
[官方文档](https://cn.rx.js.org/class/es6/Observable.js~Observable.html#static-method-merge)

![merge](/rxjs-merge.png)

```typescript
import { Observable } from "rxjs/Observable";
import { Observer, merge } from "rxjs";
import "rxjs/observable/merge";

var observerble = Observable.create((observer: Observer<string>) => {
  observer.next("Hey guys!");
});

var observerble2 = Observable.create((observer: Observer<string>) => {
  observer.next("How is it going?");
});

var newObservable = merge(observerble, observerble2);
newObservable.subscribe((next: string) => addItem(next));
```

页面应显示 2 个字符串

#### map: 处理派发数据

类似 Array.proptotype.map, 此操作处理每一次数据派发的值

```typescript
import "rxjs/add/operator/map";

Observable.create((observer: Observer<string>) => {
  observer.next("Hey guys!");
})
  .map((val: string) => val.toUpperCase())
  .subscribe((next: string) => addItem(next));
```

页面应显示大写的字符串

#### pluck: 挑选数据

此处我们使用 from 来从数组新建一个 Observable([from 的文档(此处中文文档没有此 API 的详细介绍)](https://rxjs.dev/api/index/function/from))
使用 pluck 来挑选我们仅需要的属性: ([pluck 文档](https://cn.rx.js.org/class/es6/Observable.js~Observable.html#instance-method-pluck))

```typescript
import { from } from "rxjs/observable/from";
import "rxjs/add/operator/pluck";

from([
  {
    first: "Gary",
    last: "Simon",
    age: "34",
  },
  {
    first: "Jane",
    last: "Simon",
    age: "34",
  },
  {
    first: "John",
    last: "Simon",
    age: "34",
  },
])
  .pluck("first")
  .subscribe((next: string) => addItem(next));
```

页面上仅显示了数组每项的 first 属性

#### skipUntil: 跳过目标 Observable 开始派发过程中的值

```typescript
import { Observable } from "rxjs/Observable";
import { Subject } from "rxjs/Subject";
// import { interval } from "rxjs/observable/interval";
import "rxjs/add/operator/skipUntil";
import { Observer } from "rxjs";

var observable1 = Observable.create((data: Observer<number>) => {
  var i = 1;
  setInterval(() => {
    data.next(i++);
  }, 1000);
});

var observable2 = new Subject();
setTimeout(() => {
  observable2.next("Hey !");
}, 3000);

var newObservable = observable1.skipUntil(observable2);

newObservable.subscribe((x: number) => addItem("" + x));
```

流程图:

![rxjs-流程2](/rxjs-流程2.png)

## 总结

在数据流的操作中, 我们可以很清晰的观察数据变化并进行时间穿梭
初学者可以先仅用 Observable 的创建和订阅, 逐步使用高级操作符
不同创建 Observable 的方式:

1. Observable:

使用 Observable.create 立即创建, 并定义派发函数, 随后订阅

2. Subject

- Subject:

实例化后, 先订阅后派发

- BehaviorSubject

使用方式同 Subject, 但为下一次订阅保留了上一次 next 派发值(快照)

- ReplaySubject

使用方式同 Subject, 但可传递 2 个参数, 第一个参数接收上 n 次的派发, 第二个参数为快照时间(即下一订阅者往前推算的可接受快照时间)

- AsyncSubject

使用方式同 Subject, 但必须调用 complete 方法, 调用 complete 后派发最近一次的 next

> 尽管 Rx.js 可以解决大多异步的繁琐问题, 但使用 Rx 前请先考量团队学习成本, 不要仅仅为了使用单一操作符而去为团队增加过重的负担, Observable 订阅也要及时销毁, 否则会严重消耗内存, 影响客户端体验.
