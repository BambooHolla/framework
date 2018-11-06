## 无第三方框架实现可控的DOM动画

_往常我们的做法，就会使用类似JQ的写法：获取一个对象，对其属性进行操作，把一堆CSS属性写在JS中，用JS存储状态去控制CSS。_

这里我介绍另外一种思路：**依旧把动画状态使用CSS keyframes来写，然后js只用来控制进度**。

CSS动画属性中，有一个可以让CSS动画暂停的属性：`animation-play-state: paused`;。使用这个属性，配合`animation-duraction：{number}s`，还有最重要的：`animation-delay：-{number}s`，来实现对动画显示状态控制。因为在`paused`的状态下，动画会默认显示第一帧，所以当我们改变delay属性，使它为负值，这样就能让动画显示成未来时间的效果。
所以我们只需要让这个delay属性无限接近负duraction，（**注意不是等于，当两个值想加为0的时候，就等于回到了第一帧**），动画的状态就会无限接近我们的最终效果。

使用CSS的好处是简单，不需要消耗js那条进程来计算数值、存储状态。同时，CSS动画基本能满足我们大部分需求，而且开发速度、可读性会快很多。
```scss
.ani-root {
	animation-name: root-animation;
	/* 启用头部动画*/
	animation-fill-mode: forwards;
	animation-play-state: paused;
	animation-duration: 2s;
	animation-timing-function: linear;
	
	/* 子元素都能跟随父级元素的动画而进行动画，从而实现多个元素同时联动*/
	* {
		animation-iteration-count: inherit;
		animation-fill-mode: inherit;
		animation-play-state: inherit;
		animation-duration: inherit;
		animation-timing-function: inherit;
		animation-delay: inherit;
	}

	.child-a{
		animation-name: a-animation;
		.child-a-a{
			animation-name: a-a-animation;
		}
	}
	.child-b{
		animation-name: b-animation;
	}
}
```
如上代码，就是核心的CSS代码，其中最重要的就是`animation-play-state: paused;`以及`animation-delay: inherit;`，因为我们是使用控制父级元素`.ani-root`的`animation-delay`来进行动画的，所以子元素的`animation-delay`必须继承父元素，从而才能实现多个元素同时运动的效果。

至于JS，就没什么难点，写一些参考代码：
```ts
const root_ani = document.querySelector('.ani-root');
const ani_second = parseFloat( getComputedStyle(root_ani).animationDuration||"0s" );
// animationDuration:不论代码中写的是ms还是s，都会被css引擎转换为s做单位。

export function ani(progress:number/* 0 ~ 1 */) {
  // TODO：做一些参数检查与缓存优化
  let  ani_delay = - progress * ani_second;
  if(progress === 1){
    ani_delay += 0.00001;// 只能逼近，不能相等
  }
  root_ani.style.animationDelay = ani_delay + "s";
}
```

#### safari对应的BUG解决方案

这里还有一个很让人暴躁的问题，就是**高版本safari的BUG**。safari的BUG往往是来自它们的工程师的优化比。我猜是safari会把DOM的动画硬编码成它们自己改进版本的底层编码，这个过程中导致了动态修改animation的一些属性没法实时生效。所以我的做法是把这个DOM移除出父级然后又马上塞回来，这样的DOM操作往往能迫使safari释放掉一些东西。
```ts
const _fuck_ios_bug_placeholder_ele = document.createComment();

const parent = ani_ele.parentElement as HTMLElement;
parent.replaceChild(_fuck_ios_bug_placeholder_ele, ani_ele);
parent.replaceChild(ani_ele, _fuck_ios_bug_placeholder_ele);
```
事实证明确实可以处理一些简单的情况，但是并不能解决全部。
实际的项目中往往会有图层的混合绘制，比如ios的滚动优化带来的问题，我的做法是在touchstart和touchend的时候再调用一次上面的代码，确保重绘完后再进行滚动。
所以类似的，你可能也会遇到IOS自己的优化带来的其它问题，**找到一个正确的触发点**，强制重绘一下可能会解决问题。

或者，也可以用代价更小的reflow方案来针对处理：
```ts
ele.style.animation = 'none';
ele.offsetHeight; //reflow
ele.style.animation = null;
```

## 使用双scale模拟height/width变动

在动画中，最忌讳的就是去变动元素的布局，在手机上，多层次的结构中，路若改变一个父级的宽高，有可能就是要花上200ms+的时间去重新绘制这一帧。
所以，在特定情况下，我们可以使用Transform来替代layout变动。当然，代价是肯定有的，就是要多写很多代码，来模拟这个元素的子元素的动画，从而来替代height变动所带来的布局重计算。

举个例子，我们要模拟变动height，缩小1倍，使用一个Transform，很容易，就是`scale(1, n)`，但是，会导致这里头的内容会被压扁，文字不可读。
所以就引出了我们要做的：双Transform:scale来让让内部元素保持可读性的操作，但是要注意的是，双Transform并不是简单的在`0% scale 1`，在`100% scale 2`与`100% scale 1/2`，看以下这两条曲线：
![image](https://user-images.githubusercontent.com/2151644/37250267-af306758-2533-11e8-8302-4447690b9d65.png)

> TODO...
