# Laravel Schedule 优雅退出实现

### 问题背景

laravel定时任务重新部署，如果涉及正在运行的任务，会被强行终止，只能等待下一次执行。

并且如果使用了 `withoutOverlapping()` / `onOneServer()` 这类的锁，无法正常释放，下次即便到点也无法派发任务，需要等待锁自动过期后，下一次才执行。

### Cron 退出并等待剩余php进程

laravel定时任务的本质，是依赖crontab去每分钟执行一个`schedule:run`

```bash
* * * * *  /usr/local/bin/php /var/www/backend/artisan schedule:run

```

在项目里，docker 启动命令是将定时任务作为前台运行

```docker
CMD ["crond","-f"]

```

k8s发送停止命令，容器会被退出，正在执行的任务也被掐掉了，只能等待下次执行，如果是每日执行一次的任务，影响就很大了。

k8s优雅退出的机制是，先发送 `STOPSIGNAL`, 等待30s(terminationGracePeriodSeconds), 如果还没停，则发送 SIGKILL

并且宽限时间如果设定为超过1分钟，则cron会再次调用 schedule:run , 这个也是不合理的。

也就是程序需要自行处理SIGTERM，终止Cron继续调用 `schedule:run` 派发新的任务

然后等待所有PHP进程全部结束，主动退出

则：

1. PID1 不能为cron, 需要变为后台运行 (cron日志需要打到stdout上)
2. 检测 SIGTERM 信号，kill cron
3. 检测是否有正在执行的php, 无则退出

完整bash如下

```bash
#!/bin/sh# 0-正常运行, 1-Cron退出, 2-PHP任务全部执行完成
status=0

function stop() {
    echo "stopping crond...."
    kill $(ps aux | grep '[c]rond' | awk '{print $1}')
    status=1
}

trap stop SIGQUIT SIGINT SIGTERM

crond

while true; do
    if [ $status -eq 1 ]; then
## 判断是否有剩余的任务## 进程特征如下## sh -c ('/usr/local/bin/php' 'artisan' test:test > '/dev/null' 2>&1 ;
        running=$(ps aux | grep '[s]h -c' -c)
        echo "running process: $running"
        if [ $running -lt 1 ]; then
            echo "program exit"
            exit 0
        fi
    fi
    sleep 1
done

```

验证

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### 子任务的优雅退出

仅仅是等待php任务执行完成还是不够，如果任务本身就是耗时，还是会因为超过宽限时间而被强制终止。

这就需要各个任务自行消化 SIGTERM 信号，主动退出。

如果是laravel9以上，可以用现成的 trap 函数

[https://laravel.com/docs/11.x/artisan#signal-handling](https://laravel.com/docs/11.x/artisan#signal-handling)

这边给出laravel8及以下版本的解法

```php
class Test extends Command
{
    protected bool $shouldKeepRunning = true;

public function handle(): void
    {
        // 收到SIGTERM，标记为终止
        pcntl_signal(SIGTERM, function () {
            $this->shouldKeepRunning = false;
        });
        $start = microtime(true);
        while (true) { // 长耗时任务，一般伴随循环处理
            sleep(1);
            $this->info('Keep running: '.microtime(true) - $start);
            if(!$this->shouldKeepRunning) {
                $this->info('保存进度');
                exit(); // 主动退出
            }
        }
    }
}

```

由于 k8s 发送的 `STOPSIGNAL` 只会给到 pid为1的进程，并且 laravel schedule 常伴随 `runInBackground()` 使用，这会导致派生的子进程收不到 `STOPSIGNAL`

前面的 sh脚本 改进一下, 当 kill crond 后，主动对 php 任务发送kill信号，注意不能直接kill到 schedule:run

这里简单解释一下 laravel schedule:run 执行后会发生什么

```bash
$schedule->command('test')->everyMinute()->runInBackground();

```

上面这条任务，执行后，会生成两个进程

一个是 schedule 进程，用于跟踪真正执行的任务，任务完成后，执行对应任务的schedule:finish

```bash
sh -c ('/usr/local/bin/php' 'artisan' test > '/dev/null' 2>&1 ; '/usr/local/bin/php' 'artisan' schedule:finish "framework/schedule-***" "$?") > '/dev/null' 2>&1 &

```

另一个是真正的任务进程

```bash
/usr/local/bin/php artisan test
```

要杀的是真正的任务进程，而不能直接杀死shedule进程。如果杀死shedule 则任务就成孤儿了，laravel再也管不了他的死活，如果使用了 `withoutOverlapping()` 或者 `onOneServer()` redis锁也就无法释放了

```bash
#!/bin/sh## 判断是否有剩余的任务## 进程特征如下# 1. schedule 进程，用于跟踪真正执行的任务，任务完成后，执行对应任务的schedule:finish# sh -c ('/usr/local/bin/php' 'artisan' test > '/dev/null' 2>&1 ; '/usr/local/bin/php' 'artisan' schedule:finish "framework/schedule-***" "$?") > '/dev/null' 2>&1 &# 2. 真正的任务进程# /usr/local/bin/php artisan test

/usr/local/bin/php /var/www/backend/artisan schedule:refresh
# 0-正常运行, 1-Cron退出, 2-PHP任务全部执行完成
status=0

function stop() {
    echo "stopping crond...."
# 终止cron, 不再继续派发任务
    kill $(ps aux | grep '[/]usr/sbin/crond' | awk '{print $1}')
# 对当前正在执行的php任务, 发送SIGTERM, 注意，这里不能对schedule发送，而要对真正任务进程发送，见第8行
    running=$(ps aux | grep '[s]h -c' -c)
    if [ $running -gt 0 ]; then
        echo "Send SIGTERM to running process: $running"
        kill $(ps aux | grep '[/]usr/local/bin/php artisan' | awk '{print $1}')
    fi
    status=1
}

trap stop SIGQUIT SIGINT SIGTERM
#trap 'exit 1' SIGTERM

/usr/sbin/crond

running=0

while true; do
    if [ $status -eq 1 ]; then
# 通过观察schedule进程，判断是否有未完成的schedule，见第6行# 有变化则输出
        currentRunning=$(ps aux | grep '[s]h -c' -c)
        if [ $running -ne $currentRunning ]; then
            running=$currentRunning
            echo "running process: $running"
        fi
        if [ $running -lt 1 ]; then
            echo "program exit"
            exit 0
        fi
    fi
    sleep 1
done

```

### 坑

1. 一开始只trap SIGTERM, docker stop正常，但在k8s下始终收不到SIGTERM，排查后发现，信号在php官方镜像里被修改为了 SIGQUIT

[https://github.com/docker-library/php/blob/master/8.1/alpine3.21/fpm/Dockerfile#L275](https://github.com/docker-library/php/blob/master/8.1/alpine3.21/fpm/Dockerfile#L275)

```docker
# Override stop signal to stop process gracefully
# <https://github.com/php/php-src/blob/17baa87faddc2550def3ae7314236826bc1b1398/sapi/fpm/php-fpm.8.in#L163>
STOPSIGNAL SIGQUIT

```

至于为什么要选择 SIGQUIT 而 不是 SIGTERM 可以看下 nginx 这个 issue

[https://github.com/nginxinc/docker-nginx/issues/377](https://github.com/nginxinc/docker-nginx/issues/377)

[https://trac.nginx.org/nginx/ticket/753](https://trac.nginx.org/nginx/ticket/753)

1. 确保使用exec模式而不是shell模式

[https://docs.docker.com/reference/dockerfile/#cmd](https://docs.docker.com/reference/dockerfile/#cmd)
