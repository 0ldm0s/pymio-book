# 助手类方法函数

## 日志类

### 快速模板

#### 用于通常函数的模板

例如celery或是web的场合，这里以web来举例，这并非完整的代码，仅用于展示代码的写法。

```python
import inspect
import simplejson as json
from flask import request
from mio.util.Logs import LogHandler
from . import main


def get_logger(name: str) -> LogHandler:
    return LogHandler('{}.{}'.format('PyMio', name))

@main.route('/save_something', methods=['POST'])
def companies_save():
    logger: LogHandler = get_logger(inspect.stack()[0].function)
    try:
        data: Union[bytes, Dict[str, Any]] = request.stream.read()
        data = json.loads(data.decode('utf-8'))
        logger.debug(f'传入的数据为：{data}')
    except Exception as e:
        logger.error(e)
        return ApiResponse(code=400, description=u'无效的数据').to_jsonify()
```

#### 用于cli函数类的快速模板

```python
import inspect
from mio.util.Logs import LogHandler


class Daemon(object):
    def __get_logger__(self, name: str) -> LogHandler:
        name = '{}.{}'.format(self.__class__.__name__, name)
        return LogHandler(name)
    
    def do_test(self, app, kwargs):
        logger = self.__get_logger__(inspect.stack()[0].function)
        logger.info('do something')
```

## 方法类函数

### check_ua

检查ua字符串中是否包含特定的内容

#### 传入参数

| 名称 | 类型      | 备注               |
| ---- | --------- | ------------------ |
| keys | List[str] | 要检出的字符串列表 |

#### 返回

| 类型 | 备注                            |
| ---- | ------------------------------- |
| bool | true表示检出，false表示未检出。 |

### check_bot

检查是否是机器人访问

#### 传入参数

无

#### 返回

| 类型 | 备注                            |
| ---- | ------------------------------- |
| bool | true表示检出，false表示未检出。 |

### check_ie

检查是否为ie浏览器访问

#### 传入参数

无

#### 返回

| 类型 | 备注                            |
| ---- | ------------------------------- |
| bool | true表示检出，false表示未检出。 |

### in_dict

检测dict内是否包含指定的key

#### 传入参数

| 名称 | 类型 | 备注         |
| ---- | ---- | ------------ |
| dic  | dict | 要检出的dict |
| key  | str  | 指定的key    |

#### 返回

| 类型 | 备注                            |
| ---- | ------------------------------- |
| bool | true表示检出，false表示未检出。 |

### is_enable

判断是否启用

#### 传入参数

| 名称 | 类型 | 备注         |
| ---- | ---- | ------------ |
| dic  | dict | 要检出的dict |
| key  | str  | 指定的key    |

#### 返回

| 类型 | 备注                          |
| ---- | ----------------------------- |
| bool | false表示未启用，true表示启用 |

### get_real_ip

获取真实ip。支持大部分模式（包括cloudflare）

#### 传入参数

无

#### 返回

| 类型 | 备注   |
| ---- | ------ |
| str  | ip地址 |

### check_is_ip

判断传入的字符串是否为ip地址

#### 传入参数

| 名称    | 类型 | 备注           |
| ------- | ---- | -------------- |
| ip_addr | str  | 要检测的字符串 |

#### 返回

| 类型 | 备注                            |
| ---- | ------------------------------- |
| bool | true表示检出，false表示未检出。 |

### timestamp2str

时间戳转文本。注意：会受到系统默认设置的时区影响。

#### 传入参数

| 名称       | 类型 | 备注                              |
| ---------- | ---- | --------------------------------- |
| timestamp  | int  | 时间戳                            |
| iso_format | str  | 转出格式，默认为%Y-%m-%d %H:%M:%S |
| hours      | int  | 时差校准-小时                     |
| minutes    | int  | 时差校准-分钟                     |

#### 返回

| 类型          | 备注                                                         |
| ------------- | ------------------------------------------------------------ |
| Optional[str] | 如果不为正常的时间戳或时差校准异常，则输出None，否则输出日期文本 |

### str2timestamp

文本转时间戳。注意：会受到系统默认设置的时区影响。

#### 传入参数

| 名称       | 类型 | 备注                                                         |
| ---------- | ---- | ------------------------------------------------------------ |
| date       | str  | 文本日期                                                     |
| iso_format | str  | 日期格式，很重要，如果不匹配，会导致转出失败，默认为%Y-%m-%d %H:%M:%S |
| hours      | int  | 时差校准-小时                                                |
| minutes    | int  | 时差校准-分钟                                                |

#### 返回

| 类型          | 备注                                                         |
| ------------- | ------------------------------------------------------------ |
| Optional[int] | 如果不为正常的时间戳或时差校准异常，则输出None，否则输出时间戳 |

### get_bool

### get_int

### get_root_path

### file_lock

### write_txt_file

### read_txt_file

### write_file

### read_file

### file_unlock

### random_str

### random_number_str

### random_char

### get_file_list

### check_file_in_list

### crc_file

### is_number

### safe_html_code

### ant_path_matcher

### check_email

### get_args_from_dict

### get_variable_from_request

### get_local_now

### get_utc_now

### get_this_week_range

### get_this_month_range

### get_month_range

### get_today

### get_yesterday

### get_this_days_range

### get_now_microtime

### microtime

### md5

### base64_encode

### base64_decode

### base64_txt_encode

### base64_txt_decode

### rounded

### easy_encrypted

### check_chinese_mobile

### str2int