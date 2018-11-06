其实和Ionic一样，都是在封装一些东西。
Ionic做的是封装组件，bnqkl-framework封装的是实用工具。
因为这个项目一直在快速迭代开发改进中，所以目前以文件夹的形式存在，而不是一个独立项目。

接下来就是介绍这个framework到底包含了什么东西。

## Decorator.ts

这个文件，封装了一些与Ionic轻度耦合的**修饰器**：
```ts
export const asyncCtrlGenerator = {
  success: asyncSuccessWrapGenerator,
  loading: asyncLoadingWrapGenerator,
  error: asyncErrorWrapGenerator,
  retry: autoRetryWrapGenerator,
};
```
目的都是为了简化View Class的一些基本交互。
以上，都是的核心都是基于**Promise**。

我们从头构建一个表单提交的函数，一开始我们往往是这样写的：
```ts
class ViewPage{
	constructor(// 这里的public是简写，等于执行了 this.dataService = dataService
		public dataService: DataServiceProvider
	){}
	data = {key: "value"};
	submit(){
		this.dataService.doSomeThing(this.data, (err, res)=>{
			if(err){
				alert(err);
			}else{
				alert("success");
			}
		});
	}
}
```
然后我们来改进这段代码：
第一步，把传统的**js-callback**转化为**Promise**。

