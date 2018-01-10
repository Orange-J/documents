# HTTP事务剖析

> 原文地址: https://nodejs.org/en/docs/guides/anatomy-of-an-http-transaction/

本篇指南的目的是让读者对Node.js的HTTP处理过程有个清晰的理解. 我们假定读者已经了解一般意义上的, 无关特定编程语言和环境的HTTP请求是如何工作的. 我们也假定读者对Node.js的 `EventEmitters` 和 `Streams` 有一定程度的了解. 如果你对这两者不熟悉, 建议快速阅读API文档先了解一下.

# 创建Server

任何Node.js的web服务器应用都必须创建一个web服务器对象. 这个操作用 `createServer` 可以完成.

```
const http = require('http');

const server = http.createServer((request, response) => {
  // magic happens here!
});
```

对于每个到达server的HTTP请求, 传入 `createServer` 的函数都会被调用一次, 因此这个函数被称为请求处理函数. 实际上, `createServer` 返回的 `Server` 对象是个 `EventEmitter`. 下面代码是一个用于创建 `server` 对象并添加监听器的简写形式.

```
const server = http.createServer();
server.on('request', (request, response) => {
  // the same kind of magic happens here!
});
```

当一个HTTP请求到达server时, Node.js会调用请求处理函数并传入 `request` 和 `response` 这两个用于处理事务的对象. 很快我们就会讲到.

为了真正地处理请求, 需要调用 `server` 对象的 `listen` 方法. 在多数情况下, 你只需向 `listen` 传入需要监听的端口号. 其实还有一些其他的选项, 可以查询<a href="https://nodejs.org/api/http.html">API文档</a>.

# Method(请求类型, 如POST/GET), URL 和 Headers(请求头)

当处理一个请求时, 你可能首先会查看请求类型(method)和URL, 然后选择合适的处理方式. 通过将常用的属性放在 `request` 对象上, Node.js使得相关处理变得轻松.

```
const { method, url } = request;
```

> 注意: `request` 对象是 `IncomingMessage` 的一个实例.

这里的 `method` 总是指一个标准的HTTP请求类型(如GET/POST等). `url` 是不带服务器地址, 协议和端口的完整URL. 对于一个标准的URL来讲, `url` 是指第三个斜杠及其之后的所有内容(译者注: 对于"https://www.xx.com:8080/login.html"来讲, 其 `url` 是指"/login.html").

Headers(请求头部)也差不多, 它们在 `request` 对象的 `headers` 属性中.

```
const { headers } = request;
const userAgent = headers['user-agent'];
```

值得注意的是, 所有的headers都是以小写形式展现的, 不管客户端实际发送的是大写还是小写. 这样可以简化解析headers的过程.

如果有的头部是重复的, 那么该头部的值会被覆盖或者合并为以逗号分隔的字符串, 具体怎么处理取决于该头部. 有时候重复头部会导致问题, 因此可以通过 `rawHeaders` 获得原始值.

# 请求体

当接收到 `POST` 或 `PUT` 请求时, 请求携带的请求体对应用来说可能很重要. 获取请求体的数据比请求头数据要复杂一点. 传入请求处理函数的 `request` 对象实现了 `ReadableStream` 接口. 这个**流(stream)**像其他流一样可以被监听, 也可以通过管道输送到其他地方. 我们可以通过监听它的 `data` 和 `end` 事件, 直接从流中抓取数据.

随着每次 `data` 事件的触发而获取到的数据块(chunk)是一个 `Buffer`. 如果你已经知道数据是字符串类型, 那么收集它们的最佳方式是将数据块放到一个数组内, 然后在 `end` 事件触发时将数组内容合并再转换为字符串.

```
let body = [];
request.on('data', (chunk) => {
  body.push(chunk);
}).on('end', () => {
  body = Buffer.concat(body).toString();
  // at this point, `body` has the entire request body stored in it as a string
});
```

> 注意: 这看起来似乎有些冗长, 很多时候确实是这样. 幸运的是, npm上有像 `concat-stream` 和 `body` 这样的模块可以帮助我们隐藏掉这些处理逻辑. 但是在你使用这些模块之前, 深入理解这些细节也是很重要的, 这也是你要阅读本文的原因!

