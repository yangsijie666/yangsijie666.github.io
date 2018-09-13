--- 
layout: post
title: python学习之sqlalchemy模块
subtitle: sqlalchemy模块的基本使用
date: 2018-07-31
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - python
---

# sqlalchemy模块

### 数据库URL

SQLAlchemy使用URL的方式来指定要访问的数据库，整个URL的具体格式如下：

```bash
dialect+driver://username:password@host:port/database
```

其中，`dialect`是指DBMS的名称，可选的值一般有：postgresql, mysql, sqlite等。

`driver`指的是驱动的名称，若不指定，SQLAlchemy会使用默认值。

`database`指的是DBMS中的一个数据库，一般是指通过CREATE DATABASE语句创建的数据库。

在openstack中通常使用的URL格式为：

```bash
mysql+pymysql://username:password@host:port/database
```

其中`pymysql`需要单独安装`python-pymysql`模块。

### 创建Engine对象

Engine实现了对各种不同的数据库客户端的封装和调度，是所有SQLAlchemy应用程序的入口点，要使用SQLAlchemy库来操作一个数据库，首先需要获得一个Engine对象，后续的所有对数据库的操作都要通过这个对象来进行。

确定了数据库的URL之后，就可以使用`create_engine`来创建Engine对象了。

```python
from sqlalchemy import create_engine

engine = create_engine('sqlite://:memory:')
```

> `sqlite://:memory:`是使用sqlite提供的内存数据库，用于测试使用

`create_engine`还支持以下几个参数：

- connect_args: 一个字典，用于自定义数据库连接的参数，如指定客户端使用的字符编码
- pool_size和max_overflow: 指定连接池的大小
- poolclass: 指定连接池的实现
- echo: 布尔值，指定是否打印执行的SQL语句到日志中

### 定义数据模型

数据库连接完成之后，就需要开始定义数据模型，即定义映射数据库表的Python类。在SQLAlchemy中，通过Declarative的系统来完成。

在官网中介绍说，映射有两种方式，一种是Classical Mapping，这种方式需要完成三个步骤:

1. 首先定义一个映射类，该类是数据库表在代码中的对象表示，该类的类属性均为Column类的实例，对应数据库中的一列属性
2. 然后定义一个Table对象，用来表示数据库中的一个表
3. 调用`sqlalchemy.orm.mapper`函数将步骤1中定义的类映射到步骤2中定义的Table

现在由于有了Declarative，我们需要做的就只有定义步骤1中的类即可。

使用Declarative的方法就是创建一个基类，之后创建的所有映射数据表的类都继承自该基类，该基类用于维护所有映射类的元信息：

```python
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
```

#### 定义映射类

现在有了该基类后，我们就可以开始创建映射数据表的类了。假设现在我们有一个person数据库，其中有一个表user，该表具有4列（属性），分别为`id`(自增、主键、非空)，`user_id`(索引、非空)，`name`(非空、唯一)，`email`(可为空)：

```python
from sqlalchemy import Column, Integer, String

# 这里的Base是由上面的declarative_base()生成的
class User(Base):
    
    __tablename__ = 'user'
    __table_args__ = (
        Index('ix_user_user_id', 'user_id'),
    )

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(String(255), nullable=False)
    name = Column(String(64), nullable=False, unique=True)
    email = Column(String(255))
```

> 其中的`Integer`、`String`对应的数据库中的类型分别为`INT(11)`、`VARCHAR`

### Schema和Metadata

Schema metadata在官方文档中也被称为database metadata，简称为metadata，是一个容器，其中包含了和DDL相关的所有信息，包括Table、Column等对象。当SQLAlchemy根据数据表映射类生成SQL语句时，它会查询metadata中的信息，然后根据信息生成SQL语句。

当使用Classical Mapping的方式，我们需要先创建一个metadata实例，然后每次创建一个Table对象的时候就把metadata传递进去。对于用户而言，metadata提供的接口仅仅用于创建和删除表而已，因此该方法几乎用不到。

