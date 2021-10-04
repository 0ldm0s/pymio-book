# 数据库操作相关

## MongoDB

安装、权限配置和新建数据库不在这里赘述，请自行查阅对应文档。

### 启用压缩传输

主要使用的是zstd，需要先安装好`zstandard`

```python
MONGODB_SETTINGS = {
    'db': '数据库名',
    'host': 'mongodb://127.0.0.1/数据库名?compressors=zstd',
    'username': '用户名',
    'password': '密码',
    'connect': False
}
```

### 多数据库

#### 配置文件

```python
MONGO_DB_MARS_ALIAS = 'mars'
MONGO_DB_EARTH_ALIAS = 'earth'

MONGODB_SETTINGS = [
    {
        'ALIAS': MONGO_DB_MARS_ALIAS,
        'DB': MONGO_DB_MARS_ALIAS,
        'USERNAME': MONGO_DB_MARS_ALIAS,
        'PASSWORD': 'password_mars',
        'HOST': 'private-db.mars.net',
        'PORT': 27017,
        'REPLICASET': 'MainRepSet',
    },
    {
        'ALIAS': MONGO_DB_EARTH_ALIAS,
        'DB': MONGO_DB_EARTH_ALIAS,
        'USERNAME': 'MONGO_DB_EARTH_ALIAS',
        'PASSWORD': 'password_earth',
        'HOST': 'private-db.earth.net',
        'PORT': 27017,
    }
]
```

#### 模型类

```python
from config import MONGO_DB_MARS_ALIAS, MONGO_DB_EARTH_ALIAS

class MarsActivityLogs(DynamicDocument):
    module = StringField(default='')
    tags = ListField(StringField(), default=[])
    data_dict = DictField(default={})

    created_at = DateTimeField(default=datetime.now().strftime('%Y-%m-%d %H:%M:%S'))
    updated_at = DateTimeField(default=datetime.now().strftime('%Y-%m-%d %H:%M:%S'))

    meta = {
        'db_alias': MONGO_DB_MARS_ALIAS, ## set database alias, this is the most important thing
        'collection': 'activity_logs' ## this is the collection name from your database alias
    }

class EarthActivityLogs(DynamicDocument):
    module = StringField(default='')
    tags = ListField(StringField(), default=[])
    data_dict = DictField(default={})

    created_at = DateTimeField(default=datetime.now().strftime('%Y-%m-%d %H:%M:%S'))
    updated_at = DateTimeField(default=datetime.now().strftime('%Y-%m-%d %H:%M:%S'))

    meta = {
        'db_alias': MONGO_DB_EARTH_ALIAS,
        'collection': 'activity_logs'
    }
```

### 需要注意的地方

#### 使用之前先启用

切记！务必！必须！设为`True`

```python
MONGODB_ENABLE = os.environ.get('MIO_MONGODB_ENABLE', True)
```

#### 自增陷阱

MongoEngine提供自增字段，但是其本质上是一个Int字段，搭配上一个用于控制的集合。因此直接写入id的时候，并不会触发控制集合的数据产生自增，因此如果必须要导入外部数据，而且必须使用自增id主键的时候。请务必使用`insert`或是直接新建对象然后`save`。最差情况下，可以采用导入后，手工修改控制字段，或是透过command执行对应的操作。如无特殊情况，请勿使用自增字段。

#### 原子性算法陷阱

MongoDB提供`inc`和`dec`的原子性操作用于增加或减少特定字段的数值。但这里需要注意的是，浮点字段，哪怕是在MongoEngine里定义成双精度字段，依然会出现计算偏差。这是因为MongoDB目前（5.0）仅提供了单精度浮点，就算定义成双精度字段，实际存储到MongoDB内时依然是单精度浮点，因此执行原子性计算时，会发生数值偏差。

所以，如果需要对浮点数进行计算，**请读取后作为双精度来计算或是转换成INT类型**（例如金额x100强制转INT）

### Code Time

以下是方便快速搭建的模板，可以直接复制后改改字段名就能使用。

#### 定义模型 config/models.py

虽然数据库操作类并没有特别的指定，但模型类的位置和文件名是固定的。

要注意的是，被引用的模型类，必须在引用的模型类之前。下面的代码就是一个比较常见的组合情况。

需要注意的是，如果需要使用`LazyReferenceField`则需要被引用的模型类为`DynamicDocument`