# 关于错误处理的快速讨论

`request` 对象不仅是一个 `ReadableStream`, 它也是一个 `EventEmitter`, 并且发生错误时的行为表现和 `EventEmitter` 一样.

`request` 流中发生的错误通过触发 `error` 事件来展示错误. **如果你没有监听该事件, 那么错误将会被抛出, 这会导致Node.js程序崩溃**. 因此你应该对所有的 `request` 流添加 `error` 事件处理程序, 即使你只是打印日志(确实可以这样做, 尽管最好的方式可能是发送一个HTTP错误的响应. 详情稍后会讲.)

```
request.on('error', (err) => {
  // This prints the error message and stack trace to `stderr`.
  console.error(err.stack);
});
```

也有其他处理这种错误的方式, 例如其他的抽象和工具, 但一定要意识到错误是可能发生的, 你必须要处理它们.

# 到目前为止我们获得了什么

到目前为止, 我们已经讲过了创建服务器, 获取请求类型, URL, 头部以及从请求中获取请求体. 当我们将这些内容组合到一起, 看起来就会像下面这样:

```
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // At this point, we have the headers, method, url and body, and can now
    // do whatever we need to in order to respond to this request.
  });
}).listen(8080); // Activates this server, listening on port 8080.
```

如果执行这个例子, 它便可以*接收*请求, 但无法*响应*. 实际上, 如果你在浏览器中访问它, 你的请求会超时, 因为没有任何响应回复给浏览器.

目前我们还没讲到 `response` 对象. `response` 对象是 `ServerResponse` 的实例, 同时也是一个 `WritableStream`. 它包含了许多有用的方法用于向浏览器回传数据. 接下来就会讲到它.

# HTTP状态码

如果你没有去设置, 响应的HTTP状态码总会是200. 当然, 并不是所有HTTP响应都应这样, 有时候需要明确地发送一个不同的状态码. 你可以使用 `statusCode` 来实现:

```
response.statusCode = 404; // Tell the client that the resource wasn't found.
```

也有其他更简便的方式来完成这个操作, 稍后我们会看到.

# 设置响应头

通过 `setHeader` 方法可以方便地设置响应头.

```
response.setHeader('Content-Type', 'application/json');
response.setHeader('X-Powered-By', 'bacon');
```

当设置响应头时, 头部名称无需区分大小写. 如果你重复设置了响应头, 那么生效的值是你最后设置的那个.

# 明确地发送头部数据

之前我们已经讨论过的设置头部和状态码的方法会假定你要使用"隐式头部". 这意味着在开始发送响应体之前, 你将指望Node.js在正确时间发送响应头.

如果需要的话, 你可以*明确地*将响应头写入到响应流里. 你需要使用 `writeHead` 方法实现这个操作, 它会把状态码和响应头写入流.

```
response.writeHead(200, {
  'Content-Type': 'application/json',
  'X-Powered-By': 'bacon'
});
```

一旦你设置了响应头(不论是隐式的还是明确的), 你就已经做好了发送响应数据的准备.

# 发送响应数据体

由于 `response` 对象是一个 `WritableStream`, 将响应数据体发送到客户端只需调用流对象的方法即可.

```
response.write('<html>');
response.write('<body>');
response.write('<h1>Hello, World!</h1>');
response.write('</body>');
response.write('</html>');
response.end();
```

上面的 `end()` 方法也可以接收可选的数据, 并将数据作为流数据的末尾数据发送出去, 因此上例可简化为:

```
response.end('<html><body><h1>Hello, World!</h1></body></html>');
```

> 注意: 在开始向响应体写入数据*之前*先设置状态码和响应头是很重要的. 这很容易理解, 因为在HTTP响应中响应头是在响应体前面的.

# 另一个关于错误处理的快速讨论

`response` 流也会触发 `error` 事件, 并且你也要去处理它. 所有关于 `request` 流的建议同样适用于 `response` 流.

# 将所有内容组合到一起

既然我们已经了解了如何使用HTTP响应, 那就组合到一起看一下. 基于之前的例子, 我们打算创建一个服务器, 它会把用户发送来的全部数据都发送回去. 我们会用 `JSON.stringify` 把数据格式化为JSON.

