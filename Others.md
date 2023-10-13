# git bisect 二分查找错误代码
https://www.ruanyifeng.com/blog/2018/12/git-bisect.html
# js string的存储方式
https://www.cnblogs.com/goloving/p/15352261.html
# 登录鉴权
https://juejin.cn/post/7129298214959710244
### 3. 1 access token 和 refresh token
https://juejin.cn/post/6859572307505971213
**为何access token实效不设置长一点，只用一个token就够了**
- refresh token 是本地储存的，发出去的是 access token. 所以 access token 容易被通讯拦截。refresh token只有在更新access token的时候才会被发出去，暴露的频率低。
- access token是无状态的，过期时间短，则被盗用后只能在短时间内有效。当用refresh token去刷新，就会被服务器判断为双双失效（例如发现在黑名单内）
## 学习曲线
https://github.com/kamranahmedse/developer-roadmap