# 关于PyMio

一个基于flask的多功能框架，整合tornado、libuv、markdown、babel、celery、redis及mongoengine。

可能是目前最好的flask框架，快速、轻量、像我的光头一样易于打理。

## 搭建框架

事实上，pymio只是一个轻量化的框架核心，除了部分文件夹命名需要跟随规则外，其他均无特定规则。

### 从模板项目开始

推荐初学者从这个模式开始，这个模式整合了大量常用的配置和文件，可以做到开箱即用。

```shell
# 请自行替换<新项目名称>为实际的内容
git clone https://github.com/0ldm0s/base_mio.git <新项目名称>
# 建议移除旧的记录
cd <新项目名称>
rm -Rf .git
# 跳至安装依赖安装步骤
```

### 从核心开始搭建

初学者并不推荐这个模式，但可以作为理解整个项目结构的一个方法。范例所使用的操作系统为 Ubunutu 20.04，不考虑实际的权限控制之类的优化措施，仅只展示如何搭建一个框架。

**备注：接下来采用的是正统的git子模组的方式进行，这并非唯一的方案。亦可采用直接下载核心后构建文件夹结构的方式来实现。**

接下来的范例，将以传统`helloworld`来进行。在具体的实践中，请替换为实际的项目名称。

#### 建立项目

```shell
mkdir helloworld
cd helloworld
git init
git submodule add https://github.com/0ldm0s/pymio.git mio
cd mio
# 安装依赖
apt-get install build-essential zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev libncurses5-dev xz-utils libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev sqlite3 libpcre3 libpcre3-dev libssl-dev uuid-dev libtbb-dev libuv1-dev
# 参阅依赖安装步骤完成依赖安装
# 完成依赖安装后，返回项目根目录
cd ..
```

#### 建立项目结构

```shell
mkdir config
mkdir web
mkdir -p web/main
mkdir -p web/static
mkdir -p web/template
touch config/__init__.py
touch config/config.yaml
touch web/main/__init__.py
touch web/main/views.py
touch web/template/index.html
# 为了防止提交git时空文件夹被排除，建立一个空的隐藏文件
touch web/static/.nomedia
```

#### 建立配置文件

##### config/config.yaml

这是一个标准的yaml文件，里面主要定义了蓝本和一些站点安全的配置。

**备注：虽然cli不需要配置蓝本，但是websocket需要**

```yaml
blueprint:
  - main:
      class: web.main
config:
  login_manager:
    enable: false
    session_protection: strong
    login_view: main.login
  csrf:
    enable: false
```

这里定义了一个基础的蓝本`main`，并且class的路径为`web.main`。login_manager仅作为历史遗留设置存在，前后端完全分离时，login_manager应显示的禁用以提升性能，csrf亦是如此。

##### config/__init__.py

系统的主要配置在这个文件中配置。此文件仅作为基础，您可以自由的调整或添加新的配置参数。3套配置分别对应：`development`开发环境（默认）、`testing`测试环境和`production`生产环境。可以通过系统变量或启动参数调整为不同的环境。

**注意：由于考虑到性能和安全性问题，PyMio不再内置sqlalchemy支持，建议使用者迁移至MongoDB或MongoDB群集。但如需要使用sqlalchemy，可自行安装和配置。**

**小提示：或许您已经注意到了，大部分的配置信息都可以通过系统环境变量传入**

