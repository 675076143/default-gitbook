---
description: '2024-11-21'
---

# 如何加载动态代码

### 背景

需求是通过动态的多条规则对一批数据做合法校验，并且能在多个模块间复用

这个规则不同于简单的关键词匹配，而是更为复杂的逻辑判断，所以需要抽象成各个函数执行, 数学表示为 f(x)

如果，直接将规则硬编码到程序里，对于后期维护、规则查询、复用都是个问题。

转而考虑将规则维护在表中，可供配置与展示，如下

| 规则表               |                                                    |             |
| ----------------- | -------------------------------------------------- | ----------- |
| 名称                | 实现(伪代码)                                            | 说明          |
| 包含A               | f(x) => x.includes(A)                              | 字符串查找       |
| 包含大于5%            | f(x) => x.match(/\[5-9]%/)                         | 正则匹配        |
| 包含2024且当前时间大于2024 | f(x) => x.includes(2024) && now().getYear() > 2024 | 借用预设函数now() |

### 技术实现(Nodejs)

仅仅只是把一串代码加载出来执行，这并不是什么难事，多数语言都支持 eval 特性。

但是会有很大的安全问题，试想如果在函数中写了个阻塞、删库、内存泄漏，甚至被注入攻击，这都会对服务造成很大的影响。

所以需要一个虚拟环境来运行。

[Nodejs 原生](https://nodejs.org/api/vm.html#vm-executing-javascript) 支持了一个虚拟环境，但官方也声明了该环境不是安全的

> The node:vm module is not a security mechanism. Do not use it to run untrusted code.

这边选用 [isolated-vm](https://github.com/laverdet/isolated-vm) 作为替代品

```javascript
const context = isolate.createContextSync();
const rules = [
  {
    name: "包含🎄",
    functionString: `x.includes('🎄')`,
    operator: "Robin",
    created_at: "2024-11-21 00:00:00",
    status: "enable",
  },
].map((i) => {
  // 编译可执行代码const hostile = isolate.compileScriptSync(i.functionString)
  return { ...i, function: hostile };
});
const inputs = ["🎄Christmas"];
const result = inputs.map((input) => {
  const row = {
    input,
    output: [],
  };
  rules.forEach((rule) => {
    // 切换上下文    context.global.setSync('x', input)
    // 执行    row.output.push(rule.function.runSync(context))
  });
  return row;
});

```

### 参考阅读

{% embed url="https://en.wikipedia.org/wiki/Lambda" %}

{% embed url="https://temporal.io/blog/intro-to-isolated-vm" %}