因此Declarative系统在使用`declarative_base()`生成一个基类Base的时候，该基类就已经包含了一个metadata实例，后面基于该Base定义的映射类都会被自动加入到这个metadata中，我们可以通过`Base.metadata`来访问这个metadata。

下面就来看看我们一般会用这个metadata做什么：

```python
Base = declarative_base()

# 这里写上基于Base定义的数据表映射类

Base.metadata.create_all(engine)
```

看到`create_all`，大概你也能猜到了，没错就是用于根据映射类创建表的操作，它会生成对应所有表的`CREATE TABLE`语句，并且通过engine发送到数据库上执行，可以通过以下代码进行测试：

```python
from sqlalchemy import create_engine
from sqlalchemy import Column, Integer, String, Index
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    
    __tablename__ = 'user'
    __table_args__ = (
        Index('ix_user_user_id', 'user_id'),
    )

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(String(255), nullable=False)
    name = Column(String(64), nullable=False, unique=True)
    email = Column(String(255))

    
engine = create_engine('mysql+pymysql://person:123456@localhost/person')

Base.metadata.create_all(engine)
```

该代码会在数据库person中创建一个user表

### 会话(session)

session使我们通过SQLAlchemy来操作数据库的入口。其功能是管理程序和数据库之间的会话，利用engine的连接管理功能来实现会话。使用session可以实现对数据库的增、删、改、查。

要想使用session，我们需要通过`sessionmaker`函数创建一个session类，然后通过这个类的实例来使用会话：

```python
from sqlalchemy.orm import sessionmaker

DBSession = sessionmaker(bind=engine)
session = DBSession()
```

通过`sessionmaker`的bind参数将Engine对象传递给`DBSession`去管理。接着`DBSession`实例化的对象`session`就能被我们用来进行增、删、改、查了。

### 增、删、改、查

增删改查是SQLAlchemy中最常用的几个功能，并且都是由上一节中的`session`对象实现的

#### Create

在数据库中插入一条记录，是通过`session.add()`方法来实现的，然后调用`session.commit()`来提交事务：

```python
import uuid

new_user = User(user_id=str(uuid.uuid4()), name='new_user', email='new_user@example.com')
session.add(new_user)
session.commit()
```

### Delete

在数据库中删除一条记录，是通过`session.delete()`实现的。当然也可以先使用`session.query()`查询到需要删除的记录，再调用`session.delete()`进行删除：

```python
new_user = session.query(User).filter(User.name=='new_user').one()
session.delete(new_user)
session.commit()
```

### Update

更新一条记录，需要先使用`session.query()`获得一条记录对应的对象，然后修改对象的属性，再用`session.add()`方法来完成更新操作。

```python
jeck = session.query(User).filter(User.name=='jeck').one()
jeck.name = 'alice'		# 将该记录的name字段修改为'alice'
session.add(jeck)
session.commit()
```

#### Read

查询一条记录，通过`session.query()`方法创建一个query对象，再通过调用该对象的一系列方法（如`one()`、`first()`、`all()`等方法）来完成查询操作。

```python
# 使用多个filter进行筛选，相当于SQL中的and。返回值为一个User对象，这是由query中的'User'决定的
session.query(User).filter(User.name=='jeck').filter(User.email=='jeck@example.com').one()

# 使用all可以查找所有的User对象，返回值为一个由User对象组合而成的列表
session.query(User).all()
```

