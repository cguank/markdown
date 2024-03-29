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