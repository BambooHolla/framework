我简化一个实际中的需求来说：有一个固有间隔更新的排名列表`allRankList`，在更新的时候，需要进行大量计算而后排序的任务，客户端根据这个固有的时间间隔来获取数据。如何确保在这个间隔时间点“发起的请求”得到的数据就是准确的数据？

实现很简单，如下源码：
```js
class RankListSerive{
//...

    get allRankListPromise(){
        if (!this._allRankList) {
            // _initAllRankListAsync 意味着应用重启的时候，从数据库取数据来初始化
            this._allRankList = this._initAllRankListAsync();
        }
        return Promise.resolve(this._allRankList);
    };

//...
}
```
首先是定义了这个allRankListPromise对象，最后的那行：`Promise.resolve(this._allRankList)`就意味着，这个`this._allRankList`可以是一个数组、也能是一个Promise对象。
因此在使用这个对象的时候，需要这样调用：
```js
const allRankList = await this.allRankListPromise;
```
接下来就是重点了，如何更新这个对象：
```js
class RankListSerive{
//...
    /**
     * 将上一轮的rankList存入内存
     * @param {Array|Promise<Array>|PromiseLike<Array>} rankList
     */
    setAllRankList(rankList){
        this._allRankList = rankList;
    }
    async calcRankListAsync(round){
        const allRankListPromise = new PromiseOut();
        try {
            this.setAllRankList(allRankListPromise.promise);
            // await 计算、排序 任务
            allRankListPromise.resolve(allRankList);
            
            // 写入数据库，确保应用重启的时候能拿到这个数据
        }catch(err){
            allRankListPromise.reject(err);
            throw err;
        }
    }
//...
```
以上代码，意思是在在开始计算任务前，使用setAllRankList接口，让allRankListPromise的重置成一个新的Promise对象，并在所有计算任务结束后才得到resolve。
这样，**在这个任务期间，客户端所有的请求，或者所有依赖于allRankListPromise的任务，都会被这个`await allRankListPromise`所暂停**，并在其之被resolve后才返回，这样就确保了顺序问题。

----

基于这种模式，**可以很明确的声明每一个任务所依赖的其它任务**。从而能轻松地构建依赖关系网。充分利用异步带来的好处。
