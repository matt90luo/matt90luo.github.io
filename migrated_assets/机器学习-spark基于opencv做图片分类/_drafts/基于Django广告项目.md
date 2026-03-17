---
title: 基于Django广告项目
tags:
---

##  Django命令以及相关解释
1.  python manage.py startapp account  commet:初始化一个app应用

2. python manage.py rumserver 0.0.0.0:8000 --settings=newads.settings.base comment: django 运行命令 运行时指定settings


## Django urls路由 

实际上Django提倡项目有个根urls.py，各app下分别有自己的一个urls.py，既集中又分治，是一种解耦的模式

1. 决定要使用的根URLconf模块。通常，这是ROOT_URLCONF设置的值，但是如果传入的HttpRequest对象具有urlconf属性（由中间件设置），则其值将被用于代替ROOT_URLCONF设置。通俗的讲，就是你可以自定义项目入口url是哪个文件！
2. 加载该模块并寻找可用的urlpatterns。 它是django.urls.path()或者django.urls.re_path()实例的一个列表。
3。 依次匹配每个URL模式，在与请求的URL相匹配的第一个模式停下来。也就是说，url匹配是从上往下的短路操作，所以url在列表中的位置非常关键。
4. 导入并调用匹配行中给定的视图，该视图是一个简单的Python函数（被称为视图函数）,或基于类的视图。 视图将获得如下参数:
    一个HttpRequest 实例。
    - 如果匹配的表达式返回了未命名的组，那么匹配的内容将作为位置参数提供给视图。
    - 关键字参数由表达式匹配的命名组组成，但是可以被django.urls.path()的可选参数kwargs覆盖。
5. 如果没有匹配到任何表达式，或者过程中抛出异常，将调用一个适当的错误处理视图。

Django内置下面的路径转换器: str, slug, path . 也可以自定义path转换器 写完类后，在URLconf中注册，并使用它，如下所示，注册了一个yyyy：
```python
from django.urls import register_converter, path

from . import converters, views

register_converter(converters.FourDigitYearConverter, 'yyyy')

urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<yyyy:year>/', views.year_archive),
    ...
]
```
请求的URL被看做是一个普通的Python字符串，URLconf在其上查找并匹配。进行匹配时将不包括GET或POST请求方式的参数以及域名。

URLconf不检查使用何种HTTP请求方法，所有请求方法POST、GET、HEAD等都将路由到同一个URL的同一个视图。在视图中，才根据具体请求方法的不同，进行不同的处理。
6. 有一个小技巧，我们可以指定视图参数的默认值。
```python
from django.urls import path

from . import views

urlpatterns = [
    path('blog/', views.page),
    path('blog/page<int:num>/', views.page),
]

# View (in blog/views.py)
def page(request, num=1):
    # Output the appropriate page of blog entries, according to num.
    ...
```
在上面的例子中，两个URL模式指向同一个视图views.page。但是第一个模式不会从URL中捕获任何值。 如果第一个模式匹配，page()函数将使用num参数的默认值"1"。 如果第二个模式匹配，page()将使用捕获的num值。
**这边就要弄清楚 put方法传过来的num**
7. 自定义错误页面
当Django找不到与请求匹配的URL时，或者当抛出一个异常时，将调用一个错误处理视图。Django默认的自带的错误视图包括400、403、404和500，分别表示请求错误、拒绝服务、页面不存在和服务器错误。它们分别位于

## 路由转发
通常，我们会在每个app里，各自创建一个urls.py路由模块，然后从根路由出发，将app所属的url请求，全部转发到相应的urls.py模块中。例如，下面是Django网站本身的URLconf节选。 它包含许多其它URLconf
```python
from django.urls import include, path

urlpatterns = [
    # ... 省略...
    path('community/', include('aggregator.urls')),
    path('contact/', include('contact.urls')),
    # ... 省略 ...
]
```
路由转发使用的是include()方法，需要提前导入，它的参数是转发目的地路径的字符串，路径以圆点分割。 
**这里就要注意路径问题是不是用一个相对路径指向对应urls模块**
每当Django 遇到include()时，它会去掉URL中匹配的部分并将剩下的字符串发送给include的URLconf做进一步处理，也就是转发到二级路由去。
另外一种转发其它URL模式的方式是使用一个path()实例的列表

目的地URLconf会收到来自父URLconf捕获的所有参数

向视图传递额外的参数**有时候需要向视图views传递额外的参数**
URLconfs具有一个钩子（hook），允许你传递一个Python字典作为额外的关键字参数给视图函数
```python
from django.urls import path
from . import views

urlpatterns = [
    path('blog/<int:year>/', views.year_archive, {'foo': 'bar'}),
]
```
在上面的例子中，对于/blog/2005/请求，Django将调用views.year_archive(request, year='2005', foo='bar')。理论上，你可以在这个字典里传递任何你想要的传递的东西。但是要注意，URL模式捕获的命名关键字参数和在字典中传递的额外参数有可能具有相同的名称，这会发生冲突，要避免。
**归纳总结url如何传递视图参数**

