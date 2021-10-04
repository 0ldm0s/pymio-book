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

正常执行的话，应该会看到类似的内容

```shell
{'key1': 'hello', 'key2': 'world'}
```

需要注意的是，`||`被作为分隔符使用，因此参数中建议不要出现`||`，除此之外，与普通的命令行传参无异。

## Celery

