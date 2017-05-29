# 理解NodeJS中基于事件驱动的架构

对于绝大部分的nodejs中的对象，比如HTTP请求、相应以及'流',他们都是使用了`eventEmitter`模块的支持来监听和发射事件。

最简单的基于事件驱动的样子是一些流行的nodejs函数回调风格,例如：fs.readFile.在这个类比模型中，callback扮演着事件处理者的角色，当Node准备好调用callback的时候：事件将会被发射一次。

让我们来探究一下这个基础形式。

##### Node,在你准备好的时候调用我把！

在很久以前，当没有原生的promise、async/await特性支持的时候，Node最原始的处理异步的方式是使用callback。

callback函数基本上是你传给其他函数的函数，在JS中这是可能的，因为方法是第一级的对象。

理解`callback函数并不是在代码中被异步调用的`非常重要的，在函数中，我们可以随心所欲的同步/异步调用callback。

例如,在下面这种情况中，主函数fileSize包含一个回调函数cb,并且会在同步和异步的情况下调用这个回调函数

```js
function fileSize (fileName, cb) {
  if (typeof fileName !== 'string') {
    return cb(new TypeError('argument should be string')); // Sync
  }
  fs.stat(fileName, (err, stats) => {
    if (err) { return cb(err); } // Async
    cb(null, stats.size); // Async
  });
}
```

但是我们要记住，这种把主函数在处理callback的时候同时采用同步和异步的方式并不是一个好的实践，它也许会带来一些难以处理的错误。

我们再来看看下面这种典型的用callback风格处理的异步node函数：

```js
const readFileAsArray = function(file, cb) {
  fs.readFile(file, function(err, data) {
    if (err) {
      return cb(err);
    }
    const lines = data.toString().trim().split('\n');
    cb(null, lines);
  });
};
```

readFileAsArray有文件的路径和callback，它把文件读取并切割成一行一行的数组来当做参数调用callback

这里有一个使用他的实例，假设同目录下我们有一个numbers.txt文件中有如下内容:

```txt
10
11
12
13
14
15
```

我们想求这个文件中的基数之和，我们可以像下面这样调用readFileAsArray这个函数：

```js
readFileAsArray('./numbers.txt', (err, lines) => {
  if (err) throw err;
  const numbers = lines.map(Number);
  const oddNumbers = numbers.filter(n => n%2 === 1);
  console.log('Odd numbers count:', oddNumbers.length);
});
```

代码会读取数组中的字符串内容，解析成数字并求奇数之和。

在NodeJS的回调风格中的写法是这样的：第一个参数err代表着错误处理，当是null的时候我们就是正常调用callback,用户和作者都习惯了这样写，主函数接受callback作为最后一个参数并且调用的时候给callback第一个参数传递err对象。

##### 现代Javascript中callback的竞争者。

在高版本的JS中，我们有Promise对象，作为callback的有力竞争者，它的异步API替代了把callback作为一个参数传递并且同时处理错误信息，一个Promise对象允许我们分别处理成功和失败两种情况，并且链式的调用多个异步方法避免了回调地狱。

如果刚刚的readFileAsArray方法允许使用Promise，它的调用将是这个样子的：

```js
readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
```

作为调用callback的替代品，我们用.then函数来接受主方法的返回值；.then函数通常给我们和之前callback一样的参数，我们处理起来和以前一样，但是对于错误我们使用.catch函数来处理。

这要非常感谢新的Promise对象， 它让现代JavaScript中主函数支持Promise接口更加EZ，我们把刚刚的readFileAsArray方法用Promise来改写一下：

```js
const readFileAsArray = function(file, cb = () => {}) {
  return new Promise((resolve, reject) => {
    fs.readFile(file, function(err, data) {
      if (err) {
        reject(err);
        return cb(err);
      }
      const lines = data.toString().trim().split('\n');
      resolve(lines);
      cb(null, lines);
    });
  });
};
```

现在这个函数返回了一个Promise对象，这里面包含着`fs.readFile`这个异步调用，Promise对象中同时包含一个resolve函数和reject函数。

reject函数的作用就和我们之前callback中处理错误是一样的，而resolve函数也就和我们正常处理返回值是一样的。

我们剩下唯一要做的就是在实例中制定一个reject resolve函数的默认值，在Promise中，我们只要写一个空函数即可，例如\(\) =&gt; {}.

##### Promise的大敌：async/await函数。

