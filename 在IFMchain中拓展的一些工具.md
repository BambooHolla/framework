## 使用原生async/await替代async库

我在IFMchain源码中，对底层原型链添加了一些方法。来让项目平滑地用上原生的async/await。

这里事先定义了一种规范，凡事`*Async`结尾的函数，就意味着是async函数或者是返回Promise的函数。

最核心的就是`Object.prototype.promise`。基于Proxy类，实现了对自身callback风格的代码转Promise风格的操作：

```js
// 原来的写法
instance.handleName(...args,function cb(err, res){
  // do some thing
});
// 转成Promise后的写法：
const res = await instance.promise.handleNameAsync(...args);
```

如果`instance`上已经有`handleNameAsync`方法，那么如果还是采用`.promise.handleNameAsync`写法，会提供警告：建议用户直接使用：`instance.handleNameAsync`。

### async库中的一些方法在原生中的替代写法
我就挑最常见的3个来说，基本也就是这三个在用。

#### async.waterfall
这个是最基本的，流程写法：
```js
async.waterfall(cb1, cb2)
// 使用await即可
await cb1Async();
await cb2Async();
```

#### async.eachSeries
这个方法是将一个数组的元素异步循环执行一些任务。简单的模拟如下：
```js
for await(let item of arr){
  // do something
}
// 但是for await属于比较高级的语法，目前好像就是浏览器级别的js支持。
// 所以目前应该使用下面的写法：
for(let item of arr){
  await item;
  // do something
}
```

#### async.series
这个是多个任务并发执行，用Promise.all来进行改写：
```js
await Promise.all(arr.map(async (item,i)=>{
  // do something
}))
```

> **async库还是是可取的，它提供了一些很好的API**，由于它自身也是callback风格的。所以也能直接用`await async.primise.XXX(...args)`来进行使用。

## promise兼容callback风格

有了callback转Promise，同时也要有Promise转callback，不然async函数必须一层套一层，全部用async的直接改动的工作量会很大。所以就提供一个简单的方法来让Promise转callback。

我考虑过三种写法：
1. callback.fromPromise(promise)
1. promise.toCallback()(callback)
1. promise.fromCallback(callback)

项目中目前采用第三种写法：`fromCallback(cb)`也有简写：`fromCb(cb)`

有了这个接口，如果你实现了一个原本callback风格的api的async改写后，比如：
```js
// 原本代码
class Instance{
  say(word, cb){
    cb(null, 'SAY:' + word);
  }
}
// 新版代码
class Instance{
  async sayAsync(word){
    return 'SAY:' + word;
  }
  // 使用fromCallback做兼容
  say(word, cb){
    this.sayAsync(word).fromCallback(cb)
  }
}
```

## 异步排序

源码位于`helpers/asyncSort.js`
这里的异步，是因为使用了多线程技术。基于[napajs](https://github.com/Microsoft/napajs)
来处理这种CPU密集型的动作。
比如1W条的排序，在主线程中可能需要2~3s，即便将这些排序改成主线程异步操作，还是有3s的CPU时间必须进行消耗，在主线程异步并不是好的做法。
所以我就另外开了一条进程来进行专门的排序操作。如果有需要同时排序的任务，可以另外再开一条进程。

目前内置了两个与业务关联比较紧密的API：

#### sortBigNumberByKeyAsync
大概的功能就是传入一个数组，这个数组里头的元素，是要基于某个属性进行排序的，这个属性是数字或者可转成bigNumber的字符串对象。

#### bubbleSortAsync
这个是代理人按照在线率排序的算法，是`modules/delegates.js`和`modules/accounts.js`中的需求。茂春说要提出来我就提出来了。

另外还有一个灵活性比较大的API：
#### customSortAsync
可以实现自定义排序规则，bubbleSortAsync就是基于这个接口进行实现的。

> 目前由于ShareArrayBuffer的支持不是很好，要在node9和10上才能用。所以暂时不支持并发排序。但后期等nodejs版本跟进，可以使用并发快排，来让这个排序API的速度超越主进程排序的速度。
