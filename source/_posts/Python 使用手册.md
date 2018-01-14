---
title: Python 使用手册
date: 2017-11-14 00:00:00
categories: 工具
tags: python
---

# python使用手册

## 关于自动导入项目所有依赖库

1.  生成 requirements.txt 文件

    `pip freeze > requirements.txt`

2.  安装 requirements.txt

    `pip install -r requirements.txt`


## 连接mysql数据库

如果使用 flask-sqlalchemy 连接mysql数据库的时候，需要给 app 传送一个 SQLALCHEMY_DATABASE_URI 值，值的格式为`mysql://${username}:$${password}@${hostname}:${port}/${database}`,如`mysql://root:123456@localhost:3306/app`，但是要使用这种方法连接mysql，需要下载一个库`mysqlclient`，有些时候下载会不成功，可以使用以下方法：

```bash
sudo apt-get update
sudo apt-get install python-pip (对应python3的是python3-pip)

sudo apt-get install mysql-server
sudo apt-get install mysql-client

sudo apt-get install libmysql-dev
sodu apt-get install libmysqlclient-dev
sudo apt-get install python-dev
sudo apt-get install mysqlclient
```

一般情况下这样应该能解决。

关于定义数据映射表的信息如下：

|     类型名      |      Python类型      |          说明           |
| :----------: | :----------------: | :-------------------: |
|   Integer    |        int         |       整形，一般32位        |
| SmallInteger |        int         |         一般16位         |
|  BigInteger  |        int         |       不限制精度的整形        |
|    Float     |       float        |          浮点数          |
|   Numeric    |  decimal.Decimal   |          定点数          |
|    Stirng    |        str         |         变长字符串         |
|     Text     |        str         |    变长字符串，适用于较长字符串     |
|   Unicode    |      unicode       |     变长Unicode字符串      |
| UnicodeText  |      unicode       | 变长Unicode字符串，适用于较长字符串 |
|   Boolean    |        bool        |          布尔值          |
|     Date     |   datetime.date    |          日期           |
|     Time     |   datetime.time    |          时间           |
|   DateTime   | datetime.datetime  |         日期和时间         |
|   Interval   | datetime.timedelta |         时间间隔          |
|     Enum     |        str         |         一组字符串         |
|  PickleType  |     任何python对象     |     自动使用pickle序列化     |
| LargeBinary  |        str         |         二进制文件         |

|     列选项     |      说明      |
| :---------: | :----------: |
| primary_key |  Ture：设置为主键  |
|   unique    | True：不允许重复的列 |
|    idnex    | True：为此列创建索引 |
|  nullable   |  True：允许为空   |
|   default   |  为此列设置一个默认值  |



之后就是sqlAlchemy的使用，比如下面:

```python
from flask_sqlalchemy import SQLAlchemy
import app

db = SQLAlchemy(app)

class User(db.Model)
	__tablename__ = 'User'
  
  id = db.Column(db.INTEGER, primary_key=True, autoincrement=True)
  name = db.Column(db.String(64), index=True, nullable=False)
  _password = db.Column(db.String(128), nullable=False)
  email = db.Column(db.String(128), unique=True, index=True)
```

这样一个映射的数据表就建好了，然后就可以通过以下命令，`db.create_all()`来创建所有与之相关的真实的数据表。然后同时需要设置的是 app 对象的 config 中 `SQLALCHEMY_DATABASE_URI`中的值，如上所述。还有一点，如果是使用python命令行来执行创建数据表语句的时候，不仅要导入db，还要导入所有的映射表，如此处还需要导入User类。至于数据库的增删改，可以通过新建一个用于被继承的父类`DBModel`，如：

```python
class DBModel(object):
  
  def save(self):
    db.session.add(self)
    db.session.commit()
    
  def delete(self):
    db.session.delete(self)
    db.session.commit()
    
  def save_session(self):
    db.session.save(self)
    
  def delete_session(self):
    db.session.delete(self)

# 然后让User继承多个父类

class User(db.Model, DBModel):
  ...
  ...
```

这样的话每一个User对象就可以直接调用 save 和 delete 函数来执行数据修改了。其中`save_session`意为在事务中的操作，如果想开启事务同时处理多个操作的时候，需要调用这个方法，然后最后统一commit。如：