> 这里赘诉一下，在后端的代码中，可以时候`util.promisify`来将一个**js-callback**转成**Promise**风格的函数，或者使用[bluebird](https://www.npmjs.com/package/bluebird)相关的API还能做到批量转换。
为什么要把Promise看得那么重，**我们知道直接的callback效率是最高的这点毋庸置疑，而且以后Promise在怎么优化也无法超越callback的效率**。但是，Promise与callback的效率差异，在木桶原理上来说，仅仅是那几条长木条的关系而已。在前端，性能瓶颈在于DOM，在后端，性能的瓶颈在于IO。所以，除非是底层的API，否则，在业务层面，应该尽可能的使用Promise。

我们拿**asyncCtrlGenerator.success**来说，只要submit返回的是一个Promise规范对象，我们就能判断这个`submit`执行是否成功，如果成功，我们就执行这个修饰器后面的任务，也就是我们的`alert("success")`。
```ts
class ViewPage{
	//...
	@asyncCtrlGenerator.success("success")
	submit(){
		return this.dataService.doSomeThing(this.data);// 将Promise对象返回除去
	}
}
```
同样的，如果我们要`alert(err)`，只需要再加一个`error`修饰器：
```ts
class ViewPage{
	//...
	@asyncCtrlGenerator.success("success")
	@asyncCtrlGenerator.error("error Title")
	submit(){
		return this.dataService.doSomeThing(this.data);// 将Promise对象返回除去
	}
}
```

所以，其它的两个loading与retry修饰器的作用，也就不需要用代码来说明了。loading就是显示一个通用的弹出层，retry就是遇到异常的时候自动再次执行（这个用的比较少，是IBT2这个项目的特殊需求才加入的，因为分布式网络，万一有一个节点崩了，重试过多的时候，就要去寻找新的节点进行重连）。

### 自定义success、error、loading

这里是一个模板，可以作为参考：
```ts
class ViewPage{
	//...
	async submit(){
		this.submitting = true; // 开始loading
		try{
			await this.dataService.doSomeThing(this.data);
			console.log('success'); // 成功
		} catch(err) {
			console.error(err); // 失败
		} finally {
			this.submitting = false;// 结束loaidng
		}
	}
}
```

## FirstLevelPage.ts

这个可以说是整个项目的核心了，由`Tool,Lifecycle,Route,Form,Data`等这些工具组合而成，有的项目可能会多少增加一些文件，但无所谓，这些都是按照项目需求可进行额外增加的。

### FLP_Tool.ts

工具层，在`app.component.ts`中，有用依赖注入得到一些单利，并将其放置到全局。`@FLP_Tool.FromGlobal`修饰器将全局的对象以一种**不是很强势的方法**拿到手：
![image](https://user-images.githubusercontent.com/2151644/36541802-0c5077c4-181a-11e8-8e11-5ccae203e289.png)
如图红框中的代码，如果开发者依旧使用依赖注入，那么将以开发者依赖注入的对象优先，而不是全局的对象，从而确保了灵活性。
> PS：理论上我应该使用Symbol，但是ts毕竟是要编译到es6的语法，所以除非是后端的nodejs，前端这里我还是建议不要使用Symbol。

### FLP_Lifecycle.ts

生命周期，这个文件将Ionic与Angular的生命周期进行了封装。也就出现了贯穿整个项目的注册生命周期的写法：
```ts
class ViewPage{
	@ViewPage.willEnter
	doSomeThing() {

	}
}
```
另外，还提供了`autoUnsubscribe`修饰器，这个修饰器是服务于Rxjs的`Subscription`对象，在离开页面或者组件销毁的时候，自动解除监听。

### FLP_Route.ts

这个类可以看成是navigator。有几个核心的API需要介绍一下：

#### routeTo(path: string, params?: any, opts?: any, force = false)
页面跳转，之所以要另外做封装，是因为有些页面在进入之前，是需要做一些检测的。以本能理财项目为例，在进入买入卖出页面的时候，需要做大量的账户检查：实名认证检测、检测银行卡绑定、检测支付密码是否已经设置、检测委托扣款协议。
使用routeTo进行跳转的话，能确保这间检查都通过了在进入页面，如果没有检测通过，则可以执行指定任务，比如跳转到指定的页面执行对应的设置与表单填写。
![image](https://user-images.githubusercontent.com/2151644/36544204-d8e3f90e-1820-11e8-81b2-16a94295fa17.png)
如图中的红框中的代码，在`is_settd_pay_pwd`检测不通过的时候，进入到`set-pay-pwd`，在密码设置完成后，首先是`remove_view_after_finish`配置的触发，会自定关闭`set-pay-pwd`这个页面，然后执行了after_finish_job的任务。调度这两个配置的API，便是：`finishJob`。

#### finishJob(remove_view_after_finish?:boolean, time?:number)
有些页面，在手机上，可以说是用来针对执行某些任务的，就像桌面版本的弹出框一样。
继续`set-pay-pwd`这个页面的例子，在`setPayPWD`这个提交函数的最后，我们执行了`finishJob`，它会自动检测传入页面的参数，执行我们上面所说的那些配置所对应的任务。

#### jobRes(data: any)
用于辅助`finishJob`这个函数，将任务的数据返回到上一级页面中。上一级的接受方法如下：
![image](https://user-images.githubusercontent.com/2151644/36545222-67577808-1823-11e8-9c3f-1be4fc1ea79d.png)
设计接口的时候之所以没有将接受返回值的回调接口与`routeTo`接口合并，是为了保持分离所带来的书写遍历，因为routeTo接口往往是在html那边写的，那个地方指定回调什么的，写起来会很冗余麻烦。

### FLP_Form.ts
因为Angular的那套表单设计过于简单，官方文档只介绍了一些简单的用法，所以不得已，参考了官方的接口设计，实现了一套自主维护的API。
有这么几个字段是固定的：formData、errors、canSubmit、inputstatus
简单的看这个例子：
```ts
class ViewPage{
	formData = {
		username: "",
	};
	@AccountSetUsernamePage.setErrorTo("errors", "username", [
		"tooLong",
		"haveSpaces",
	])
	check_username() {
		const res: any = {};
		if (/\s/.test(this.formData.username)) {
			res.haveSpaces = true;
		}
		if (this.formData.username.length > 20) {
			res.tooLong = true;
		}
		return res;
	}
	submit(){
		// do something...
		this.resetFormData();// 默认会清空formData中所有的字符串类型的数据
	}
}
```
```html
<input [(ngModel)]="formData.username" set-input-status="username">
<button [disabled]="!canSubmit" (click)="submit()">提交</button>
```
`set-input-status`是一个`direcitive`，当用户输入内容的时候，根据配置的`username`这个参数，回去访问`setErrorTo`这个修饰器修饰的`username`检测器（第二个参数），并将检测的结果返回到`errors`这个字段中（第一个参数），可能的错误必须明确写明在第三个参数中（因为检测器可能有多个，每个检测器只检查部分字段，所以必须明确声明检查了哪些字段。但大部分情况下只有一个检测器，这个可改进）。
`canSubmit`默认是检测formData的字段是否都存在，errors是否会空。开发者可以针对某个页面自行重写这个属性。
`inputstatus`，是用来描述一个文本输入框的状态，一般是配合errors来实现表单错误的提示：
```html
<div class="error-tip" [ngClass]="inputstatus.phone" *ngIf="errors.phone?.noPhoneNumber">
  <ng-container *ngIf="inputstatus.phone=='blur'">
    手机号码有误
  </ng-container>
</div>
```
> 以上代码的意思就是如果用户离开了输入框，才显示错误，输入状态下不显示错误。


### FLP_Data.ts
这个模块提供了数据显示相关的服务：
#### `setAfterPageEnter`修饰器
页面进入有动画，如果列表也有动画的话，两个动画同时绘制，会导致渲染效率奇低，在低端Android手机上会出现掉帧的行为，所以`setAfterPageEnter`所修饰的数据，会在指定配置的页面进入后的一段时间后才将数据显示到页面上。等于延迟了渲染。
#### refreshShowList()
这个函数是默认会调用的，没什么性能消耗。作用是：如果页面上有**timeago**这类的时间显示，为了确保**timeago**的更新，可以在页面绑定数据的时候这样写：
```html
{{(YOUR_TIME_DATA+timeago_clock)|timeagoPipe}}
```


最后，再简单描述一下FirstLevelPage这个模块里头封装的一些东西。基本就是页面效果相关的：

比如页面滚动的时候，自动为title增加阴影。IBT2项目中的配色是偏亮的，所以我自行加了这个动态阴影来增加头部与内容直接的层次感。

还有就是本能理财APP中的头部模拟IOS的背景虚化功能。也是在这个模块中开了一个对页面content元素的控制，因为这个控制依赖于生命周期，所以不得不写在这里。

## SecondLevelPage.ts

这个页面就是提供了一个隐藏tabs的功能，ionic官方的tabs隐藏的配置有BUG，不该用，所以就自己写了一套。
