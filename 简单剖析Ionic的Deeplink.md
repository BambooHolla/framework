## 关于懒加载的编译与优化

不论是Ionic官方文档中还是命令行的创建一个新的页面，Ionic推荐的做法都是使用懒加载。
等于在编译后，应用启动的时候，只需要下载核心的main.js，其它的页面按需加载进来。Ionic官方也给出了一些配置可以配置页面的预加载以及配置页面的优先级。

看上去很美好，但是要注意的是！懒加载进来的页面，由于每个页面文件的独立性很高，由于是基于Webpack来编译的，所以每一个页面的编译方法都是：与main.js做对比，而后得出一个独立的页面文件中需要包含那些东西。
目前Ionic并没有做到多个文件之间自动对比出公用模块然后自动抽离成一个独立的辅助模块。

所以，务必要记住，如果有什么需要公用的模块，就直接在`app.module.ts`中进行`import`，就只要导入就行，这样公用的模块就直接编译到`main.js`中，这虽然会稍微拖慢应用的启动速度，但总比每一个页面都要去再次消耗CPU去解析这些重复的代码要好很多。

## 懒加载与直接加载的差异
> 这一小节的内容是我基于**官方的Demo**和**个人项目经验**以及**Ionic-app-scripts项目的源码**总结得出的，并不直接代表官方内容，如果有错，欢迎修改。

从代码与文件结构上来看简单的来看：
### 懒加载
有一个`page.module.ts`，并且`page.ts`中有`@IonicPage({ name: "pagename" })`的配置。
### 直接加载
没有上面懒加载提及的两点以外，需要在`app.module.ts`中导入页面模块，然后在**declarations**与**entryComponents**中也都要添加这个页面模块。
如果要自定义links，就像`@IonicPage`做的那样，就还要在的`IonicModule.forRoot`的第三个参数中，手动配置`links:[{ component: TabsPage, name: "tabs" }]`。

最终结果就是直接加载的页面会被硬编译到main.js中。打开这些页面的速度明显会快于懒加载页面的速度。特别是Android手机上，这种优势尤为明显。

### 混合加载
二者在简单的情况下是可以同时存在的。所谓简单情况，就是**无自定义links**，就是`IonicModule.forRoot`的第三个参数是空的情况。
当我们要自定义**deepLink**，就不得不配置这第三个参数，但如果配置第三个参数，Ionic的编译工具就无法编译出那些需要懒加载的页面，这是矛盾的！
> 具体的实现可以看[ionic-app-scripts](https://github.com/ionic-team/ionic-app-scripts)项目源码的[实现](https://github.com/ionic-team/ionic-app-scripts/blob/master/src/deep-linking/util.ts)，我这里大概解释一下如何实现Deeplink注入的：查找名为`IonicModule.forRoot`的调用，检测调用传入的参数，如果没有第三参数就进行自动生成并注入到生成的代码中。

所以我们要实现就是，在不传入这第三个参数的情况下，让Ionic检测没有这第三参数，从而自动生成代码并注入第三参数，我们再混入我们的自定义deeplink。
传统的Jser的思路做法，往往是重写forRoot这个函数。嗯，在开发模式下，这当然是有效的。但是AOT编译，是不允许有这种骚操作的。

NgModule的作用，就是要开发者明明白白把控制反转（就是上面说的骚操作）写到代码中，所以我们不得不回到Ionic源码，看到底是什么在使用这个DeeplinkConfig，并将Deeplink指向对应的页面模块。

在[Ionic源码的module模块中](https://github.com/ionic-team/ionic/blob/master/src/module.ts)，我们搜索`DeepLinkConfigToken`，可以看到找出下面三行代码：
```
1. { provide: DeepLinkConfigToken, useValue: deepLinkConfig },
2. { provide: APP_INITIALIZER, useFactory: setupPreloading, deps: [ Config, DeepLinkConfigToken, ModuleLoader, NgZone ], multi: true },
3. { provide: UrlSerializer, useFactory: setupUrlSerializer, deps: [ App, DeepLinkConfigToken ] },
```
第一行，就是上面所提及的Ionic自动生成的*第三参数*。将之赋给`DeepLinkConfigToken`这个`provide`。
第二行，就是预加载，我们可以看到通过依赖注入`DeepLinkConfigToken`，`setupPreloading`将`deepLinkConfig`中属于延迟加载的页面，通过配置执行了异步加载，也就是我们所谓的懒加载。
第三行，就是最重要的：将页面的构造函数与我们自定义的name进行挂钩关联。

也就是在第三行，我们需要利用Angular官方的方法，对这个`UrlSerializer`进行覆盖重写，
打开我们的`app.module.ts`，在`providers`中，添加两个对象：
```ts
//providers
    {
      provide: MyDeepLinkConfigToken,
      useFactory: customDeepLinkConfig,
      deps: [DeepLinkConfigToken],
    },
    {
      provide: UrlSerializer,
      useFactory: setupUrlSerializer,
      deps: [App, MyDeepLinkConfigToken],
    },
```

第一个对象，就是获取Ionic自己生成的`deepLinkConfig`，并将之与我们的自定义deepLinkConfig进行混合：
```ts
export function customDeepLinkConfig(deepLinkConfig) {
  const static_links = [
    { component: TutorialPage, name: "tutorial" },
    { component: TabsPage, name: "tabs" },
    // 静态页面的deepLinkConfig.links
  ];
  if (deepLinkConfig && deepLinkConfig.links) {
    const static_links_name_set = new Set(static_links.map(link => link.name));
    deepLinkConfig.links = deepLinkConfig.links.filter(
      link => !static_links_name_set.has(link.name as string),
    );
    deepLinkConfig.links.push(...static_links);
  }
  return deepLinkConfig;
}
```

第二个对象，就是覆盖操作，使用我们自己的`MyDeepLinkConfigToken`来生成`UrlSerializer`。

上面的代码，涉及到一些依赖，可以看IBT2的源码看依赖从哪里导入，也可以直接看[Ionic的module.ts文件](https://github.com/ionic-team/ionic/blob/master/src/module.ts)。