```python
...
try:
    db.session.commit()
catch:
    db.session.roolback()
...
```

这样就可以确保一个事务中所有的操作都是统一的。

然后关于数据查询，有以下一些信息：

|    过滤器    |            说明            |
| :-------: | :----------------------: |
|  filter   |  把一个过滤器添加到原查询上，返回一个新查询   |
| filter_by | 把一个等值过滤器添加到原查询上，返回一个新查询  |
|   limit   |  使用指定的限制返回的结果数量，返回一个新查询  |
|  offset   |     偏移原查询返回的结果，返回一个      |
| order_by  | 根据指定条件对原查询结果进行排序，返回一个新查询 |
| group_by  | 根据指定条件对原查询结构进行分组，返回一个新查询 |

|      方法      |              说明              |
| :----------: | :--------------------------: |
|    first     |    返回查询的第一个结果，如果没有返回None     |
| first_or_404 |    返回查询的第一个结果，如果没有返回404错误    |
|     get      |     返回指定主键的行，如果没有返回None      |
|  get_or_404  |     返回指定主键的行，如果没有返回404错误     |
|    count     |          返回查询结果的数量           |
|   paginate   | 返回一个 Paginate 对象，它包含指定范围内的结果 |
|     all      |         以列表形式返回所有结果          |

对于过滤器的使用，可以多层过滤器嵌套，一般的使用为：

```python
# filter_by()为一个过滤函数，参数是键值对，"key=value"
User.query.filter_by(id=2).all()
# filter()为过滤函数，参数是一个布尔表达式
User.query.filter(User.id>2).all()
# 如要选择所有id大于10的人，并按照从小到大选取10人
from sqlalchemy import asc
User.query.filter(User.id>10).order_by(asc(User.id)).limit(10).all()
```

等等，由于一个用户表会涉及到密码的存储，而行业的基本约定就是，私密信息绝不明文存储，所以上述的User表当然是不符合规定的，还要进行修改，这种加密需要用到的库可以是 flask_bcrypt。

## 关于服务器内部的数据加密

flask有一个插件，`flask_brycpt`，用于在python中的简单加密，基本使用为：

```python
>>> from flask import Flask
>>> from flask_bcrypt import Bcrypt
>>> app = Flask(__name__)
>>> bcrypt = Bcrypt(app)
>>> password = 'your password'
>>> hash_password = bcrypt.generate_password_hash(password).docode()
>>> hash_password
'$2b$12$G/yyzu7H8TU8VgGYNJ/49OhzvA5KUDJFbeneUx/yEBxk3l7CzOiT2'
>>> bcrypt.check_password_hash(password_hash, 'my password')
False
>>> bcrypt.check_password_hash(password_hash, password)
True
```

由此可见，bcrypt 能够很方便的将不适合铭文存储在服务器中的数据，如密码等，使用加密后的数据保存，之后只需要判断外来数据与本地数据是否能够匹配就可以了。与之相关的在flask配置文件中有一个配置名为`BCRYPT_LOG_ROUNDS`的变量，是用来标示bcrypt的hash次数的，这个数值越大，加密时bcrypt hash的次数就越多，密码就越难破解，使用的时间也就越久，在Bcrypt内部这个值预设的是12，其范围是4-31，也就是说，在能够尽可能保证不被破解的又能尽快得出结果的情况下，12应该是最合适的数值，如果不是对秘文有特别高的要求，这个数值能改能满足大多数的需要。

## 用于网络传输的RSA 加密

python 上的 rsa 加密一般使用的是 pycrypto 库，使用`pip install pycrypto`安装即可。

然后还需要使用openssl生成一个私匙以及与其对应的公匙，一般情况下，公匙放在客户端上用于对初始数据进行加密，然后在服务端用私匙对其进行解密，这样就不怕客户端的公匙被知道了，因为知道了也不能根据公匙来破解被公匙加密的数据。生成的方法如下：

```bash
# 1. 生成私匙
openssl genrsa -out private_key.pem 1024
# 2. 将私匙转化为PKCS8编码，作为最终使用的私匙
openssl pkcs8 -topk8 -in private_key.pem -out pkcs8_private_key.pem -nocrypt
# 3. 根据私匙生成对应的公匙
openssl rsa -in private_key.pem -out public_key.pem -pubout
```

