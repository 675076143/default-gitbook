---
description: '2023-05-04'
---

# laravel-permission 缓存导致的Redis带宽占用问题

### 先决条件

spatie/laravel-permission: 3.18.0

权限数: 634

角色数: 94

### 问题

laravel-permission 的缓存机制设计的不太合理将所有的权限存放在一个大string中，且存放了许多不必要的信息。当项目拥有大量的角色权限，使得这个string达到了11mb，非常影响性能。

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

### 解决方案

spatie/laravel-permission: 4.3.0 及以上版本，这个问题得到了解决。然而我们不会贸然去选择升级依赖包的大版本，有风险，所以选择fork了一份到私有gitlab中，自行维护v3版本。并将对应的优化措施集成到 3.18.0 上，发布 3.19.0

优化后能够将大小缩减为原来的5%

<figure><img src="../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

出口流量速率由原来的140mb/s降到了6mb/s

<figure><img src="../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

### 集成

```json
{
    "repositories": [
        {
            "type": "gitlab",
            "url": "😝/packagist/laravel-permission"
        }
    ],
    "require": {
        "spatie/laravel-permission": "3.19.0"
    }
}
```

### 参考资料

{% embed url="https://github.com/spatie/laravel-permission/pull/1799" %}

\