更多的关于`query`的使用，可以查看其[官方地址](http://docs.sqlalchemy.org/en/rel_1_0/orm/tutorial.html#querying)

### 事务

当用户调用一个session的方法，导致session执行一条SQL语句时，它就会自动开始一个事务，直到下次调用`session.commit()`或者`session.rollback()`，它就会结束这个事务。

```python
jeck = session.query(User).filter(User.name=='jeck').one()
jeck.name = 'alice'		# 将该记录的name字段修改为'alice'
session.add(jeck)
session.rollback()		# 在commit之前调用rollback，则名字的修改不会写入数据库中
```

> `session.rollback()`必须在`session.commit()`之前调用，否则无效

或者也可以显示调用`session.begin()`来开始一个事务，并且其可以配合Python的`with`来使用。

---

### 数据库版本管理

之前，我们是使用`create_engine`，将所有需要的数据表都创建一次的方式来创建数据表的，但是这种方式不好的就是每次执行这句话，就会重新创建一次数据表，并且之前的数据也会消失。

可能有人会说，可以在运行项目代码之前，先手动将所有的数据表都建好。但是这样子是比较麻烦的，原因就不多说了。

为了解决这个问题，就出现了'数据库版本管理'的概念，称为'Database Migration'。可以理解为，我们可以通过代码指定各个版本的数据库中的表是什么样子的，当我们都定义完成后，只需要执行migration操作，则会将数据表升级为目标版本的样子，打个比方：

1. 在version 1中，在数据库中建立一个user表
2. 在version 2中，在数据库中建立一个project表
3. 在version 3中，在user表中增加一个age列

如果我们现在的版本是version 1，当设定的目标版本是version 3时，操作就是：建立一个project表，在user表中添加age列，并将当前数据库版本设置为version 3。

我们这里使用`Alembic`实现版本管理。

#### Alembic

要使用alembic，需要以下步骤：

1. 安装Alembic
2. 在项目中创建Alembic的migration环境
3. 修改Alembic配置文件
4. 创建migration脚本
5. 执行迁移动作

##### 安装Alembic：

该步骤很简单，采用pip安装即可：

```bash
pip install alembic
```

##### 在项目中创建Alembic的migration环境：

在openstack项目中，一般Alembic的环境都是放在**db/splalchemy/**目录下，因此，我们首先进入webdemo2/db/sqlalchemy/，然后在该目录下初始化Alembic：

```bash
alembic init alembic
```

执行完成后，在webdemo2/db/sqlalchemy/下会生成以下文件：

```
(venv) root@yangsijie-PC:~/PycharmProjects/webdemo_ysj/webdemo2/db/sqlalchemy# tree .
.
├── alembic
│   ├── env.py
│   ├── README
│   ├── script.py.mako
│   └── versions
└── alembic.ini
```

##### 修改Alembic配置文件：

修改webdemo2/db/sqlalchemy/alembic.ini文件，主要是修改其中的`sqlalchemy.url`，将其指向我们需要使用的数据库：

```ini
sqlalchemy.url = mysql+pymysql://person:123456@localhost/person
```

##### 创建migration脚本：

在创建migration脚本方面，共有两种方式：

- 自动创建一个空的migration脚本，在其中手动添加具体操作
- 根据models.py中的模型，自动创建migration脚本并生成具体操作

下面具体解释一下两种方式：

1. 在webdemo2/db/sqlalchemy下执行以下语句：

   ```bash
   alembic revision -m "Create user table"
   ```

   > `-m`后面跟的是注释，随意填写即可

   执行完后，可以发现versions中自动生成了一个脚本`ca888fa021cb_create_user_table.py`。现在我们需要在其中手动添加创建user表的操作：

   ```python
   """Create user table
   
   Revision ID: ca888fa021cb
   Revises: 
   Create Date: 2018-07-23 11:03:48.380526
   
   """
   from alembic import op
   import sqlalchemy as sa
   
   
   # revision identifiers, used by Alembic.
   revision = 'ca888fa021cb'
   down_revision = None
   branch_labels = None
   depends_on = None
   
   
   def upgrade():
       op.create_table(
           'user',
           sa.Column('id', sa.Integer, primary_key=True, autoincrement=True),
           sa.Column('user_id', sa.String(255), nullable=False),
           sa.Column('name', sa.String(64), nullable=False, unique=True),
           sa.Column('email', sa.String(255))
       )
   	op.create_index('ix_user_user_id', 'user', ['user_id'], unique=True)
   
   def downgrade():
       op.drop_table('user')
   ```

   我们可以继续执行上述命令，创建新的migration脚本，这里就不做更多的解释了，因为流程是跟上面一模一样的。

2. 首先需要在webdemo2/db/sqlalchemy/alembic/env.py中添加以下修改：

   ```python
   import os.path
   import sys
   
   sys.path.append(os.path.realpath('./..'))
   import models
   
   target_metadata = models.Base.metadata
   ```

   > `models.py`是放在webdemo2/db/下的

   接着，对models.py文件中的数据表的映射类做任何修改或是创建表的映射类，这里就不多解释了，与之前说的定义方式是一模一样的。

   定义或是修改完后，需要在webdemo2/db/sqlalchemy/下执行以下语句，使得根据models.py文件自动生成migration脚本，并添加具体操作：

   ```bash
   alembic revision --autogenerate -m "add index of user_id in table user"
   ```

   做完此步后，会在versions目录下生成一个新的文件`4573acea8246_add_index_of_user_id_in_table_user.py`，并且该文件中的具体操作已经被写好了，检查无误即可。

##### 执行迁移动作：

在webdemo2/db/sqlalchemy/下执行以下命令，即可执行迁移动作：

```bash
alembic upgrade head
```

执行完此句后，登录数据库就可以发现数据库已经发生了变化。

---

### 案例：

该案例是接着Pecan框架那篇文档中的案例，在其基础上添加了数据库的操作，将数据库的操作作为hook，融入Pecan框架中。

添加完数据库的架构图如下图所示：

```bash
webdemo2
├── api
│   ├── app.py
│   ├── config.py
│   ├── controllers
│   │   ├── __init__.py
│   │   ├── root.py
│   │   └── v1
│   │       ├── controller.py
│   │       ├── __init__.py
│   │       └── users.py
│   ├── expose.py
│   ├── hooks.py
│   └── __init__.py
├── cmd
│   ├── api.py
│   └── __init__.py
├── db
│   ├── api.py
│   ├── __init__.py
│   ├── models.py
│   └── sqlalchemy
│       ├── alembic
│       │   ├── env.py
│       │   ├── README
│       │   ├── script.py.mako
│       │   └── versions
│       │       ├── 016a6017da95_create_project_table.py
│       │       ├── 4573acea8246_add_index_of_user_id_in_table_user.py
│       │       ├── 53b9ae3402c7_add_age_to_user.py
│       │       ├── 7fa4861c26fa_add_index_of_user_id_in_table_user.py
│       │       └── ca888fa021cb_create_user_table.py
│       └── alembic.ini
└── __init__.py
```

相比于之前，增加了db目录以及hooks.py文件。

#### db/models.py:

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext import declarative
from sqlalchemy import Index


Base = declarative.declarative_base()


class User(Base):

    __tablename__ = 'user'
    __table_args__ = (
        Index('ix_user_user_id', 'user_id'),
    )

    id = Column(Integer, primary_key=True, autoincrement=True)
    user_id = Column(String(255), nullable=False)
    name = Column(String(64), nullable=False, unique=True)
    email = Column(String(255))
```

> 定义了user数据表的映射类

#### db/api.py:

```python
from sqlalchemy import create_engine
import sqlalchemy.orm
from sqlalchemy.orm import exc

from webdemo2.db import models as db_models


_ENGINE = None
_SESSION_MAKER = None


def get_engine():
    global _ENGINE
    if _ENGINE is not None:
        return _ENGINE

    _ENGINE = create_engine('sqlite:///../../webdemo2.db')
    # db_models.Base.metadata.create_all(_ENGINE)	由于假设数据库已经创建完成，其中的表的创建由数据库版本控制完成，因此此处无需另外创建表
    return _ENGINE


def get_session_maker(engine):
    global _SESSION_MAKER
    if _SESSION_MAKER is not None:
        return _SESSION_MAKER

    _SESSION_MAKER = sqlalchemy.orm.sessionmaker(bind=engine)
    return _SESSION_MAKER


def get_session():
    engine = get_engine()
    maker = get_session_maker(engine)
    session = maker()

    return session


class Connection(object):

    def __init__(self):
        pass

    def get_user(self, user_id):
        query = get_session().query(db_models.User).filter_by(user_id=user_id)
        try:
            user = query.one()
            return user
        except exc.NoResultFound:
            print ('No Result Found.')
            return None

    def list_users(self):
        session = get_session()
        query = session.query(db_models.User)
        users = query.all()

        return users

    def update_user(self):
        pass

    def delete_user(self):
        pass
```

#### api/hooks.py:

```python
from pecan import hooks
from webdemo2.db import api as db_api


class DBHook(hooks.PecanHook):

    def before(self, state):
        state.request.db_conn = db_api.Connection()
```

#### api/app.py:

```python
import pecan
from webdemo2.api import config as api_config
from webdemo2.api import hooks


def get_pecan_config():
    filename = api_config.__file__.replace('.pyc', '.py')   # get the absolute path of the pecan config.py
    return pecan.configuration.conf_from_file(filename)


def setup_app():      # the main functhing, start listening
    config = get_pecan_config()

    app_hooks = [hooks.DBHook()]	# 添加数据库操作的hook
    app_conf = dict(config.app)
    app = pecan.make_app(
        app_conf.pop('root'),
        logging=getattr(config, 'logging', {}),
        hooks = app_hooks,			# 添加数据库操作的hook
        **app_conf)

    return app
```

#### api/controllers/v1/users.py:

```python
from wsme import types as wtypes
from pecan import rest, request
from webdemo2.api.expose import expose as wsexpose
import pecan


class User(wtypes.Base):
    id = int
    user_id = wtypes.text
    name = wtypes.text
    email = wtypes.text


class Users(wtypes.Base):
    users = [User]


class UsersController(rest.RestController):

    # HTTP GET /users/
    @wsexpose(Users)
    def get(self):
        # 获取Connection实例（api/hooks.py中将该实例赋给了request）
        db_conn = request.db_conn
        users = db_conn.list_users()
        users_list = []
        for user in users:
            u = User()
            u.id = user.id
            u.user_id = user.user_id
            u.name = user.name
            u.email = user.email
            users_list.append(u)
        return Users(users=users_list)

    # HTTP POST /users
    @wsexpose(None, body=User, status_code=201)
    def post(self, user):
        print user

    @pecan.expose()
    def _lookup(self, user_id, *remainder):
        return UserController(user_id), remainder


class UserController(rest.RestController):

    _custom_actions = {
        'kill': ['POST']
    }

    def __init__(self, user_id):
        self.user_id = user_id

    # HTTP GET /users/123456/
    @wsexpose(User)
    def get(self):
        db_conn = request.db_conn
        user = db_conn.get_user(self.user_id)
        if user is None:
            raise wsme.exc.ClientSideError('user_id: \'%s\' is not exist.' % self.user_id, status_code=403)
        u = User()
        u.id = user.id
        u.user_id = user.user_id
        u.name = user.name
        u.email = user.email
        return u

    # HTTP PUT /users/123456/
    @wsexpose(User, body=User)
    def put(self, user):
        user_info = {
            'id': self.user_id,
            'name': user.name,
            'age': user.age + 1
        }
        return User(**user_info)

    # HTTP DELETE /users/123456/
    @wsexpose()
    def delete(self):
        print ('Delete user_id: %s' % self.user_id)

    # HTTP POST /users/123456/kill
    @wsexpose(status_code=202)
    def kill(self):
        print ('Kill user_id: %s' % self.user_id)
```

> 相比之前的代码，修改了两处地方：
>
> - 修改了UsersController中的get()方法，调用Connection对象的list_users()方法
> - 修改了UserController中的get()方法，调用Connection对象的get_user()方法

该案例可以从我的github上下载到，地址是：https://github.com/Seanybalabala/webdemo2