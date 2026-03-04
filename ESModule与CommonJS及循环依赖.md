# ES Module 与 CommonJS 的区别及循环依赖处理

> 本文档说明 ES Module 和 CommonJS 的差异，以及二者在循环依赖场景下的行为差异。

---

## 一、ES Module 与 CommonJS 的区别

### 1. 语法与来源

| 维度 | ES Module (ESM) | CommonJS (CJS) |
|------|-----------------|----------------|
| **标准** | ECMAScript 标准，浏览器原生支持 | Node.js 历史方案，非 JS 标准 |
| **导入** | `import x from 'm'` / `import { a } from 'm'` | `const x = require('m')` |
| **导出** | `export default x` / `export { a, b }` | `module.exports = x` 或 `exports.a = a` |
| **加载时机** | 静态：解析阶段确定依赖，**先解析再执行** | 动态：**执行到 require 时才加载** |
| **顶层 await** | 支持 | 不支持 |
| **静态分析** | 可做 Tree Shaking、静态分析 | 难做静态分析 |

### 2. 加载与执行时机（核心差异）

- **ESM**：  
  - 先根据所有 `import` 建立模块图，确定加载顺序。  
  - 模块只**执行一次**，导出的是**绑定（live binding）**：导入方拿到的始终是当前模块内该变量的“引用”，值会随原模块内赋值变化。

- **CJS**：  
  - 执行到 `require('x')` 时才会去加载并执行模块 `x`。  
  - 导出的是**值的拷贝**（对原始类型是拷贝，对对象是引用拷贝）：`require` 得到的是当时 `module.exports` 的快照，后续原模块内再改 `exports.xxx` 不会影响已 require 到的对象（除非改的是对象内部属性）。

### 3. 导出的是“引用”还是“拷贝”

- **ESM**：导出的是与模块内变量**绑定**的引用，模块内变量变化，导入处看到的值也会变。
- **CJS**：导出的是 `module.exports` 的引用；若写 `exports.xxx = value`，导入方拿到的是对同一对象的引用；若写 `module.exports = 新对象`，则已执行过的 `require` 拿到的仍是旧对象。

---

## 二、循环依赖是什么

**循环依赖**：A 依赖 B，B 又依赖 A（或更长的环：A → B → C → A）。

- 在 ESM 中：依赖在**解析阶段**就确定，所以循环依赖在语法/静态结构上是允许的，关键看**执行顺序**和**使用时机**。
- 在 CJS 中：依赖在**执行阶段**按 `require` 调用顺序确定，循环依赖容易导致某个模块在**未执行完**时就被另一个模块 require，从而拿到**未完成初始化的** `module.exports`。

---

## 三、CommonJS 如何处理循环依赖

### 机制简述

1. 首次执行 `require('A')` 时，Node 会**先**在缓存里为 A 创建一个**空的** `module.exports` 对象，并标记 A 为“正在加载”。
2. 然后执行 A 的代码。若 A 里执行到 `require('B')`，则去加载 B。
3. 若 B 里又 `require('A')`，此时 A 已在“正在加载”状态，Node **不会重新执行 A**，而是**直接返回**当时缓存里那个**尚未执行完的**、可能是空的 `module.exports`。
4. 所以 B 拿到的 A 可能是“未初始化完成”的：A 里在 `require('B')` 之后的赋值还没执行。

### 简单例子

```js
// a.js
console.log('a 开始');
exports.done = false;
const b = require('./b.js');  // 这里会去执行 b.js，b 里又会 require('./a.js')
console.log('a 中 b.done =', b.done);
exports.done = true;
console.log('a 结束');

// b.js
console.log('b 开始');
exports.done = false;
const a = require('./a.js');  // 此时 a 未执行完，拿到的是 a 已导出部分的快照
console.log('b 中 a.done =', a.done);
exports.done = true;
console.log('b 结束');
```

**执行顺序与输出：**

