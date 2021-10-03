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

## 方法类函数

### check_ua

检查ua字符串中是否包含特定的内容

#### 传入参数

| 名称 | 类型      | 备注               |
| ---- | --------- | ------------------ |
| keys | List[str] | 要检出的字符串列表 |

#### 返回

| 类型 | 备注                                                  |
| ---- | ----------------------------------------------------- |
| bool | 是否检出包含的字符串。true表示检出，false表示未检出。 |

### check_bot

### check_ie

### in_dict

### is_enable

### get_real_ip

### check_is_ip

### timestamp2str

### str2timestamp

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