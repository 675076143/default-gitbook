---
description: '2023-04-26'
---

# Filtering系统与AST的应用

分享个我在项目里用ast的案例

restful api 经常会需要支持filter

```jsx
filter[id]=10

filter[type]=normal
```

代码里面设计了这样的一个东西:

```jsx
const allowedFilter = {

id: true,

type: true

}
```

如果需要默认值的话:

```jsx
const allowedFilter = {

id: true,

type: 'default:normal'

}
```

如果复杂查询的话，用函数:

```jsx
const allowedFilter = {

id: true,

type: (subQuery, value) => subQuery.where('type', value)

}
```

如果复杂查询，并且支持默认值的话:

```jsx
const allowedFilter = {

id: true,

type: (subQuery, value = 'normal') => subQuery.where('type', value)

}
```

那么这个filter就需要推断当前给定的函数，其参数是否包含默认值，即：

1. 输入filter\[type], 调用函数
2. 无输入filter\[type], 推断第二个参数value有默认值，调用函数
3. 无输入filter\[type], 推断第二个参数value无默认值，不调用函数

### 部分实现

```jsx
// 分词
function paramsHasDefaultValue(fnStr){
  const nameTokens = [...acorn.tokenizer(fnStr, {ecmaVersion: 2020})]
  let nameCount = 0;
  let step = 0;
  let hasEqualSympol = false;
  let hasValueSympol = false
  nameTokens.forEach(token => {
    if(token.type.label === 'name'){
      nameCount += 1;
    }
    if(nameCount === 2){
      step += 1
    }
    if(step === 2 && token.type.label === '='){
      hasEqualSympol = true;
    }
    if(step === 3 && ['string'].includes(token.type.label)){
      hasValueSympol = true
    }
  })
  return hasEqualSympol && hasValueSympol
}

// 应用
(allowedFilters, queryBuilder) => {
      Object.entries(allowedFilters).forEach(([k,v]) => {
        if(typeof v === 'function'){
          if(filter[k]) {
            v(queryBuilder, filter[k])
          }else{
            if(paramsHasDefaultValue(v)){
            v(queryBuilder)
            }
          }
        }
      })
}
```
