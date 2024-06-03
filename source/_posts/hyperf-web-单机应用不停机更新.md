---
title: hyperf web 单机应用不停机更新
date: 2024-06-03 09:54:29
tags: hyperf
---

在 `hyperf` 应用中如果修改了代码，我们就必须要重启应用才能使代码生效，但是生产环境中停机是会对服务产生巨大影响的，如果服务频繁的改动需要频繁的停机更新，这肯定是不合理的。有没有什么办法可以在服务能够继续使用的情况下更新代码呢，答案肯定是有的。

<!-- more -->

根据官方文档，一种是使用 `docker` 容器方案，比如使用一些 `k8s` 或者 `swarm` 等容器管理工具，开启多个服务，当需要更新时，依次销毁并建立新容器，当销毁一个容器时，`k8s` 之类的工具会把流量引入到还未销毁的容器当中，交替完成以后就是所有的服务更新完毕。

但是，并不是所有的公司或者应用都需要使用到容器方案，一个是会增加了我们的学习成本，另一个是增加了运维成本，当我们需要原机上也要实现这种效果应该怎么做？

其实根据容器方案，我们可以想到两个关键信息，一个是多应用，第二个是负载均衡。

那么根据关键信息，我们可以想到对应方案是：
1. 多应用，可以在多机上部署多个服务，但是我们的应用又不一定要多机，那单机多端口一样是实现了多应用
2. 负载均衡，那就可以考虑到 `nginx`，当第一个应用不可用时，我们就把流量转到第二个应用

## 前期准备

1. 两个 `web` 应用 `web1`, `web2`，分别监听 `9502`, `9503` 端口
2. 使用 `supervisor` 管理应用
3. 有 `nginx` 服务

## 启动服务

```
# 两个 web 应用的目录 
# /home/vagrant/code/dev/hyperf-test/web1
# /home/vagrant/code/dev/hyperf-test/web2

cd /home/vagrant/code/dev/hyperf-test/web1

# 启动应用
php bin/hyperf.php start

# 访问 9502 看看能否正常使用
# 同理启动 web2 的应用
```

## 优雅关闭

前面说到有一个环节需要重启服务，那停服务的时候，如果已经有请求还没处理完毕，那我们直接 `kill` 掉，那么会导致这个请求丢失，最后无法响应，那么有没有方法是当我们需要关闭服务的时候，里面所有的请求处理完毕再关闭呢。当我们关闭服务的时给应用发送一个信号 `kill -15`(`kill sigterm`), `hyperf` 可以正确的响应这个信号，把还在处理中的请求处理完毕再进行关闭服务。

我们用 `web1` 测试一下

`App\Controller\IndexController@index`
```
public function index()
{
    $user = $this->request->input('user', 'Hyperf');
    $method = $this->request->getMethod();

    Coroutine::sleep(3);
    
    return [
        'method' => $method,
        'message' => "Hello web1.",
    ];
}
```

重新启动服务并访问，可以看到请求在3秒后做出响应
![](/images/hyperf-web-单机应用不停机更新/1.png)

这次我们测试，请求以后马上对服务(进程pid)发出停止讯号，注意后续如果有不同的测试,`pid` 在项目的 `runtime` 中查找
```
sudo kill -15 1525411
```
![](/images/hyperf-web-单机应用不停机更新/2.png)
![](/images/hyperf-web-单机应用不停机更新/3.png)
![](/images/hyperf-web-单机应用不停机更新/4.png)

可以看到应用还是正常的得到响应，我们试试把程序睡眠的时间调大一点，改到10试试
![](/images/hyperf-web-单机应用不停机更新/5.png)
![](/images/hyperf-web-单机应用不停机更新/6.png)
可以看到这次，服务没有得到正确的响应，看报错那是因为 `swoole` 接受到停止信号后 `worker` 的最大等待时间默认为3秒, `max_wait_time` 那我们修改这个值就可以了