1. 执行 a.js → 打印 "a 开始"，`a.done = false`，然后 `require('./b.js')`。
2. 执行 b.js → 打印 "b 开始"，`b.done = false`，然后 `require('./a.js')`。
3. 此时 a 未执行完，返回的是**当前** a 的 `module.exports`，即 `{ done: false }`。
4. b 中打印 `a.done` → **false**；b 执行完，`b.done = true`。
5. 回到 a，拿到 b 的导出，打印 `b.done` → **true**；a 执行完，`a.done = true`。

**结论**：CJS 通过“未完成导出的缓存”避免死循环，但**依赖方可能拿到未初始化完成**的模块，容易出 bug，需要靠代码结构（例如把互相依赖的访问放到函数里延后执行）来规避。

---

## 四、ES Module 如何处理循环依赖

### 机制简述

1. **解析阶段**：先扫描所有 `import`/`export`，建立模块依赖图，不执行模块体。
2. **加载与执行**：按依赖顺序加载并执行。遇到循环时，引擎会先为环上所有模块创建**导出环境**（即“槽位”），再按顺序执行各模块体。
3. 导出是 **live binding**：导入方始终读到的是模块内该变量**当前的值**。因此只要在**使用**时该变量已经赋值，就不会出现 CJS 那种“拿到未初始化导出”的问题；若在赋值前就使用，则可能得到 `undefined`。

### 简单例子

```js
// a.mjs
console.log('a 开始');
import { bDone } from './b.mjs';
export let aDone = false;
console.log('a 中 bDone =', bDone);  // 此时 b 可能还没执行到 export，可能是 undefined
aDone = true;
console.log('a 结束');

// b.mjs
console.log('b 开始');
import { aDone } from './a.mjs';
export let bDone = false;
console.log('b 中 aDone =', aDone);  // 若 a 已执行完 aDone = true，这里为 true；否则为 false
bDone = true;
console.log('b 结束');
```

- 执行顺序由引擎的模块图决定（例如先 a 再 b，或先 b 再 a，视具体实现而定）。
- 关键点：**读到的总是“当前绑定值”**。若 a 在 `aDone = true` 之后才被 b 读取，b 会看到 `true`；若 a 还没执行到赋值，b 会看到 `undefined`（或初始值）。

因此 ESM 的循环依赖**不会**像 CJS 那样给出一份“半成品”的静态快照，而是**总能看到最新状态**；是否出错取决于**谁在什么时候读**，若在对方未赋值前就读，就需要在代码里避免或做判空。

---

## 五、对比小结

| 项目 | CommonJS | ES Module |
|------|----------|-----------|
| **循环依赖** | 不报错，但可能拿到**未执行完**的导出（半成品） | 不报错，导出是 **live binding**，读到的是当前值 |
| **根本原因** | 执行时加载 + 导出是“当时快照” | 先建依赖图再执行 + 导出是“绑定” |
| **风险** | 容易在循环依赖中用到未初始化的导出 | 若在对方未赋值前使用，可能为 `undefined` |
| **建议** | 尽量避免循环依赖；若无法避免，把对循环模块的访问放到函数内延后执行 | 可容忍简单循环依赖；避免在模块顶层立即使用循环依赖的导出，可放到函数或异步中再使用 |

---

## 六、实践建议

1. **架构上**：尽量让依赖单向流动，减少循环依赖。
2. **CommonJS**：循环依赖时，不要依赖“尚未执行到的”导出；可把 `require` 或对对方导出的使用放到函数内部，延后到双方都执行完后再用。
3. **ES Module**：循环依赖下，避免在模块顶层立刻使用对方导出，可放到函数、`setTimeout` 或异步回调里使用，确保对方已执行完赋值。
4. **工具**：可用打包/分析工具（如 webpack、madge）检测循环依赖，便于重构。

---

*文档位置：`a-react-demo/docs/ESModule与CommonJS及循环依赖.md`*
