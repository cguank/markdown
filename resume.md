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
### shopeepay portal

## duitnow check eligibility优化
## contact list 优化