# 一. 子应用是为何能和主应用在同一域名？
qiankun框架主应用通过子应用的entry.js来访问子应用，这样即使分开部署，但是只要主应用通过http请求子应用entry.js即可。

图中entry对象就会触发qiankun的匹配规则![[image/Pasted image 20220709162539.png]]
# 二. 子应用是怎么展示在主应用基座框架下的？
主应用在入口处调用了```qianku.registerMicroApps```
( [registerMicroApps](https://qiankun.umijs.org/zh/api#registermicroappsapps-lifecycles)), 其中子应用设置了

- activeRule（当配置为字符串时会直接跟 url 中的路径部分做前缀匹配，匹配成功表明当前应用会被激活）e.g. /adminv2/wallet
- 和container（微应用的容器节点的选择器或者 Element 实例）`#${appContainerId}`

主portal在入口处设置了路径匹配
![[image/Pasted image 20220709163048.png]]
子应用由于registerMicroApps设置了container，子应用路径/adminv2/xxx开头都会通过乾坤匹配到SubAppContainer这个组建，appContainerId这个结点内
![[image/Pasted image 20220709163128.png]]
至于Menu，在主应用中已经设置，里面包含了子应用的path
![[./image/Pasted image 20220709163147.png]]