`config/autoload/server.php`
```php
// 在 settings 中增加
'settings' => [
    Constant::OPTION_MAX_WAIT_TIME => 10
],
```
```
public function index()
{
    $start = time();
    $user = $this->request->input('user', 'Hyperf');
    $method = $this->request->getMethod();

    Coroutine::sleep(10);

    $end = time();
    return [
        'method' => $method,
        'message' => "Hello web1.",
        'start' => $start,
        'end' => $end,
        'interval' => $end - $start,
        'now' => date('Y-m-d H:i:s', $end),
    ];
}
```
启动服务再次测试
![](/images/hyperf-web-单机应用不停机更新/7.png)
![](/images/hyperf-web-单机应用不停机更新/8.png)
可以看到我们在 `11.06.59` 的时候就已经做出了关闭请求，最后应用能够正常的响应服务，`11.07.07` 处理完请求。

把代码和配置同步到 `web2`

```
public function index()
{
    $start = time();
    $user = $this->request->input('user', 'Hyperf');
    $method = $this->request->getMethod();

    Coroutine::sleep(10);

    $end = time();
    return [
        'method' => $method,
        'message' => "Hello web2.", // 修改这行
        'start' => $start,
        'end' => $end,
        'interval' => $end - $start,
        'now' => date('Y-m-d H:i:s', $end),
    ];
}
```

## Supervisor

使用 `supervisor` 去管理我们的 `web` 应用

当我们使用 `supervisotctl stop` 的时候，实际上它也是给应用发送了 `kill sigterm` 信号

`web1.conf`
```
[program:web1]
directory=/home/vagrant/code/dev/hyperf-test/web1
command=php ./bin/hyperf.php start
user=root
autostart=true
startsecs=1
autorestart=true
startretries=3
stderr_logfile=/var/www/hyperf/web1/runtime/stderr.log
stdout_logfile=/var/www/hyperf/web1/runtime/stdout.log
```

`web2.conf`
```
[program:web2]
directory=/home/vagrant/code/dev/hyperf-test/web2
command=php ./bin/hyperf.php start
user=root
autostart=true
startsecs=1
autorestart=true
startretries=3
stderr_logfile=/var/www/hyperf/web2/runtime/stderr.log
stdout_logfile=/var/www/hyperf/web2/runtime/stdout.log
```

```
sudo supervisorctl update
sudo supervisorctl status
```
![](/images/hyperf-web-单机应用不停机更新/9.png)
![](/images/hyperf-web-单机应用不停机更新/10.png)

## Nginx 负载

使用一个统一的入口，然后把流量再转发给 `web1`, `web2`

```
sudo vim hyperf-update-code-test.conf

# 加入以下配置

# 至少需要一个 Hyperf 节点，多个配置多行
upstream hyperf-test {
    # Hyperf HTTP Server 的 IP 及 端口
    server 127.0.0.1:9502;
    server 127.0.0.1:9503;
}

server {
    # 监听端口
    listen 9999;

    location / {
        # 将客户端的 Host 和 IP 信息一并转发到对应节点
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 转发Cookie，设置 SameSite
        proxy_cookie_path / "/; secure; HttpOnly; SameSite=strict";

        # 执行代理访问真实服务器
        proxy_pass http://hyperf-test;
    }
}
```
```
sudo nginx -t
sudo nginx -s reload
```

![](/images/hyperf-web-单机应用不停机更新/11.png)
可以看到负载已经成功,但是我们真正想要的并不是负载，我们想要的只是 `web2` 做一个备份就行，流量大部分情况下还是往 `web1` 去运行就行，改一下参数
```
server 127.0.0.1:9503 backup;
```
```
sudo nginx -t
sudo nginx -s reload
```
![](/images/hyperf-web-单机应用不停机更新/12.png)
可以看到流量都往 `web1` 上转发了，这时候我们测试开启4个浏览器端口，第一个先访问应用，然后这时关闭掉 `web1` 的服务，再访问第二个，第三个，开启 `web1` 的服务，访问第四个