```python
# -*- coding: utf-8 -*-
import os

basedir = os.path.abspath(os.path.dirname(__file__))
MIO_HOST = os.environ.get('MIO_HOST', '127.0.0.1')
MIO_PORT = int(os.environ.get('MIO_PORT', 5000))
MIO_SITE_HOST = os.environ.get('MIO_SITE_HOST', MIO_HOST)


class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'PYMIO_SECRET_KEY'  # 默认秘钥
    SESSION_TYPE = 'filesystem'
    # 邮件系统设置相关
    MIO_MAIL = False
    MIO_SEND_MAIL = False
    MAIL_SUBJECT_PREFIX = os.environ.get('MIO_MAIL_SUBJECT_PREFIX', '[Mio System]')  # 默认邮件标题前缀
    MAIL_SENDER = os.environ.get('MIO_MAIL_DEFAULT_SENDER', 'Mio System Administrator <admin@example.com>')  # 默认发件人
    MAIL_SERVER = os.environ.get('MIO_MAIL_SERVER', 'localhost')
    MAIL_PORT = os.environ.get('MIO_MAIL_PORT', 25)
    MAIL_USE_TLS = os.environ.get('MIO_MAIL_USE_TLS', False)
    MAIL_USERNAME = os.environ.get('MIO_MAIL_USERNAME', '')
    MAIL_PASSWORD = os.environ.get('MIO_MAIL_PASSWORD', '')
    # 是否使用MONGODB
    MONGODB_ENABLE = os.environ.get('MIO_MONGODB_ENABLE', False)
    # 是否使用CELERY
    CELERY_ENABLE = os.environ.get('MIO_CELERY_ENABLE', False)
    # 是否使用Redis
    REDIS_ENABLE = os.environ.get('MIO_REDIS_ENABLE', False)
    # Redis前导
    REDIS_KEY_PREFIX = 'PYMIO'
    # 是否使用CACHE
    CACHED_ENABLE = os.environ.get('MIO_CACHED_ENABLE', False)
    # 是否使用CORS
    CORS_ENABLE = os.environ.get('MIO_CORS_ENABLE', False)
    CORS_URI = os.environ.get('MIO_CORS_URI', {r"/*": {"origins": "*"}})
    # 支持的语言
    LANGUAGES = ['zh-CN']
    # 默认语言
    DEFAULT_LANGUAGE = 'zh-CN'

    @staticmethod
    def init_app(app):
        app.jinja_env.trim_blocks = True
        app.jinja_env.lstrip_blocks = True


class DevelopmentConfig(Config):
    DEBUG = True
    MONGODB_SETTINGS = {
        'db': 'db_name',
        'host': 'localhost',
        'username': 'username',
        'password': 'password',
        'connect': False
    }
    REDIS_URL = 'redis://localhost:6379/0'
    CACHE_TYPE = 'simple'
    CACHE_REDIS_URL = REDIS_URL
    CELERY_BROKER_URL = REDIS_URL
    CELERY_BACKEND_URL = REDIS_URL


class TestingConfig(Config):
    TESTING = True
    MONGODB_SETTINGS = {
        'db': 'db_name',
        'host': 'localhost',
        'username': 'username',
        'password': 'password',
        'connect': False
    }
    REDIS_URL = 'redis://localhost:6379/0'
    CACHE_TYPE = 'simple'
    CACHE_REDIS_URL = REDIS_URL
    CELERY_BROKER_URL = REDIS_URL
    CELERY_BACKEND_URL = REDIS_URL


class ProductionConfig(Config):
    MONGODB_SETTINGS = {
        'db': 'db_name',
        'host': 'localhost',
        'username': 'username',
        'password': 'password',
        'connect': False
    }
    REDIS_URL = 'redis://localhost:6379/0'
    CACHE_TYPE = 'simple'
    CACHE_REDIS_URL = REDIS_URL
    CELERY_BROKER_URL = REDIS_URL
    CELERY_BACKEND_URL = REDIS_URL


config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,

    'default': DevelopmentConfig
}

```

#### 建立默认蓝本

##### web/main/__init__.py

**小提示：因为flask本身奇怪的限制问题，不得不建立一个web文件夹，因此如果仅需要使用cli的情况下，只要到这一步建立完，就可以开始编写cli命令行了**

```python
# -*- coding: utf-8 -*-
from flask import Blueprint

main = Blueprint('main', __name__)
from . import views

```

#### 建立默认网页

到这一步，就是标准的flask+jinja2的操作了，就不再赘述了。具体请参阅flask与jinjia2。

##### web/main/views.py

```python
# -*- coding: utf-8 -*-
import sys
from flask import render_template
from . import main


@main.route('/')
def index():
    sys_ver = sys.version
    return render_template('index.html', sys_ver=sys_ver)

```

##### web/template/index.html

```html
Powered by PyMio <br>
Python {{ sys_ver }}
```

到这里，一个标准的pymio项目就搭建完成。您可以执行以下命令来看效果

```shell
python mio/pymio.py
```

如无异常，则会提示当前绑定的ip和端口号，只要在浏览器中打开该地址即可看到类似这样的文字。

```
Powered by PyMio
Python 3.9.5 (default, May 18 2021, 22:50:08) [Clang 13.0.0 ]
```

## 核心依赖版本

**如已有现存的环境，请确保核心依赖的版本正确**

```
python ~= 3.7
flask ~= 2.0.1
tornado ~= 6.1
mongoengine ~= 0.23.1
zstandard ~= 0.15.2
```

## 依赖安装

### *nix系统

**注意：生产环境需要安装libuv和uvloop以提高整体性能**

```shell
# 安装基础依赖
pip install -r requirements.first.txt
# 安装依赖
pip install -r requirements.txt
# 如需使用circus，则安装
pip install -r requirements.circus.txt
# 生产环境手动安装uvloop
pip install uvloop
```

**备注：FreeBSD系统的安装略微不同**

### Windows系统

如非开发环境，建议使用3.9.x获取更好的性能，调试则建议使用3.7.x配合Pycharm。

如条件允许，则应使用wsl或hyper-v，不推荐直接运行于windows环境。

```shell
# 安装基础依赖
pip install -r requirements.first.txt
# 安装依赖
pip install -r requirements.txt
```