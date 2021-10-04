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