当你需要一个常常的异步loop函数的时候，使用promise会让你coding的时候比callback的回调地狱简单一些，

Promise是一个小小的进步，generator也是一个小小的进步，但是async/await函数的到来，让这一步变得更有力了，它的编码风格让函数的可读性就像同步函数一样轻松。

我们用async/await函数特性来改写一下刚刚的调用readFileAsArray过程：

```js
async function countOdd () {
  try {
    const lines = await readFileAsArray('./numbers');
    const numbers = lines.map(Number);
    const oddCount = numbers.filter(n => n%2 === 1).length;
    console.log('Odd numbers count:', oddCount);
  } catch(err) {
    console.error(err);
  }
}
countOdd();
```

首先我们创建了一个async函数，只是在定义functiond的时候前面加了async关键字，在async函数里，我们把readFileAsArray这个异步方法用await来作为一个普通变量，这样我们的代码看起来真的是同步的呢！

当async函数执行的过程是非常易读的，处理错误，我们只需要使用try/catch即可。

在async/await函数中我们没有使用特殊API\(像: .then and .catch这种\)，我们仅仅使用了特殊关键字，但是像普通函数那样coding.

我们可以在支持Promise的函数中嵌套async/await函数，但是不能在callback风格的异步方法中使用它，比如setTimeout等等。

### EventEmitter模块 {#a418}

EventEmitter是NodeJS中基于事件驱动的架构的核心，它是促进了各个对象之间交流的模块，很多nodejs的原生模块都使用了这个模块。

关于概念这一块很简单，Emitter对象emit命名好的事件，使得之前注册好的监听器被调用起来，Emitter对象有两个显著特点：

* emit注册好的事件
* 注册和取消注册listener方法

如何使用呢？我们只需要穿件一个类来继承EventEmitter即可：

```
class MyEmitter extends EventEmitter {

}
```

然后我们创建一个基于EventEmitter的实例

```
const myEmitter = new MyEmitter();
```

有了实例，就可以在实例的全部生命周期里，emit任何我们命名好的方法了。

```
myEmitter.emit('something-happened');
```

emit一个事件就代表着有些情况的发生，这些情况通常是关于Emitter对象的状态改变的。

我们使用on方法来注册，然后这些监听的方法将会在每一个Emitter对象emit的时候执行

#### Events !== Asynchrony {#e554}

让我们看一个例子：

```js
const EventEmitter = require('events');

class WithLog extends EventEmitter {
  execute(taskFunc) {
    console.log('Before executing');
    this.emit('begin');
    taskFunc();
    this.emit('end');
    console.log('After executing');
  }
}

const withLog = new WithLog();

withLog.on('begin', () => console.log('About to execute'));
withLog.on('end', () => console.log('Done with execute'));

withLog.execute(() => console.log('*** Executing task ***'));
```

定义的WithLog类是一个event emitter.它有个方法excutej接受一个参数，并且有很多执行顺序的输出log,并且分别在开始和结束的时候emit了两次。

让我们来看看运行它会有什么样的结果：

```
Before executing
About to execute
*** Executing task ***
Done with execute
After executing
```

需要我们注意的是所有的输出log都是同步的，在代码里没有任何异步操作。

* 第一步‘’Before executing‘’
* 命名为begin的事件的emit导致了‘’About to execute‘’
* 内含方法的执行输出了“\*\*\* Executing task \*\*\*”
* 另一个命名事件输出“Done with execute”
* 最后“After executing”

就像之前的callback,我们在events中并没有假设同步或者异步的代码

这一点很重要，假如我们有异步代码，那么很多结果就会迥然不同

```js
// ...

withLog.execute(() => {
  setImmediate(() => {
    console.log('*** Executing task ***')
  });
});

// Now the output would be:

Before executing
About to execute
Done with execute
After executing
*** Executing task ***
```

这明显有问题，它的输出看起来不再精确了。

当异步方法结束的时候emit一个事件,我们需要把callback/promise与合并事件驱动的交流合并起来，刚刚的例子证明了这一点。

使用事件驱动来代替传统callback有一个好处是在定义多个listener后，我们可以多次对同一个emit做出反应。如果要用callback来做到这一点的话，我们需要些很多的逻辑在同一个callback中，事件是应用程序允许多个外部插件在应用程序核心之上构建功能的一个好方法，你可以把它们当作钩子点来允许围绕状态变化来做更多自定义的事。

##### 异步事件





































