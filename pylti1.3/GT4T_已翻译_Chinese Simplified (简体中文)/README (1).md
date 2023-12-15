# Python 中的 LTI 1.3 Advantage 工具实现



这个项目是一个类似[PHP tool](https://github.com/IMSGlobal/lti-1-3-php-library)的 Python 实现。该库包含用于 Django 和 Flask Web 框架的适配器。进行拓展只需要重新实现 `OIDCLogin` 和 `MessageLaunch` 类，因为它已经在现有适配器中完成了。

# 配置

要配置专属的工具可以使用内置适配器：

``` python
from pylti1p3.tool_config import ToolConfJsonFile
tool_conf = ToolConfJsonFile('path/to/json')

from pylti1p3.tool_config import ToolConfDict
settings = {
    "<issuer_1>": { },  # one issuer ~ one client-id (outdated and not recommended)
    "<issuer_2>": [{ }, { }]  # one issuer ~ many client-ids (recommended method)
}
private_key = '...'
public_key = '...'
tool_conf = ToolConfDict(settings)

client_id = '...' # must be set if implementing the "one issuer ~ many client-ids" concept

tool_conf.set_private_key(iss, private_key, client_id=client_id)
tool_conf.set_public_key(iss, public_key, client_id=client_id)
```

或者使用自己的实现。 `pylti1p3.tool_config.ToolConfAbstract` 接口必须完全实现才能正常工作。 `one issuer ~ many client-ids` 是组织配置的推荐方法，并且在与 Canvas（<https://canvas.instructure.com>）或其他云 LMS-ES 集成的情况下可能有用，其中平台不会因每个客户而改变 `iss`。

JSON 配置示例：

``` javascript
{
    "iss1": [{
        "default": true,
        "client_id": "client_id1",
        "auth_login_url": "auth_login_url1",
        "auth_token_url": "auth_token_url1",
        "auth_audience": null,
        "key_set_url": "key_set_url1",
        "key_set": null,
        "private_key_file": "private.key",
        "public_key_file": "public.key",
        "deployment_ids": ["deployment_id1", "deployment_id2"]
    }, {
        "default": false,
        "client_id": "client_id2",
        "auth_login_url": "auth_login_url2",
        "auth_token_url": "auth_token_url2",
        "auth_audience": null,
        "key_set_url": "key_set_url2",
        "key_set": null,
        "private_key_file": "private.key",
        "public_key_file": "public.key",
        "deployment_ids": ["deployment_id3", "deployment_id4"]
    }],
    "iss2": [ ],
    "iss3": { }
}
```

| `default (bool)` -如果在登录步骤中未传递客户端 ID，则将使用此 ISS 配置
| `client_id` -这是启动期间在“AUD”中收到的 ID
| `auth_login_url` -平台的 OIDC 登录端点
| `auth_token_url` -平台的服务授权端点
| `auth_audience` -平台的 OAuth2 受众（AUD）。用于获取平台的访问令牌。通常与“身份验证 _ 令牌 _URL”相同，可以跳过，但在常见情况下可以是不同的 URL
| `key_set_url` -平台的 JWKS 终结点
| `key_set` -如果平台的 JWKS 端点不可用，可以在此处粘贴 JWKS
| `private_key_file` -工具私钥的相对路径
| `public_key_file` -工具公钥的相对路径
| `deployment_ids (list)` -平台在启动期间传递的部署 _ID



# 使用Flask

## Open ID 连接登录请求

这是一个 API 端点的实例。

创建 `FlaskRequest` 适配器。然后创建一个实例 `FlaskOIDCLogin`。如果登录成功，该 `redirect` 方法将返回一个指向 LTI 平台的 `werkzeug.wrappers.Response` 实例。确保处理异常。

``` python
from flask import request, session
from pylti1p3.flask_adapter import (FlaskRequest, FlaskOIDCLogin)

def login(request_params_dict):

    tool_conf = ... # See Configuration chapter above

    # FlaskRequest by default use flask.request and flask.session
    # so in this case you may define request object without any arguments:

    request = FlaskRequest()

    # in case of using different request object (for example webargs or something like this)
    # you may pass your own values:

    request = FlaskRequest(
        cookies=request.cookies,
        session=session,
        request_data=request_params_dict,
        request_is_secure=request.is_secure
    )

    oidc_login = FlaskOIDCLogin(
        request=request,
        tool_config=tool_conf,
        session_service=FlaskSessionService(request),
        cookie_service=FlaskCookieService(request)
    )

    return oidc_login.redirect(request.get_param('target_link_uri'))
```

## LTI 消息启动

这是一个 API 端点的实例。

创建 `FlaskRequest` 适配器。然后创建一个实例 `FlaskMessageLaunch`。这使你可以在启动成功时访问 LTI 启动消息中的数据。确保处理异常。

``` python
from flask import request, session
from werkzeug.utils import redirect
from pylti1p3.flask_adapter import (FlaskRequest, FlaskMessageLaunch)

def launch(request_params_dict):

    tool_conf = ... # See Configuration chapter above

    request = FlaskRequest()

    # or

    request = FlaskRequest(
        cookies=...,
        session=...,
        request_data=...,
        request_is_secure=...
    )

    message_launch = FlaskMessageLaunch(
        request=request,
        tool_config=tool_conf
    )

    email = message_launch.get_launch_data().get('email')

    # Place your user creation/update/login logic
    # and redirect to tool content here
```

# 访问缓存的启动请求

在后续请求期间，可能希望返回到稍后的启动。这是通过使用启动 ID 来标识缓存的请求来完成的。可以使用以下方式找到启动 ID：

``` python
launch_id = message_launch.get_launch_id()
```

获得启动 ID 后，可以将其链接到会话，并将其作为查询参数传递。

使用启动 ID 检索启动可以通过以下方式完成：

``` python
message_launch = DjangoMessageLaunch.from_cache(launch_id, request, tool_conf)
```

检索后可以正常调用启动对象上的任何方法，例如

``` python
if message_launch.has_ags():
    # Has Assignments and Grades Service
```

# 深度链接响应

如果收到深度链接启动，很可能会希望使用平台的资源来响应深度链接请求。

要创建深度链接响应，只需要获取当前启动的深度链接：

``` python
deep_link = message_launch.get_deep_link()
```

现在需要创建 `pylti1p3.deep_link_resource.DeepLinkResource` 以返回：

``` python
resource = DeepLinkResource()
resource.set_url("https://my.tool/launch")\
    .set_custom_params({'my_param': my_param})\
    .set_title('My Resource')
```

现在，一切都已设置为将资源返回到平台。有两种方法可以这样做。

下面的方法将输出一个自动发布表单的 HTML.

``` python
deep_link.output_response_form([resource1, resource2])
```

或者，也可以通过调用来请求需要张贴回平台的签名JWT。

``` python
deep_link.get_response_jwt([resource1, resource2])
```

# 名称和角色服务

在使用名称和角色之前，应该检查是否有权访问它：

``` python
if not message_launch.has_nrps():
    raise Exception("Don't have names and roles!")
```

一旦知道可以访问它，就可以从启动中获得服务的实例。

``` python
nrps = message_launch.get_nrps()
```

从该服务中，可以通过调用以下命令获得所有成员的列表：

``` python
members = nrps.get_members()
```

要获取包含成员的特定页面：

``` python
members, next_page_url = nrps.get_members_page(page_url)
```

# 派任及职系服务

在使用作业和成绩之前，应该检查是否有权访问它：

``` python
if not launch.has_ags():
    raise Exception("Don't have assignments and grades!")
```

一旦知道可以访问它，就可以从启动中获得服务的实例：

``` python
ags = launch.get_ags()
```

检查不同 `ags` 权限的功能很少：

``` python
# ability to read line items
ags.can_read_lineitem()

# ability to create new line item
ags.can_create_lineitem()

# ability to read grades
ags.can_read_grades()

# ability to pass grades
ags.can_put_grade()
```

要将成绩传递回平台，需要创建一个 `pylti1p3.grade.Grade` 对象，并用必要的信息填充它：

``` python
gr = Grade()
gr.set_score_given(earned_score)\
     .set_score_maximum(100)\
     .set_timestamp(datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S+0000'))\
     .set_activity_progress('Completed')\
     .set_grading_progress('FullyGraded')\
     .set_user_id(external_user_id)
```

要将成绩发送到平台，可以调用：

``` python
ags.put_grade(gr)
```

这将把等级放入默认提供的 LineItem 中。

如果要发回多种类型的成绩，可以通过指定以下内容 `pylti1p3.lineitem.LineItem` 来完成：

``` python
line_item = LineItem()
line_item.set_tag('grade')\
    .set_score_maximum(100)\
    .set_label('Grade')

ags.put_grade(gr, line_item)
```

如果存在相同 `tag` 的 LineItem，则将使用该 LineItem，否则将创建一个新的 Line Item.其他方法：

``` python
# Get one page with line items
items_lst, next_page = ags.get_lineitems_page()

# Get list of all available line items
items_lst = ags.get_lineitems()

# Find line item by ID
item = ags.find_lineitem_by_id(ln_id)

# Find line item by tag
item = ags.find_lineitem_by_tag(ln_tag)

# Find line item by resource ID
item = ags.find_lineitem_by_resource_id(ln_resource_id)

# Find line item by resource link ID
item = ags.find_lineitem_by_resource_link_id(ln_resource_link_id)

# Return all grades for the passed lineitem (across all users enrolled in the line item's context)
grades = ags.get_grades(ln)
```

# 数据隐私发布

Data Privacy Launch 是一种新的可选 LTI 1.3 消息类型，它允许支持 LTI 的工具帮助管理用户管理和执行与数据隐私相关的请求。

``` python
data_privacy_launch = message_launch.is_data_privacy_launch()
if data_privacy_launch:
    user = message_launch.get_data_privacy_launch_user()
```

# 提交审核

Submission Review 为教师或学生提供了一种标准方法，使其从平台的成绩册返回到进行交互的工具，以显示学习者对特定行项目的提交。

``` python
if launch.is_submission_review_launch()
    user = launch.get_submission_review_user()
    ags = launch.get_ags()
    lineitem = ags.get_lineitem()
    submission_review = lineitem.get_submission_review()
```

# 课程组服务

向工具传达课程中可用的组及其各自的注册。

``` python
if launch.has_cgs()
    cgs = launch.get_cgs()

    # Get all available groups
    groups = cgs.get_groups()

    # Get groups for some user
    user_id = '0ae836b9-7fc9-4060-006f-27b2066ac545'
    groups = cgs.get_groups(user_id)

    # Get all sets
    if cgs.has_sets():
        sets = cgs.get_sets()
        sets_with_groups = cgs.get_sets(include_groups=True)
```

# LTI 启动后检查用户的角色

``` python
user_is_staff = message_launch.check_staff_access()
user_is_student = message_launch.check_student_access())
user_is_teacher = message_launch.check_teacher_access()
user_is_teaching_assistant = message_launch.check_teaching_assistant_access()
user_is_designer = message_launch.check_designer_access()
user_is_observer = message_launch.check_observer_access()
user_is_transient = message_launch.check_transient()
```

## Flask 缓存数据存储

``` python
from flask_caching import Cache
from pylti1p3.contrib.flask import FlaskCacheDataStorage

cache = Cache(app)

def login():
    ...
    launch_data_storage = FlaskCacheDataStorage(cache)
    oidc_login = DjangoOIDCLogin(request, tool_conf, launch_data_storage=launch_data_storage)

def launch():
    ...
    launch_data_storage = FlaskCacheDataStorage(cache)
    message_launch = DjangoMessageLaunch(request, tool_conf, launch_data_storage=launch_data_storage)

def restore_launch():
    ...
    launch_data_storage = FlaskCacheDataStorage(cache)
    message_launch = DjangoMessageLaunch.from_cache(launch_id, request, tool_conf,
                                                    launch_data_storage=launch_data_storage)
```

# 公钥缓存

每次在消息启动步骤中，库都会尝试获取平台的公钥。该公钥可以存储在缓存（Memcache/Redis）中，以加速启动过程：

``` python
# Django cache storage:
launch_data_storage = DjangoCacheDataStorage()

# Flask cache storage:
launch_data_storage = FlaskCacheDataStorage(cache)

message_launch.set_public_key_caching(launch_data_storage, cache_lifetime=7200)
```

使用此函数时**重要注意事项！**要小心，因为旋转密钥的时间周期可能小于缓存生存期。例如，D2L 似乎每小时都会使其密钥过期。可以在消息启动期间传递自定义 `requests.Session` 对象，这允许使用 HTTP 响应头进行缓存：

``` python
import requests_cache

requests_session = requests_cache.CachedSession('cache')
message_launch = DjangoMessageLaunch(request, tool_conf, requests_session=requests_session)
```

# 获取 JWKS 的 API

可以从工具配置对象生成 JWK：

``` python
tool_conf.set_public_key(iss, public_key, client_id=client_id)
jwks_dict = tool_conf.get_jwks()  # {"keys": [{ ... }]}

# or you may specify iss and client_id:
jwks_dict = tool_conf.get_jwks(iss, client_id)  # {"keys": [{ ... }]}
```

不要忘记设置公钥，因为没有它，就无法生成 JWKS.你还可以使用以下构造为任何公钥生成 JWK：

``` python
from pylti1p3.registration import Registration

jwk_dict = Registration.get_jwk(public_key)
# {"e": ..., "kid": ..., "kty": ..., "n": ..., "alg": ..., "use": ...}
```
