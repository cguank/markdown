[流程图](https://excalidraw.com/#json=TdEqEhiIyJp0QRJtPynI6,cP5_hYrf6hA2AwUtvKeXEA)

## 1. 背景
## 2.画整体时序图
![[Pasted image 20250605111951.png]]
![[Pasted image 20250605112015.png]]
1. root/index入口文件会initRoutecenter，参数有routes
2. 初始化中会对routes中每一个item注册
this.registerPage(pageName, this.PageContainer, { nativeTabName });
this.PageContainer返回的是公共React组件LazyComponent
3. LazyComponent读取config
const config = routeCenter.getConfig(pageName)!;
如果页面已经加载过了则直接返回
4. 页面没有加载过，调用routeCenter.resolveGuardWithLoading
这里面会读取config.lazyload也就是routes里面每一项的lazyload。
5. lazyload会引入页面，页面通过pageContainer注册，其中会更新config.Component=页面Component
```ts
routeCenter.updateConfig({
      pageName,
      // @ts-expect-error fix in the future
      Component, // 页面组件
      beforeEnter,
      timeout,
    })
```
6. routecenter通过config.Component是否是函数来判断是否加载过页面




- 外部页面是怎么通过navigator跳转页面的，为何能命中未注册的页面
- 从外部页面跳，在beforenter中直接navigator另一个页面 且return false， 则页面不能正常跳转到B页面，会被routecenter pop掉。参考topupEntry，用户被ban了的错误处理（错误码100022）
### router init的时候会用LazyRouteComponent注册，在LazyRouteComponent中非内部跳转会调用AppRegistry.runApplication，这时候是怎么渲染页面的？（在哪里注册了页面）AppRegistry.runApplication这个是怎么运行的？为何页面没注册也能正常打开（把target路径给改成了未load的A路径页面，也能正常打开）
AppRegistry.runApplication 会走向A路径注册的LazyRouterComponent，在该LazyRouterCOmponent里会调用resolveGuardWithLoading，load A页面，routerCenter.updateConfig A路径的config
### 如何内部跨plugin通信？
基于Shopee框架提供的ComponentManager.getComponent，加载内部bundle提供的internalRouteCenter
### 为何要内部plugin通信，不是都可以使用外部跳转方式：push新页面，在新页面里渲染结果
1. 统一内部plugin的跳转。
2. 跳转分为内部和外部跳转，
- 如果外部跳转，会先跳转空白loading页再渲染页面
- 而内部会先在当前页面loading，等待目标页loaded，然后跳转
## 为何要用AppRegistry.runApplication，而不是直接返回目标的component
- 便于debug，为了改变目标页面的路径，否则A页面路径渲染的却是B页面的内容
### 整体架构
### init流程图
### LazyRouteComponent流程图

### 难点：暴露给外部plugin，组件丢失翻译如何解决：
- packages/page-container/src/pluginInitFactory.tsx，在这里面去初始化翻译、toggle、持久化缓存
* 在每个提供给外部使用的 react 组件外层包一个 PluginInit 组件，更改后的react组件层级结构如下：
```
<PluginInit>
 <BizReactComponent />
</PluginInit>
```
### 难点：外部页面跳转，怎么正确执行路由守卫逻辑
- 路由中心的跳转会先读取页面配置，拿到precheck，执行完后跳转目标页面；且会注入一个flag，用来识别是否内部跳转；
- 每个页面，我们都使用一个Shell包裹目标页面；Shell会去读取路由中心的相关配置。
### 难点：A页面的路由守卫返回B页面，要如何省去跳转动画（用户无感知）
- 如果A页面的路由守卫最终指向B页面，那么跳转到B页面会有一个500ms左右的跳转动画，如果B又指向C，则又会有一个跳转动画。用户体验不好。我们预期是，不论最终指向哪个页面，总是在当前页面渲染最终页面。
- 我们最终判断，如果finalPage和currentPage不一致则调AppRegistry.runApplication，该方法能够直接渲染目标页面。
### 懒加载过程能做的事
- 上报每个页面加载耗时
- 提供统一的注册页面文件，收拢了注册逻辑
### 优化流程
1. 路由守卫，进入页面前precheck，和上游页面节藕；优化用户体验
2. 代理跳转，跳转前先执行precheck（加载阶段注册precheck，执行阶段读取precheck）
3. 为了解决外部页面跳转，无法执行代理跳转，于是在注册阶段代理了页面