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

- 是什么：调用Native 提供的Bridge获取联系人列表。和传统的联系人不同，我们还会请求后端的接口，来获取每个联系人在我们系统的用户信息；另一块就是对手机号格式化加上国家号。这两块比较耗时，当用户通讯录达到3000条时往往会需要10+ s来加载，优化后本地能实现秒开。线上数据p80，大概是从5s多多降到了1.2s
![[Pasted image 20240329175138.png]]
- 怎么做：native采用缓存策略，返回缓存的联系人列表给前端。而我对格式化手机号做了两步优化。1. 原本是调用bridge的接口，这个接口在量大的时候会出错。经过了解他们也是用谷歌开源库做的格式化，我也就想调研下是否有JS版本的，叫做[libphonenumberjs](https://gitlab.com/catamphetamine/libphonenumber-js)，通过将接口调用转成纯JS代码后，优化从10+s，降到了3s左右。2.第二步优化就是类似分页，批量每100条处理一次，将控制权交给主线程，主线程空闲后继续批量处理。这次优化后，从3s降到了300ms
![[Pasted image 20240329181719.png]]
- 问题：
1. 为什么native不分页返回
	 最主要就是页面有搜索功能，而用户会通过国家号搜索不同国家的用户，那么这就必须对所有用户进行手机格式化给他们加上国家号。另外我们让PM和Locale调研用户通讯录一般不会超过2000条；
	 不过我们也有考虑进一步优化就是这一块格式化交给后端去做，让在查询用户信息的时候，后端顺便对手机进行格式话，因为不论是JS还是Native执行速度都跟用户手机性能强相关。
```ts
export function getNewFormattedContacts(
  contacts: RowItem[],
  startIndex: number
): Promise<{ list: RowItem[]; paginateTime: number; nextStartIndex: number }> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      try {
        const startTime = Date.now();
        const endIndex = Math.min(
          startIndex + getInitPhoneNumberCountAndStep(),
          contacts.length
        );
        const list = contacts.slice();
        for (let i = startIndex; i < endIndex; i += 1) {
          const rowItem = cloneDeep(contacts[i]);
          rowItem.displayUI.formattedPhoneNumber = getRowItemFormattedPhoneNumber(
            contacts[i]
          );
          rowItem.isPhoneNumberInit = true;
          list[i] = rowItem;
        }

        resolve({
          list,
          paginateTime: (Date.now() - startTime) / 1000,
          nextStartIndex: endIndex,
        });
      } catch (error) {
        RNLogger.reportError(
          ['ContactBridgeV2', 'paginate format'],
          (error as any)?.message || 'unknown error'
        );
        // eslint-disable-next-line prefer-promise-reject-errors
        reject({
          list: [],
          paginateTime: 0,
          nextStartIndex: 0,
        });
      }
    }, 0);
  });
}
```
### 3. RN遇到什么坑，动态模块，联系人入口，安卓是动态模块，IOS是预制的