并且使用`LazyReferenceField`字段的数据时，需要先进行`fetch()`

```python
# -*- coding: UTF-8 -*-
from bson import ObjectId
from mongoengine import *


class User(DynamicDocument):
    id = ObjectIdField(primary_key=True, default=lambda: ObjectId())
    nickname = StringField()
    avatar = StringField()
    gender = IntField(default=0)  # 0隐藏 1男 2女
    birthday = LongField(default=0)  # 生日
    openid = StringField()  # 那就以openid为准
    money = IntField(default=0)
    is_del = BooleanField(default=False)
    is_block = BooleanField(default=False)  # 是否封禁
    create_at = LongField()
    update_at = LongField()
    meta = {
        'auto_create_index': True,
        'index_background': True,
        'index_cls': True,
        'indexes': [
            '$openid',
        ]
    }


class Feedback(Document):
    id = ObjectIdField(primary_key=True, default=lambda: ObjectId())
    from_user = LazyReferenceField(User, default=None)  # 如果有就记录，没有就不记录
    name = StringField()  # 手填，姓名
    contact = StringField()  # 手填，联系方式
    message = StringField()  # 正文
    is_del = BooleanField(default=False)
    create_at = LongField()
    update_at = LongField()
    meta = {
        'auto_create_index': True,
        'index_background': True,
        'index_cls': True,
        'indexes': [
            '#id',
        ]
    }

```

#### 对象操作类

对象操作类，一般为了方便阅读，建议放在`model`目录下。

##### model/UserObj.py

只展示几个基础函数，具体细节自行处理。因为都很简单，就不再进行细节说明。具体请参阅MongoEngine文档。

```python
import inspect
from bson import ObjectId
from mongoengine.queryset import Q
from flask_mongoengine.pagination import Pagination
from typing import Optional, Union, List, Tuple, Dict, Any
from mio.util.Logs import LogHandler
from mio.util.Helper import get_local_now, timestamp2str
from config.models import User

class UserObj(object):
    def __get_logger__(self, name: str) -> LogHandler:
        name = '{}.{}'.format(self.__class__.__name__, name)
        return LogHandler(name)

    def get_list(self, page: int, limit: int, keyword: Optional[str] = None) -> Tuple[int, List[User]]:
        logger = self.__get_logger__(inspect.stack()[0].function)
        try:
            query = Q(is_del=False)
            if keyword is not None:
                query = query & Q(nickname__icontains=keyword)
            pagination = Pagination(User.objects.filter(query).order_by('-id'), page, limit)
            return pagination.total, pagination.items
        except Exception as e:
            logger.error(e)
            return 0, []
        
    def get_one_by_id(self, oid: Union[str, ObjectId]):
        logger = self.__get_logger__(inspect.stack()[0].function)
        try:
            item: User = User.objects.get(id=oid, is_del=False)
            return item
        except Exception as e:
            logger.debug(e)
            return None
        
    def add_one(self, nickname: str, avatar: str, gender: str, birthday: str, openid: str) -> Optional[User]:
        logger = self.__get_logger__(inspect.stack()[0].function)
        try:
            dt: int = get_local_now(hours=8)
            item: User = User(
                nickname=nickname, avatar=avatar, gender=gender, birthday=birthday, openid=openid
            )
            item.create_at = dt
            item.update_at = dt
            item.save()
            return item
        except Exception as e:
            logger.error(e)
            return None
        
    @staticmethod
    def quick_save(item: User):
        item.update_at = get_local_now(hours=8)
        item.save()
        
    @staticmethod
    def unpack_item(item: User) -> Dict[str, Any]:
        birthday: Optional[str] = timestamp2str(item.birthday, '%Y-%m-%d')
        birthday = '' if birthday is None else birthday
        output: Dict[str, Any] = {
            'id': str(item.id),
            'nickname': item.nickname,
            'avatar': item.avatar,
            'gender': item.gender,
            'birthday': birthday,
            'money': '{:0,.2f}'.format(float(item.money / 100)),
            'is_block': item.is_block,
            'create_at': timestamp2str(item.create_at, '%Y-%m-%d %H:%M'),
        }
        return output
```

## Redis

Redis是整个框架的重要部分，即负责页面缓存，同时也负责数据缓存，并且作为简易消息队列为celery提供支持。

### 记得先启用

