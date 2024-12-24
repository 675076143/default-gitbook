---
description: '2024-12-05'
---

# Redis出口流量分析与压缩处理

### 现象说明

从2024-11-25开始到今天，一直有redis流量告警，阿里云面板也没有发现热key或符合条件的大key，猜测是不满足标准

{% embed url="https://help.aliyun.com/zh/redis/user-guide/identify-and-handle-large-keys-and-hotkeys" %}

在不考虑尖峰的情况下，出口流量基本跟请求量匹配，考虑并不是因为某个版本业务需求导致在09:00-12:00，13:30-18:00这两个时间段，基本占用都处于60%\~80%之间 （最大带宽36MB/s）并且qps，cpu，延迟等各项指标都很健康

虽然告警是从2024-11-25出现的，但通过3周的环比数据看，没有明显的增长，可以认为就是业务量的上涨导致的带宽不足

### 单个请求的流量占用

这边本地准备了一个grafana, 挑选了几个高频接口用于测试，流量开销基本都在1m

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

这个流量主要是因为每次请求都需要完整读取laravel-permission

TODO:: 引用文档

这个大key虽然已经过了一轮处理，但是随着系统增加的权限越来越多还是无法避免。。。截止到当前，系统总权限数量有**1031**个，占用**1.65mb**, 妥妥的大key，可恨的是，几乎每个请求，都需要读取这个缓存。

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

### 针对permissions的压缩

选用cpu等各项指标都十分优秀的lz4算法，**单独**对permission缓存做压缩phpredis扩展支持加载该算法，需要在docker容器里做进一步配置

{% embed url="https://github.com/phpredis/phpredis?tab=readme-ov-file#session-compression" %}

{% embed url="https://laravel.com/docs/8.x/redis#phpredis-serialization" %}

注意这个配置是在laravel8才支持的，如果低于8可以考虑单独抽取这个pr

{% embed url="https://github.com/laravel/framework/pull/40282/files#diff-6a8e18e894ab849a0ff821f0a5f2127282c9c1fbe0ae3de7c17745ec7f4a0906" %}

```dockerfile
RUN apk add lz4-dev
RUN pecl install -o -f -D 'enable-redis-igbinary="yes" enable-redis-lz4="yes"' igbinary redis \
 && docker-php-ext-enable igbinary redis 
```

项目里修改三个文件config/database.php 新增redis链接，配置压缩选项为lz4

```php
<?php

return [
    'connections' => [
      'redis' => [
        'cache-compress' => [
            'url' => env('REDIS_CACHE_URL'),
            'host' => env('REDIS_CACHE_HOST', '127.0.0.1'),
            'password' => env('REDIS_CACHE_PASSWORD', null),
            'port' => env('REDIS_CACHE_PORT', 6379),
            'database' => env('REDIS_CACHE_COMPRESSION_DB', 2),
            'timeout' => 8 * 3600,
            'read_timeout' => 8 * 3600,
            'read_write_timeout' => 8 * 3600,
            'persistent' => true,
            'options' => [
//                'serializer' => 2, //Redis::SERIALIZER_IGBINARY,
                'compression' => 3, // Redis::COMPRESSION_LZ4
            ]
        ],
    ]
];
```

config/cache.php 增加一个store, 并指定redis链接

```php
<?php

return [   
    'stores' => [
        'redis-compress' => [
            'driver' => 'redis',
            'connection' => 'cache-compress',
        ],
    ],

];
```

config/permission.php 修改缓存store为redis-compress

```php
<?php
return [
    'cache' => [
        // 'store' => 'default',
        'store' => 'redis-compress',
    ],
];
```

### 最终效果

**1.65mb** -> **200.96kb**

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

单次请求的流量占用也缩小到了100kb+

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

### 参考文档

{% embed url="https://tech.meituan.com/2021/01/07/pack-gzip-zstd-lz4.html" %}

{% embed url="https://deliveroo.engineering/2016/10/07/optimising-session-key-storage.html" %}

{% embed url="https://github.com/laravel/framework/pull/40282/files#diff-6a8e18e894ab849a0ff821f0a5f2127282c9c1fbe0ae3de7c17745ec7f4a0906" %}

{% embed url="https://github.com/laravel/framework/pull/47531" %}

\
\
