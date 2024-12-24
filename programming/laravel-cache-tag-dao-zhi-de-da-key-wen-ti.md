---
description: '2024-05-30'
---

# Laravel Cache Tag 导致的大Key问题

### 问题说明

项目使用 Laravel 8近期发现 Redis Cache 中，有个 Set 元素有 11w个，占用了 24.88m

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

### 原因

这个 Set，是 Laravel Cache Tag功能, 用来存放缓存标签

{% embed url="https://laravel.com/docs/8.x/cache#storing-tagged-cache-items" %}

Set里大多的元素，都是由 laravel-passport 的创建的，这个行为本身没有问题。问题在于，当缓存标签对应的 **缓存本身** 过期时，缓存标签**永远不会被移除**。（除非清理所有缓存）

{% embed url="https://github.com/laravel/framework/issues/22981https:/github.com/laravel/framework/discussions/46175" %}

### 解决方案

Laravel 10.x 之后通过有序集合解决了这个问题： https://github.com/laravel/framework/pull/45690但升级总是伴随着风险，这个项目选择了个更保守的方式: 每天定时清理过期的Key

1. 通过 scan 命令扫描所有 standard\_ref

⚠️不能使用 keys 命令，这个可能导致命令执行耗时过长引发Redis阻塞

2. 遍历该set中的所有元素
3. 若元素对应的 redis key 已过期， 则删除该元素
4. Schedule 中每日执行这个清理命令

```php
<?php

namespace App\Console\Commands;

use Illuminate\Cache\RedisTaggedCache;
use Illuminate\Cache\TagSet;
use Illuminate\Console\Command;
use Illuminate\Redis\Connections\PhpRedisConnection;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;

class PurgeCacheTagCommand extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'cache:purge-tag';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Purge Expired Tags';

    /**
     * Create a new command instance.
     *
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }

    /**
     * Execute the console command.
     *
     */
    public function handle(): void
    {
        $prefix = config('database.redis.options.prefix');
        $key = $prefix. config('cache.prefix') .':*:'. RedisTaggedCache::REFERENCE_KEY_STANDARD;
        // function batch 用于分批清理，防止因过期元素过多导致unpack table失败
        $lua = <<<LUA
    local function batches(n, batchSize)
      local i = 0
      return function()
        local from = i * batchSize + 1
        i = i + 1
        if (from <= n) then
          local to = math.min(from + batchSize - 1, n)
          return from, to
        end
      end
    end

    local keys = redis.call('SMEMBERS', '%s')
    local expired = {}
    for i, key in ipairs(keys) do
    local ttl = redis.call('ttl', '%s'..key)
    if ttl == -2 then
        table.insert(expired, key)
    end
    end
    if #expired > 0 then
    for from, to in batches(#expired, 7000) do
        redis.call('SREM', '%s', unpack(expired, from, to))
    end
    end
    LUA;
        $redis = Redis::connection('cache');
        $start = microtime(true);
        $iterator = NULL;
        while($iterator !== 0) {
            // 游标查找
            [$iterator,$result] = $redis->scan($iterator, ['match' => $key]);
            $this->info('Scan cost: '.microtime(true) - $start);
            if(!$iterator) {
                $iterator = 0;
            }
            if(is_array($result)) {
                foreach($result as $tagKey) {
                    $this->info($tagKey);
                    $this->info($redis->eval(sprintf($lua,$tagKey, $prefix, $tagKey), 0));
                    $this->info('SREM cost: '.microtime(true) - $start);
                }
            }
        }
        $this->info('Redis Error: '.$redis->getLastError());
    }
}
```

```php
$schedule->command('cache:purge-tag')->dailyAt('00:00')->withoutOverlapping()->onOneServer()->runInBackground();
```

\
