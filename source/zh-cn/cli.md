# 命令行/CLI

命令行是PyMIO的一个重要组成部分，不仅可以承担大部分的耗时操作，自身亦可作为TCP/UDP服务器来使用。只要能想得到，基本上就能实现。CLI是Web的重要扩展和后台支撑。

## 标准CLI

标准cli可以用于实现各种耗时操作（例如TCP服务器）

一个标准的cli函数会包含如下几个部分

```python
def hello(self, app, kwargs):
```

`app`等价于`from flask import current_app`，但因为cli模式无法直接获取到`current_app`，故而直接传入。

`kwargs`为一个dict，用于存放通过命令行传入的参数。

### 实例

#### 启动命令

```shell
env FLASK_APP=mio.shell flask cli exe -cls=cli.WorkMan.Daemon.hello
```

正常执行的话，应该会看到类似的内容

```shell
2021-10-04 06:52:36,419 [PID 31791] [INFO] PyMio -> Initializing the system......profile: default
Powered by PyMio.
Python: 3.9.5 (default, May 18 2021, 22:50:08)
[Clang 13.0.0 ]
```

#### 带参数的启动命令

首先修改代码，打印出`kwargs`

```python
print(kwargs)
```

然后执行这个命令行

```shell
env FLASK_APP=mio.shell flask cli exe -cls=cli.WorkMan.Daemon.hello -arg="key1=hello||key2=world"
```

如果需要指定pid文件路径，可以使用`-pid`参数来指定。例如

```shell
env FLASK_APP=mio.shell flask cli exe -cls=cli.WorkMan.Daemon.hello -arg="key1=hello||key2=world" -pid=cli.pid
```

正常执行的话，应该会看到类似的内容

```shell
{'key1': 'hello', 'key2': 'world'}
```

需要注意的是，`||`被作为分隔符使用，因此参数中建议不要出现`||`，除此之外，与普通的命令行传参无异。

## Celery

celery可以用来作为后端的worker支撑，同时也可以用来实现定时计划。需要注意的是，worker模式和心跳模式是分开的，虽然可以同时启动，但启动参数需要各自单独指定。

不要忘记一点，celery的worker进程数是和cpu核心数是一致的。例如有1个核心，就只会启动1个worker进程，2个核心则启动2个，如此类推。一个worker同时只能执行一个工作，在这个工作未结束之前，不会出处理后续的事务。当然，也可以同时部署多个worker来进行横向扩展。

小内存的机器并不推荐使用celery来作为定时器使用，因为会增加额外的内存开销。小内存机器建议使用cron搭配cli来使用，除非本身有异步任务操作的需求，否则应优先考虑cron。docker/k8s这种没有cron的情况例外。

### Code Time

为了文件夹看起来规整，推荐放在cli文件夹下。例如`cli.celery`。

假定我已经建立了一个文件`Worker.py`在`cli.celery`文件夹中。

#### 定义任务函数和定时任务

```python
# -*- coding: utf-8 -*-
import inspect
from celery.schedules import crontab
from mio.sys import celery_app
from mio.util.Logs import LogHandler
from model.DiscountCodeObj import DiscountCodeObj

# 这里定义了一个每天0点0分执行的定时任务，时区为Asia/Shanghai
celery_app.conf.update(
    timezone='Asia/Shanghai',
    enable_utc=True,
    beat_schedule={
        'do_someting': {
            'task': 'cli.celery.Worker.do_someting',
            'schedule': crontab(minute=0, hour=0),
            'args': ()
        },
    }
)

def get_logger(name='logger') -> LogHandler:
    logger = LogHandler(f'cli.celery.{name}')
    return logger


@celery_app.task
def do_someting():
    console_log: LogHandler = get_logger(inspect.stack()[0].function)
    console_log.info('起来干活了')

# 这里定义了一个被外部调用的worker函数
# 这里只能传入str类型的变量，不限制变量个数，也可以设置默认值，当然也可以不传入任何变量
@celery_app.task
def iam_worker(txt: str):
    console_log: LogHandler = get_logger(inspect.stack()[0].function)
    console_log.info(f'收到了任务，参数为 {txt}')
```

#### 从web调用worker

这是比较常见的一种情况，例如批量生成、导入导出等场景，均推荐这种方式来进行。避免数据量过大导致脚本超时等问题。

这只是代码片段，具体请根据实际情况来处理

```python
from cli.celery.Worker import iam_worker


@api_admin.route('/do/someting')
def do_someting():
    iam_worker.delay('我是参数')
    return 'OK'
```

更多细节，请参阅[celery的官方文档](https://docs.celeryproject.org/en/stable/userguide/daemonizing.html)。

### 启动celery

#### 只启动worker的场合

`-A` 表示要启动的celery_app类的位置，这里就直接用范例的位置。

`-w` 表示附加参数，具体可以[参阅celery的官方文档](https://docs.celeryproject.org/en/stable/userguide/daemonizing.html)

```shell
env FLASK_APP=mio.shell flask celery run -A cli.celery.Worker.celery_app -w "loglevel=info E"
```

#### 附带定时器的场合

定时器不能单独启动，只能随worker进程启动。

```shell
env FLASK_APP=mio.shell flask celery run -A cli.celery.Worker.celery_app -w "loglevel=info B E"
```

如果需要指定pid文件

```shell
env FLASK_APP=mio.shell flask celery run -A cli.celery.Worker.celery_app -w "loglevel=info pidfile=celery.pid B E"
```

