## monorepo
## qiankun主应用和子应用
### MSS开发工具
![image](MSS-develop-tool.png) 本质上就是执行localStorage.setItem
设置后乾坤通过 localStorage.getItem() 来获得子应用的入口entry
```
localStorage.getItem('SHOPEE_DEV_TOOLS_APP_ENTRY')
'[{"appName":"shopee-pay","appEntry":"https://localhost:3000/"}]'
```
### partner portal
- 本地开发通过MSS开发工具，将http请求的子portal路径改成locahost由webpack生成的文件
- 部署在jenkins上配置，主要就是遵守配置规则
- 在partner portal主portal下也会设置shopeepay子portal的实际访问地址
- 【待确认】通过乾坤匹配子portal规则来访问相应的地址
- https://qiankun.umijs.org/zh/api#registermicroappsapps-lifecycles
- https://juejin.cn/post/6987296209807343623
### shopeepay portal

## duitnow check eligibility优化
## contact list 优化
2. 分析联系人页面痛点，完成了联系人页面的优化和重构：

#1 从用户体验方面，大幅缩短了RN侧多联系人情况，页面加载的时间从大于5s到3s（时间取决于联系人bridge的返回）；从页面滚动卡顿，优化到平滑滚动。

# 2 从开发体验方面，重构后的页面，最直观的感受就是index页面代码行数从700行到100行；定义联系人页面基础数据，拆分复杂逻辑，收拢分散的生成新联系人至一个工厂函数内等等。总之重构后，便于维护同时减少bug率。