按照我们的设想，如果正常,第一个被 `web1` 接收到，虽然是中断了服务，但是是优雅关闭，请求应该还是能正常响应，第二、三个服务被 `web2` 接受响应，第四个又被重新启动的 `web1` 响应处理

开始模拟测试

```
# 第一个访问之后执行
sudo supervisorctl stop web1

# 第三个访问之后执行
sudo supervisorctl start web1
```
![](/images/hyperf-web-单机应用不停机更新/13.png)
![](/images/hyperf-web-单机应用不停机更新/14.png)

可以看到确实是我们想要的效果，`web2` 只有在 `web1` 不再接受流量的时候处理，当 `web1` 可用的时候 `web2` 又不再处理了

至此，整个流程已经完毕，那后面的代码更新就比较简单了，原理就是先让 `web1` 拉取最新代码，关闭服务，启动服务，等待服务完全启动可用以后，再让 `web2` 重复上述步骤，完成以后本次更新完毕。

## 编写更新脚本

注意在写脚本的时候，我们应该是先更新备用端口，等备用端口可用以后再去更新主端口

`update.py`
```python
import subprocess
import time
import requests

class Update:
    projects = [
        {'name': 'web2', 'dir': '/home/vagrant/code/dev/hyperf-test/web2', 'url': 'http://192.168.56.56:9503'},
        {'name': 'web1', 'dir': '/home/vagrant/code/dev/hyperf-test/web1', 'url': 'http://192.168.56.56:9502'}
    ]

    def start(self):
        n = len(self.projects)
        for index in range(n):
            appName = self.projects[index]['name']
            projectDir = self.projects[index]['dir']
            appURL = self.projects[index]['url']
            print(f"Updating project: {appName} in directory: {projectDir}")

            # 拉取代码
            # process = subprocess.Popen(['git', 'pull'], cwd=projectDir, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            # for line in iter(process.stdout.readline, b''):
            #     print(line.decode('utf-8').strip())
            
            # 生成新项目缓存
            process = subprocess.Popen(['composer', 'dump-autoload', '-o'], cwd=projectDir, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            for line in iter(process.stdout.readline, b''):
                print(line.decode('utf-8').strip())

            # 使用 supervisorctl 重启项目
            process = subprocess.Popen(['sudo', 'supervisorctl', 'restart', appName], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            for line in iter(process.stdout.readline, b''):
                print(line.decode('utf-8').strip())

            # 监听项目启动
            print(f"Waiting for {appName} to start...")
            process = subprocess.Popen(['sudo', 'supervisorctl', 'status', appName], stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            # 监听状态，直到状态为 RUNNING
            while True:
                try:
                    # 发送HTTP请求检查项目是否启动
                    response = requests.get(appURL)
                    if response.status_code == 200:
                        print(f"{appName} is running.")
                        # 等待一段时间再继续下一个项目
                        time.sleep(3)
                        break
                    else:
                        print(f"{appName} is not yet running. Waiting...")
                        time.sleep(1)
                except Exception as e:
                    pass

        print("All projects updated and restarted.")


if __name__ == '__main__':
    Update().start()
```

`test.py`
```python
import threading
import requests
import time

# 定义要请求的URL
url = "http://192.168.56.56:9999"

# 定义请求函数
def make_request():
    while True:
        try:
            # 发起HTTP请求
            response = requests.get(url)
            # 输出响应内容
            print(response.text)
        except Exception as e:
            # 输出异常信息
            print("Exception:", e)
        # 等待一段时间再次请求
        time.sleep(1)

# 定义线程数
num_threads = 5

# 创建并启动线程
threads = []
for _ in range(num_threads):
    thread = threading.Thread(target=make_request)
    thread.start()
    threads.append(thread)

# 等待所有线程完成
for thread in threads:
    thread.join()

```

测试
```
python3 test.py

python3 update.py
```
可以看到测试脚本的输出都是正常输出的，可以尝试更新以后看看输出。