注意，只有当你确定被include的URLconf中的每个视图都接收你传递给它们的额外的参数时才有意义，否则其中一个以上视图不接收该参数都将导致错误异常


## 反向解析url 
个人认为 这是为前端服务好的 让前端使用更自然方便 Django 模板
在实际的Django项目中，经常需要获取某条URL，为生成的内容配置URL链接。比如，我要在页面上展示一列文章列表，每个条目都是个超级链接，点击就进入该文章的详细页面。
现在我们的urlconf是这么配置的：
```python
path('post/<int:pk>/'，views.some_view),
```
在前端中，这就需要为HTML的<a>标签的href属性提供一个诸如http://www.xxx.com/post/3/的值 但是前端一定不能硬编码这些
我们需要一种安全、可靠、自适应的机制，当修改URLconf中的代码后，无需在项目源码中大范围搜索、替换失效的硬编码URL。通过这个name参数，可以反向解析URL、反向URL匹配、反向URL查询或者简单的URL反查。
在需要解析URL的地方，对于不同层级，Django提供了不同的工具用于URL反查：
- 在模板语言中：使用url模板标签。(也就是写前端网页时）
- 在Python代码中：使用reverse()函数。（也就是写视图函数等情况时）
- 在更高层的与处理Django模型实例相关的代码中：使用get_absolute_url()方法。(也就是在模型model中)


其中，起到核心作用的是我们通过name='news-year-archive'为那条url起了一个可以被引用的名称
2. URL命名空间可以保证反查到唯一的URL，即使不同的app使用相同的URL名称。
第三方应用始终使用带命名空间的URL是一个很好的做法。类似地，它还允许你在一个应用有多个实例部署的情况下反查URL。 换句话讲，因为一个应用的多个实例共享相同的命名URL，命名空间提供了一种区分这些命名URL 的方法。













## 项目记录
数据库设计不会变~未来部署可能也在原来环境(但是这个不一定 反正不影响一期开发), 需要考虑一个数据同步自旋锁问题
解决的问题都是如何使用算法离线数据(一致性)
考虑算法离线数据不大的情况下 整体load进内存


外键取值规则：空值或参照的主键值
(1)插入非空值时，如果主键值中没有这个值，则不能插入。
(2)更新时，不能改为主键表中没有的值。
(3)删除主键表记录时，可以在建外键时选定外键记录一起联删除还是拒绝删除。
(4)更新主键记录时，同样有级联更新和拒绝执行的选择。
简而言之，SQL的主键和外键就是起约束作用。
关系型数据库中一条记录中有若干个属性，若其中某一个属性组（注意是组）能唯一标识一条记录，该属性就可以成为一个主键。
外键用于与另一张表相关联。是能确认另一张表记录的字段，用于保持数据的一致性。比如，A表中的一个字段，是B表的主键，那它就可以是A表的外键。


SQLite数据库中一个特殊的名叫 SQLITE_MASTER 上执行一个SELECT查询以获得所有表的索引。每一个 SQLite 数据库都有一个叫 SQLITE_MASTER 的表， 它定义数据库的模式。 SQLITE_MASTER 表看起来如下：
CREATE TABLE sqlite_master (
type TEXT,
name TEXT,
tbl_name TEXT,
rootpage INTEGER,
sql TEXT
);
对于表来说，type 字段永远是 ‘table’，name 字段永远是表的名字。所以，要获得数据库中所有表的列表， 使用下列SELECT语句：

SELECT name FROM sqlite_master
WHERE type=’table’
ORDER BY name;


RBAC（Role-Based Access Control）——基于角色的访问控制


建立项目, 跑通本地pycharm + Django, 跑通本地开发测试, 以及本地部署, 想办法抓取所有接口, 迁移接口至新的项目 
要求文档齐备池 测试充分


```python
company_name = models.CharField(max_length=255, null=False, default='', help_text='公司名称')
    account_name = models.CharField(max_length=255, null=False, unique=True, default='', help_text='用户名')
    group = models.ForeignKey(Group, null=True, on_delete=models.CASCADE, related_name='ad_accounts')
    cooperation_status = models.SmallIntegerField(null=False, default=1, help_text='1:合作中 2:解除合作')
    store_id = models.IntegerField(null=False, default=0, help_text='POP商家store id')
    store_code = models.CharField(max_length=50, null=False, default='', help_text='POP商家store code')
    region_code = models.CharField(max_length=16, null=False, default='CN', help_text='国际国家代码')
    phone_no = models.CharField(max_length=30, null=False, default='', help_text='商家手机号')
    phone_country_code = models.CharField(max_length=16, null=False, default='', help_text='手机号国家码')
    phone_country_iso = models.CharField(max_length=8, null=False, default='', help_text='手机号国家iso2代码')


sql = "CREATE TABLE stocks (date text, trans text, symbol text, qty real, price real)"


```





