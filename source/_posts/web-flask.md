---
title: 《Flask Web》Review
date: "2019/7/12"
categories:
- unclassified
---

### 概述

Flask是一个轻量且灵活的Python Web框架。[《Flask Web开发》](https://book.douban.com/subject/26274202/)介绍了用Flask开发Web的基本实践，讲解非常清晰流畅，很适合作为入门的读物。本文作为该书的提纲，并提供示例代码，为以后开发提供快速参考的资料。

一个最简单的Flask App如下：
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"
    
if __name__ == '__main__':
    app.run(debug=True)
```

### 基础

#### 基本概念

* 视图函数
* 路由
    1. `@app.route("/index")`
    2. 静态路由 `/static/filename`
* 上下文全局变量
    * 程序上下文：current_app(当前激活的程序实例）, g（临时存储的全局变量，每次请求重置）
    * 请求上下文：request, session（用于存放Cookie等）
* 请求钩子
    * `before_first_request`
    * `before_request`
    * `teardown_request`
* 响应
    * 视图函数的返回值`data[, status_code[, headers]]`
    * `make_response()`
    * `redirect()`
    * `abort(error_code)`
* 自定义错误处理函数：不进入视图函数，返回定制的模板
```python
@app.error_handler(404)
def page_not_found():...
```
```python
@app.error_handler(500)
def internal_server_error:...
```
* `url_for('index', page=2)`返回动态的地址`/index?page=2`
* `url_for('static', filename=...)`返回静态文件地址
* flash消息
    * flash()
    * 渲染flash消息`get_flashed_messages()`
* config.py文件结构

```python
class Config:
... # 通用配置
class DevelopmentConfig(Config):
... # 开发环境配置
class TestingConfig(Config):
... # 测试环境配置
class ProductionConfig(Config):
... # 生产环境配置
config = {
"development": DevelopmentConfig,
"testing": TestingConfig,
"production": ProductionConfig,
"default": DevelopmentConfig
}

```

* 配置 `app.config.from_object(config[config_name])`
* app工厂函数（一般在app/__init__.py文件中）

```python
def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    
    # 初始化flask扩展    
    
    # 设置路由和错误处理函数
    
    return app
```

* 蓝图
main蓝图，在app/main/__init__.py中

```python
from flask import Blueprint
main = Blueprint("main", __name__)
from . imort views, errors
```

注：Blueprint构造函数第一个参数是包名/蓝图名，第二个参数一般用__name__

蓝图的路由

```python
@main.route("/")
@main.error_handler(404)
```

注册蓝图（在app工厂函数中）
```python
from main import main as main_blueprint
app.register_blueprint(main_blueprint)
```

#### Flask-Script 支持命令

```python 
from flask_script import Manager
manager = Manager(app)
# ...
if __name__ == '__main__':
    manager.run()
```

* 默认命令: `runserver`, `shell`
* 为`shell`命令提供一个上下文
```python
def make_shell_context():
    return dict(app=app, db=db)
manager.add_command("shell", Shell(make_context=make_shell_context))
```

#### Jinja2和Flask-Bootstrap渲染HTML

* 实践：将表现层逻辑放在模版中
* 模板放在templates子文件夹中
* `render_template(template_filename, VAR=...)`
* 变量: `{{VAR[|[safe|...]}}`
* 控制结构: 判断、循环、宏
```html
{% if CONDITION %}
    ...
{% else %}
    ...
{% endif %}
```
```html
{% for ... in ... %}
    ...
{% endfor %}
```
```html
{% macro MACRO_DEF(...) %}
    ...
{% endmacro %}
{{ MACRO_DEF(...) }}
```
```html
{% include 'common.html' %}
```

base.html
```html
<html>
<head>
    {% block head %}
<title>{% block title %}{% endblock %} - My Application</title>
    {% endblock %}
</head>
<body>
    {% block body %}
    {% endblock %}
</body>
</html>
```
```html
{% extends "base.html" %}
{% block title %}Index{% endblock %}
{% block head %}
{{ super() }}
<style>
</style>
{% endblock %}
{% block body %}
<h1>Hello, World!</h1>
{% endblock %}
```
* super()函数：向已有的块(block)添加
* Flask-Bootstrap: 提供了bootstrap/base.html的模板

* 注入变量
```python
@main.app_context_processor
def inject_permissions():
    return dict(Permission=Permission)
```

#### Flask-Moment处理本地化时间

* 引入flask_moment包
```python
from flask_moment import Moment
moment = Moment(app)
# ...
```
* 在base.html引入moment.js库
```html
{% block scripts %}
{% super %}
{% include moment.include_moment() %}
{% endblock %}
```

#### Flask-WTF处理Web表单

* 跨站请求伪造保护（CSRF）
```python
app.config['SECRET_KEY'] = ...
```
* 表单类
```python
class MyForm(Form):
    name = StringField('What's your name', validator=[Required()])
    submit = SubmitField('Submit')
```

* 数据验证
    + 为表单类属性添加`validators`属性
    + 在表单类中添加`validate_*`方法

* 渲染表单`render_template(template_filename, form=form)`

手动渲染

```html
<form method="POST">
{{form.hidden_tag()}}
{{form.name.label}} {{form.name(id=...)}}
{{form.submit()}}
</form>
```

用Bootstrap模板渲染

```python
{% import "bootstrap/wtf.html" as wtf %}
{{ wtf.quick_form(form) }}
```

在视图函数中处理表单

```python
@app.route("/", methods=["POST", "GET"])
def index():
    name = None
    form = MyForm()
    if form.validate_on_submit():
        name = form.name.data
        form.name.data = ""
    return render_template("index.html", form=form, name=name)
```

注：POST和GET由同一个视图函数处理，第一次连接GET获取页面，POST到同一个页面

* POST/重定向/GET模式
解决：若POST是最后一个请求，则刷新页面会提示重新提交表单
方法：POST的响应重定向到一个GET请求

#### Flask-SQLAlchemy处理数据库

* 配置SQLALCHEMY_DATABASE_URI

```python
mysql://username:password@hostname/database
sqlite:////absolute/path/to/database
```

* 定义模型，db.Model的子类
    * `__tablename__`定义表名
    * 定义字段：`db.Column`类的实例，字段类型和字段选项
    * 定义关系：`db.relationship`类的实例

```python
class User:
    __tablename__ = "user"
    role_id = db.Column(db.Integer, db.ForeignKey("roles.id"))
class Role:
    __tablename__ = "role"
    users = db.relationship("User", backref="role")
```

注：role_id是真实存在的字段，users则不是；`db.ForeignKey()`的第一个参数定义外键连接roles表的id字段；db.relationship()的第一个参数定义关系另一端是哪一个模型，backref参数为User模型添加一个role字段，直接访问Role模型

* 创建表/删除表

```python
db.create_all()
db.drop_all()
```

* 插入行/修改行/删除行

```python
admin_role = Role(name="admin")
db.session.add(admin_role)
db.session.commit()
admin_role.name = "Administrator"
db.session.add(admin_role)
db.session.commit()
db.session.delete(admin_role)
db.session.commit()
```

* 查询

```python
User.query.all()
User.query.filter_by(role=...).all()
User.query.get_or_404(id)
```

* 事件监听

```python
db.event.listen(Post.body, 'set', Post.on_changed_body)
```

* 多对多关系

```python
registrations = db.Table(
    'registrations',
    db.Column(
        'student_id', 
        db.Integer, 
        db.ForeignKey('students.id')
    ),
    db.Column(
        'class_id', 
        db.Integer, 
        db.ForeignKey('classes.id')
    )
)
class Student(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String)
    classes = db.relationship('Class',
    secondary=registrations,
    backref=db.backref('students', lazy='dynamic'), lazy='dynamic')

class Class(db.Model):
    id = db.Column(db.Integer, primary_key = True)
    name = db.Column(db.String)
``` 

```python
class Follow(db.Model):
    follower_id = db.Column(db.Integer, db.ForeignKey('users.id'), primary_key=True)
    followed_id = db.Column(db.Integer, db.ForeignKey('users.id'), primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow)
    
class User(db.Model):
    followed = db.relationship(
        'Follow', 
        foreign_keys=[Follow.follower_id], 
        backref = db.backref('follower', lazy='joined'),
        lazy = 'dynamic',
        cascade = 'all, delete-orphan'   
    )
    followers = db.relationship(
        'Follow', 
        foreign_keys=[Follow.followed_id],
        backref = db.backref('followee', lazy='joined'),
        lazy = 'dynamic',
        cascade = 'all, delete-orphan'    
    )
```

注1`lazy = 'dynamic'`表示`followed`/`followers`属性返回一个查询(`query`)而不是查询结果；
注2`lazy='joined'`表示立即从联结查询中加载对象，如`user.followed.all()`，`followed`返回的是查询(`dynamic`)，`all`返回所有`Follow`对象，每一个对象的`followed`和`followee`查询以及被加载；如果是默认值`select`，对象的`followed`和`followee`查询不会被加载；
注3`cascade = 'all, delete-orphan'`配置在该对象操作会对相关对象造成的影响，这里设置成该对象被删除时，所有指向该对象的对象都删除。

* 联结查询

```python
db.session.query(Post).select_from(Follow).filter_by(follower_id=self.id).join(Post, Follow.followed_id == Post.author_id)
```

注1`db.session.query(Post)`: 查询结果返回Post对象
注2`select_from(Follow)`: 从Follow表开始查询
注3`join(Post, Follow.followed_id == Post.author_id)`: 联结Post表，联结的条件是`Follow.followed_id == Post.author_id`

 
#### Flask-Migrate实现数据库迁移

* 添加迁移命令

```python
from flask_migrate import Migrate, MigrateCommand
migrate = Migrate(app, db)
manager.add_command('db', MigrateCommand)
```

* 使用迁移命令

```shell
python manage.py db init // 迁移仓库初始化
python manage.py db migrate -m "..." // 创建迁移脚本
python manage.py db upgrade // 更新数据库
```

#### Flask-Mail提供电子邮件支持
Flask-Mail连接到SMTP服务器或localhost:25（默认），将邮件提交发送

* 配置SMTP服务器
MAIL_SERVER
MAIL_PORT
MAIL_USE_TLS
MAIL_USE_SSL
MAIL_USERNAME
MAIL_PASSWORD

注：敏感信息如MAIL_USERNAME，MAIL_PASSWORD最好通过环境变量读入

* 邮件接口 

```python
msg=Message("text subject", sender="me@email.com", recipients=["someone@email.com"])
msg.body = ...
msg.html = ...
with app.app_context():
    mail.send(msg)
```

* 异步发送邮件

```python
def send_email_async(app, msg):
    with app.app_context():
        mail.send(msg)

def send_email(to, subject, template, **kwargs):
    msg = Message(current_app.config['FLASK_MAIL_SUBJECT_PREFIX'] + subject, sender=current_app.config['FLASK_MAIL_SENDER'], recipients=[to])
    msg.body = render_template(template+'.txt', **kwargs)
    msg.html = render_template(template+'.html', **kwargs)
    thr = Thread(target=send_email_async, args=[current_app, msg])
    thr.start()
    return thr
```   

#### 程序结构

```
.
├── app
│   ├── __init__.py
│   ├── email.py
│   ├── main
│   │   ├── __init.py
│   │   ├── errors.py
│   │   ├── forms.py
│   │   └── views.py
│   ├── models.py
│   ├── static
│   └── templates
├── config.py
├── manage.py
├── migrations
├── requirements
│   ├── common.txt
│   ├── dev.txt
│   ├── docker.txt
│   ├── heroku.txt
│   └── prod.txt
├── requirements.txt
└── tests
└── __init__.py
```

#### 测试

##### 单元测试 + Flask Script 添加测试命令
* 编写单元测试

```python
import unittest
from flask import current_app
from app import create_app, db

class BasicTestCase(unittest.TestCase):
    def setUp(self):
        self.app = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        db.create_all()
        
        ...
    
    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()
        
        ...
    
    def test_...():
        ...
```

* 添加tests命令

```python
@manager.command
def test():
    import unittest
    tests = unittest.TestLoader().discover("tests")
    unittest.TextTestRunner(verbosity=2).run(tests)
```

* 运行所有单元测试

```shell
python manage.py tests
```

##### 测试客户端

```python
self.client = self.app.test_client(use_cookies=True)
```

##### ForgeryPy库生成随机数据

#### 部署

### 实例：社交博客

#### 登录、注册、以及邮箱确认

* Flask-Login: 管理登录
* 存储密码的散列值：Werkzeug
* 邮箱确认：用itsdangerous生成令牌
* POST/重定向/GET模式

#### 用户资料以及资料编辑页面

* 用户权限（用户，协管，管理员）：一对多的数据模型，Role<-+User
* 用户头像：gavatar头像管理服务
* 用户级/管理员级资料编辑页面

#### 博客文章以及文章编辑器

* 一对多的数据库模型，User<-+Post
* 分页导航, Pagination对象
* markdown->HTML: markdown-服务器端; flask-pagedown-客户端
* 传输的是markdown数据，在服务器转html后存在数据库（安全）

#### 关注功能

* 多对多的数据库模型，自引用关系：User<->Follow<->User
* 分页显示关注列表
* 联结查询：Follow->Post

#### 评论功能

* 一对多的数据库模型，User<-+Comment+->Post

### 实例：REST服务器

* 基于令牌的认证：Flask-HTTPAuth
* JSON序列化
* 分页大型资源