因为rsa加密有长度限制（pycrypto 限制 256字符），所以一般只会使用rsa传输密匙，用户密码等比较隐私的数据。之后的根据此公私匙来进行对数据的加密解密：

```python
import base64
from Crypto import Random
from Crypto.Cipher import PKCS1_v1_5
from Crypto.PublicKey import RSA

from instance import config

random_generate = Random.new().read

with open(config.RSA_PUBLIC_KEY) as f:
  	pub = f.read()
with open(config.RSA_PRIVATE_KEY) as f:
  	pri = f.read()
    
    
class RsaUtil(object):
  	pri_key = ''
    pub_key = ''
    _pri_key = ''
    _pub_key = ''
    
    def __init__(self):
      	self.pri_key = RSA.importKey(pri)
        self.pub_key = RSA.importKey(pub)
        self._pri_cipher = PKCS1_v1_5.new(self.pri_key)
        self._pub_cipher = PKCS1_c1_5.new(self.pub_key)
        
    def encrypt(self, text):
      	return base64.b64encode(self._pub_cipher.encrypt(text.encode())).decode()
      
    def decrypt(self, text):
      	return self._pri_cipher.decrypt(base64.b64decode(text), random_generate).decode()
```

首先从配置文件 instance/config.py 中读取公匙和私匙存放的位置，并把其中的内容读取出来，然后建立一个加密解密类，用于统一管理rsa加密，在构造函数中，加载出来了公私匙对应的cipher，然后在后面的 encrypt 和 decrytpt 中直接调用cipher，同时可以看到，在rsa加密的同时，还进行了base64编码，因为经过rsa加密后的是一个bytes类型，而且不能直接decode() 为字符串，示例：

```python
# 假设先给 RsaUtil 加上两个函数
def encrypt_no_base64(self, text):
    return self._pub_cipher.encrypt(text.encode
                                    
def decrypt_no_base64(self, text):
    return self._pri_cipher.decrypt(text, random_)
                                    
# 这是两个不带base64编码的加密解密函数，那么运行结果如下
```

```python
>>> imoprt RsaUtil
>>> rsa = RsaUtil()
>>> text = '123'
>>> result_1 = rsa.encrypt(text)
>>> result_1
'BQqSf1taZhtWF/jY1+/QVaw8carj2mWh6KymVULpVh44cTB9KANx3AZWcB4zzQ5lZV/H9G3qCoKNIjqwNAxsNs83YcibWRksly0+tmK9zvWRUBKtf2CEj0PgVOStwJ1YTXogsQWDZaDMVEtRXG2n6MaDyQ1lIZKM8qZZJRuCBL0='
>>> rsa.decrypt(result_1)
'123'
>>> result_2 = rsa.encrypt_no_base64(text)
>>> result_2
b'U\x06\x99\xec\xca\x01J2\xbb\xd3N\xabE\x88\x9c\xee)\x0epL*\x0c~\x98*\xf0e\xbf\xc0\xe1\x82w/=\xdf\x97F\x9fYt\xbe\x01\xaa[T<=\x8f+\xef!A\xfe\x0e\xc5\x8a\xc4c \x1f;\xb7\xedG*\xf5l\x9a\x89@\xd8\x9f0\xd7c\xb14\xf6\xbe7\x84\x8a\xd2\xd6v\x99\xc3I\xa6\\\xcfR\x9aW\x98\xf5G\xf2\\a\xc5|\xf6\xf94x1F\xa9\x13\x14\xb3J\xd6v\xb2\xe2\xca\x8f\x1b\xaa\x94\xa8\x14\xa8\x02\x80\xa5'
>>> rsa.decrypt_no_base64(result_2)
b'123'
>>> result_2.decode()
result_2.decode()
Traceback (most recent call last):
  File "<input>", line 1, in <module>
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x99 in position 2: invalid start byte
>>> type(result_1)
<class 'str'>
>>> type(result_2)
<class 'bytes'>
```

如上图所示运行结果，result\_1 是一个字符串，而 result\_2 是一个bytes类型，但是在服务器环境中，还是字符串更容易传送和查看，所以选择用base64加一层编码，是其可以转化为字符串。这样在使用 RsaUtil类的时候，我们只需要传送一个字符串进去，就可以得到加密或解密后的字符串了。

## 关于邮件使用