注意：数据缓存和页面缓存可以配置到不同的Redis上，因此开关也不同。页面缓存使用的是flask_cache，这里不进行讨论。

```python
# 是否使用Redis，记得启用
REDIS_ENABLE = os.environ.get('MIO_REDIS_ENABLE', True)
# Redis前导，记得修改
REDIS_KEY_PREFIX = 'PYMIO'

# 对应的配置不要忘记修改
REDIS_URL = 'redis://127.0.0.1:6379/0'
```

### 直接操作

```python
from mio.sys import redis_db

# redis_db就是初始化好的Redis类，可以直接操作
info = redis_db.info()
print(info)
```

### 使用插件

在`base_mio`项目中，有个`plugins`文件夹，内含`quick_cache`，推荐使用这个插件来操作redis。

只要直接引入就可以使用

```python
from plugins.quick_cache import QuickCache

is_ok, data = QuickCache().cache(redis_key)
```

### 函数方法

#### get_keys

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型      | 备注                      |
| --------- | ------------------------- |
| List[str] | 符合条件的redis key的列表 |

#### lpush

压入一个数据到列表中

##### 传入参数

| 名称        | 类型          | 备注                                                |
| ----------- | ------------- | --------------------------------------------------- |
| key         | str           | redis key                                           |
| value       | Optional[Any] | 任意数据类型                                        |
| expiry      | int           | 过期时间，对整个list生效，非单一元素                |
| is_full_key | bool          | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型 | 备注                                |
| ---- | ----------------------------------- |
| bool | 如操作成功则返回True，失败返回False |

#### llen

返回列表长度

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型 | 备注 |
| ---- | ---- |
| int  | 长度 |

#### inc_num

原子性操作，增加数值

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| num         | int  | 要增加的数值                                        |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型          | 备注                         |
| ------------- | ---------------------------- |
| Optional[int] | 成功则返回数值，失败返回None |

#### dec_num

原子性操作，减少数值

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| num         | int  | 要减少的数值                                        |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型          | 备注                         |
| ------------- | ---------------------------- |
| Optional[int] | 成功则返回数值，失败返回None |

#### rpop

移出并获取列表的最后一个元素

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型          | 备注                                       |
| ------------- | ------------------------------------------ |
| Optional[Any] | 如果有获取到数据就返回数据，无数据则为None |

#### cache

##### 传入参数

| 名称        | 类型          | 备注                                                         |
| ----------- | ------------- | ------------------------------------------------------------ |
| key         | str           | redis key                                                    |
| value       | Optional[Any] | 任意类型数据，甚至可以是QuerySet，如果不传或传None则表示读取缓存 |
| expiry      | int           | 过期时间，单位秒                                             |
| is_full_key | bool          | 是否全写key，如果为否，则会自动拼接key，默认为False          |
| is_pickle   | bool          | 是否使用pickle进行打包，默认为True。仅读取时有效。           |

##### 返回

| 类型          | 备注                                                         |
| ------------- | ------------------------------------------------------------ |
| bool          | 操作是否成功，因为有可能是操作成功，但是取回的数据是None的情况 |
| Optional[Any] | 存入的数据                                                   |

#### remove_cache

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

无

#### cache_page

可以直接替代系统的`render_template`，并且带有缓存功能

##### 传入参数

| 名称              | 类型 | 备注                                                |
| ----------------- | ---- | --------------------------------------------------- |
| key               | str  | redis key                                           |
| template_filename | str  | 模板文件相对路径路径，例如: page/index.html         |
| expiry            | int  | 过期时间，单位：秒                                  |
| is_full_key       | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |
| **kwargs          |      | 模板内调用的变量的映射关系                          |

##### 返回

| 类型          | 备注                                             |
| ------------- | ------------------------------------------------ |
| Optional[str] | 渲染后的文本内容。如果模板文件不存在，则返回None |

#### read_page

##### 传入参数

| 名称        | 类型 | 备注                                                |
| ----------- | ---- | --------------------------------------------------- |
| key         | str  | redis key                                           |
| expiry      | int  | 刷新过期时间，单位：秒                              |
| is_full_key | bool | 是否全写key，如果为否，则会自动拼接key，默认为False |

##### 返回

| 类型          | 备注                                             |
| ------------- | ------------------------------------------------ |
| Optional[str] | 渲染后的文本内容。如果不存在页面缓存，则返回None |