```
const http = require('http');

http.createServer((request, response) => {
  const { headers, method, url } = request;
  let body = [];
  request.on('error', (err) => {
    console.error(err);
  }).on('data', (chunk) => {
    body.push(chunk);
  }).on('end', () => {
    body = Buffer.concat(body).toString();
    // BEGINNING OF NEW STUFF

    response.on('error', (err) => {
      console.error(err);
    });

    response.statusCode = 200;
    response.setHeader('Content-Type', 'application/json');
    // Note: the 2 lines above could be replaced with this next one:
    // response.writeHead(200, {'Content-Type': 'application/json'})

    const responseBody = { headers, method, url, body };

    response.write(JSON.stringify(responseBody));
    response.end();
    // Note: the 2 lines above could be replaced with this next one:
    // response.end(JSON.stringify(responseBody))

    // END OF NEW STUFF
  });
}).listen(8080);
```

# 回显的服务器示例

让我们简化前面那个例子来实现一个简单的回显服务器, 它会把所收到的请求直接响应给浏览器. 我们需要做的是从请求流中抓取数据, 然后写入响应流, 和前面例子里类似.

```
const http = require('http');

http.createServer((request, response) => {
    let body = [];
    request.on('data', (chunk) => {
        body.push(chunk);
    }).on('end', () => {
        body = Buffer.concat(body).toString();
        response.end(body);
    });
}).listen(8080);
```

现在我们调整一下. 我们改为只在以下条件满足时才回显:

- 请求类型是POST.

- URL是 `/echo`.

在其他情况下, 只简单地响应404.

```
const http = require('http');

http.createServer((request, response) => {
  if (request.method === 'POST' && request.url === '/echo') {
    let body = [];
    request.on('data', (chunk) => {
      body.push(chunk);
    }).on('end', () => {
      body = Buffer.concat(body).toString();
      response.end(body);
    });
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

> 注意: 像上面那样通过检查URL来决定对应的操作, 我们实际是进行了"路由"操作. 其他形式的路由可以像 `switch` 状态判断那样简单, 也可以像 `express` 框架那样复杂. 如果你正在寻找一个只处理路由而不关心其他操作的东西, 可以试试 `router`.

真棒! 现在让我们尝试简化这个处理. 要记着, `request` 对象是个 `ReadableStream` , `response` 对象是个 `WritableStream`. 那也意味着我们可以用 `pipe` 把数据从一个对象传输另一个对象. 这正是我们想要的回显服务器!

```
const http = require('http');

http.createServer((request, response) => {
  if (request.method === 'POST' && request.url === '/echo') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

然而我们还没完全完成. 本文里多次提到的错误可能会发生, 我们需要处理它们.

为了处理请求流中的错误, 我们会把错误记录到 `stderr` 并响应一个400状态码以表明它是个`Bad Request`(出错的请求). 在真实的应用程序中, 我们需要检查错误并得出正确的状态码和错误信息. 像往常一样, 你应该查阅 `Error` 的<a href="https://nodejs.org/api/errors.html">文档</a>.

对于响应中的错误, 我们只是简单地将其记录到 `stderr`.

```
const http = require('http');

http.createServer((request, response) => {
  request.on('error', (err) => {
    console.error(err);
    response.statusCode = 400;
    response.end();
  });
  response.on('error', (err) => {
    console.error(err);
  });
  if (request.method === 'POST' && request.url === '/echo') {
    request.pipe(response);
  } else {
    response.statusCode = 404;
    response.end();
  }
}).listen(8080);
```

现在我们已经讲过了处理HTTP请求的大多数基本知识. 现在你应该可以完成:

- 实例化一个带有请求处理函数的HTTP服务器, 并使它监听一个端口.

- 从 `request` 对象中获取头部, URL, 请求类型和数据体.

- 基于 `request` 对象中的URL或其他数据进行路由决策.

- 通过 `response` 对象发送响应头, HTTP状态码和数据体.

- 用管道将数据从 `request` 对象传输到 `response` 对象.

- 处理请求流和响应流中的错误.

有了这些基础, 我们可以构建用于很多典型场景的Node.js HTTP服务器了. API文档中还提供了许多其他的资料, 所以一定要通读以下关于 `EventEmitter`, `Stream` 和 `HTTP` 的API文档.