#### 1.promise串行执行
```text
//创建方法是，不能传入Promise作为参数，因为在创建Promise的一刻它就开始pending等待处理，因此应该传入一系列的resolve
//等上一个 Promise resolved 之后再创建新的 Promise

function promiseQueue (executors) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(executors)) { executors = Array.from(executors) }
    if (executors.length <= 0) { return resolve([]) }

    var res = []
    executors = executors.map((x, i) => () => {
      var p = typeof x === 'function' ? new Promise(x) : Promise.resolve(x)
      p.then(response => {
        res[i] = response
        if (i === executors.length - 1) {
          resolve(res)
        } else {
          executors[i + 1]()
        }
      }, reject)
    })
    executors[0]()
  })
}

function alarm (msg, time) {
  return resolve => {
    setTimeout(() => {
      console.log(msg)
      resolve(msg)
    }, time)
  }
}

promiseQueue([alarm('1', 4000), alarm('2', 1000), 12, 'hellp', alarm('3', 3000)]).then(x => console.log(x))

```
#### 2.Promise实例的异步方法和then()中返回promise的区别
```text
// p1异步方法中返回p2
let p1 = new Promise ( (resolve, reject) => {
    resolve(p2)
} )
let p2 = new Promise ( ... )

// then()中返回promise
let p3 = new Promise ( (resolve, reject) => {
    resolve()
} )
let p4 = new Promise ( ... )
p3.then(
    () => return p4
)

```
p1的状态取决于p2，如果p2为pending，p1将等待p2状态的改变，p2的状态一旦改变，p1将会立即执行自己对应的回调，即then()中的方法针对的依然是p1

由于then()本身就会返回一个新的promise，所以后一个then()针对的永远是一个新的promise，但是像上面代码中我们自己手动返回p4，那么我们就可以在返回的promise中再次通过 resolve() 和 reject() 来改变状态

#### 3.express中间件机制
约定 中间件一定是一个函数并且接受 request, response, next三个参数
```text
function App() {
    if (!(this instanceof App))
        return new App();
    this.init();
}
App.prototype = {
    constructor: App,
    init: function() {
        this.request = { //模拟的request
            requestLine: 'POST /iven_ HTTP/1.1',
            headers: 'Host:www.baidu.com\r\nCookie:BAIDUID=E063E9B2690116090FE24E01ACDDF4AD:FG=1;BD_HOME=0',
            requestBody: 'key1=value1&key2=value2&key3=value3',
        };
        this.response = {}; //模拟的response
        this.chain = []; //存放中间件的一个数组
        this.index = 0; //当前执行的中间件在chain中的位置
    },
    use: function(handle) { //这里默认 handle 是函数，并且这里不做判断
        this.chain.push(handle);
    },
    next: function() { //当调用next时执行index所指向的中间件
        if (this.index >= this.chain.length)
            return;
        let middleware = this.chain[this.index];
        this.index++;
        middleware(this.request, this.response, this.next.bind(this));
    },
}
```
对request处理的中间件
```text
 function lineParser(req, res, next) {
        let items = req.requestLine.split(' ');
        req.methond = items[0];
        req.url = items[1];
        req.version = items[2];
        next(); //执行下一个中间件
    }

function headersParser(req, res, next) {
    let items = req.headers.split('\r\n');
    let header = {}
    for(let i in items) {
        let item = items[i].split(':');
        let key = item[0];
        let value = item[1];
        header[key] = value;
    }
    req.header = header;
    next(); //执行下一个中间件
}

function bodyParser(req, res, next) {
    let bodyStr = req.requestBody;
    let body = {};
    let items = bodyStr.split('&');
    for(let i in items) {
        let item = items[i].split('=');
        let key = item[0];
        let value = item[1];
        body[key] = value;
    }
    req.body = body;
    next(); //执行下一个中间件
}

function middleware3(req, res, next) {
    console.log('url: '+req.url);
    console.log('methond: '+req.methond);
    console.log('version: '+req.version);
    console.log(req.body);
    console.log(req.header);
    next(); //执行下一个中间件
}


```
