# LTI 1.3 Advantage Tool implementation in Python



This project is a Python implementation of the similar [PHP
tool](https://github.com/IMSGlobal/lti-1-3-php-library). This library contains adapters for use with the Django and Flask web frameworks.
However, there are no difficulties with adapting it to other frameworks;
you just need to re-implement the `OIDCLogin` and `MessageLaunch`
classes as it is already done in existing adapters.

# Usage Examples

Django: <https://github.com/dmitry-viskov/pylti1.3-django-example>

Flask: <https://github.com/dmitry-viskov/pylti1.3-flask-example>

# Configuration

To configure your own tool, you may use built-in adapters:

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

or create your own implementation. The
`pylti1p3.tool_config.ToolConfAbstract` interface must be fully
implemented for this to work. The concept of
`one issuer ~ many client-ids` is the recommended way to organize
configs and may be useful in the case of integration with Canvas
(<https://canvas.instructure.com>) or other Cloud LMS-es where the
platform doesn\'t change `iss` for each customer.

In the case of the Django Framework, you may use `DjangoDbToolConf` (see
[Configuration using Django Admin
UI](#configuration-using-django-admin-ui) section below).

Example of a JSON config:

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

| `default (bool)` - this iss config will be used in case if client-id
  was not passed on the login step
| `client_id` - this is the id received in the \'aud\' during a launch
| `auth_login_url` - the platform\'s OIDC login endpoint
| `auth_token_url` - the platform\'s service authorization endpoint
| `auth_audience` - the platform\'s OAuth2 Audience (aud). Is used to
  get platform\'s access token. Usually the same as \"auth_token_url\"
  and could be skipped but in the common case could be a different url
| `key_set_url` - the platform\'s JWKS endpoint
| `key_set` - in case if platform\'s JWKS endpoint somehow unavailable
  you may paste JWKS here
| `private_key_file` - relative path to the tool\'s private key
| `public_key_file` - relative path to the tool\'s public key
| `deployment_ids (list)` - The deployment_id passed by the platform
  during launch



# Usage with Flask

## Open Id Connect Login Request

This is a draft of an API endpoint. Wrap it in a library of your choice.

Create a `FlaskRequest` adapter. Then create an instance of
`FlaskOIDCLogin`. The `redirect` method will return an instance of
`werkzeug.wrappers.Response` that points to the LTI platform if login
was successful. Make sure to handle exceptions.

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

## LTI Message Launches

This is a draft of an API endpoint. Wrap it in a library of your choice.

Create a `FlaskRequest` adapter. Then create an instance of
`FlaskMessageLaunch`. This lets you access data from the LTI launch
message if the launch was successful. Make sure to handle exceptions.

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

# Accessing Cached Launch Requests

It is likely that you will want to refer back to a launch later during
subsequent requests. This is done using the launch id to identify a
cached request. The launch id can be found using:

``` python
launch_id = message_launch.get_launch_id()
```

Once you have the launch id, you can link it to your session and pass it
along as a query parameter.

Retrieving a launch using the launch id can be done using:

``` python
message_launch = DjangoMessageLaunch.from_cache(launch_id, request, tool_conf)
```

Once retrieved, you can call any of the methods on the launch object as
normal, e.g.

``` python
if message_launch.has_ags():
    # Has Assignments and Grades Service
```

# Deep Linking Responses

If you receive a deep linking launch, it is very likely that you are
going to want to respond to the deep linking request with resources for
the platform.

To create a deep link response, you will need to get the deep link for
the current launch:

``` python
deep_link = message_launch.get_deep_link()
```

We now need to create `pylti1p3.deep_link_resource.DeepLinkResource` to
return:

``` python
resource = DeepLinkResource()
resource.set_url("https://my.tool/launch")\
    .set_custom_params({'my_param': my_param})\
    .set_title('My Resource')
```

Everything is now set to return the resource to the platform. There are
two methods of doing this.

The following method will output the html for an aut-posting form for
you.

``` python
deep_link.output_response_form([resource1, resource2])
```

Alternatively you can just request the signed JWT that will need posting
back to the platform by calling.

``` python
deep_link.get_response_jwt([resource1, resource2])
```

# Names and Roles Service

Before using names and roles, you should check that you have access to
it:

``` python
if not message_launch.has_nrps():
    raise Exception("Don't have names and roles!")
```

Once we know we can access it, we can get an instance of the service
from the launch.

``` python
nrps = message_launch.get_nrps()
```

From the service we can get a list of all members by calling:

``` python
members = nrps.get_members()
```

To get some specific page with the members:

``` python
members, next_page_url = nrps.get_members_page(page_url)
```

# Assignments and Grades Service

Before using assignments and grades, you should check that you have
access to it:

``` python
if not launch.has_ags():
    raise Exception("Don't have assignments and grades!")
```

Once we know we can access it, we can get an instance of the service
from the launch:

``` python
ags = launch.get_ags()
```

There are few functions to check different `ags` permissions:

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

To pass a grade back to the platform, you will need to create a
`pylti1p3.grade.Grade` object and populate it with the necessary
information:

``` python
gr = Grade()
gr.set_score_given(earned_score)\
     .set_score_maximum(100)\
     .set_timestamp(datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S+0000'))\
     .set_activity_progress('Completed')\
     .set_grading_progress('FullyGraded')\
     .set_user_id(external_user_id)
```

To send the grade to the platform we can call:

``` python
ags.put_grade(gr)
```

This will put the grade into the default provided lineitem.

If you want to send multiple types of grade back, that can be done by
specifying a `pylti1p3.lineitem.LineItem`:

``` python
line_item = LineItem()
line_item.set_tag('grade')\
    .set_score_maximum(100)\
    .set_label('Grade')

ags.put_grade(gr, line_item)
```

If a lineitem with the same `tag` exists, that lineitem will be used,
otherwise a new lineitem will be created. Additional methods:

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

# Data privacy launch

Data Privacy Launch is a new optional LTI 1.3 message type that allows
LTI-enabled tools to assist administrative users in managing and
executing requests related to data privacy.

``` python
data_privacy_launch = message_launch.is_data_privacy_launch()
if data_privacy_launch:
    user = message_launch.get_data_privacy_launch_user()
```

# Submission review

Submission review provides a standard way for an instructor or student
to launch back from a platform\'s gradebook to the tool where the
interaction took place to display the learner\'s submission for a
particular line item.

``` python
if launch.is_submission_review_launch()
    user = launch.get_submission_review_user()
    ags = launch.get_ags()
    lineitem = ags.get_lineitem()
    submission_review = lineitem.get_submission_review()
```

# Course Group Service

Communicates to the tool the groups available in the course and their
respective enrollment.

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

# Check user\'s role after LTI launch

``` python
user_is_staff = message_launch.check_staff_access()
user_is_student = message_launch.check_student_access())
user_is_teacher = message_launch.check_teacher_access()
user_is_teaching_assistant = message_launch.check_teaching_assistant_access()
user_is_designer = message_launch.check_designer_access()
user_is_observer = message_launch.check_observer_access()
user_is_transient = message_launch.check_transient()
```

# Cookies issues in the iframes

Some browsers may deny requests to save cookies in the iframes. For
example, [Google Chrome (from ver.80 onwards) denies requests to
save](https://blog.heroku.com/chrome-changes-samesite-cookie) all
cookies in the iframes except cookies with the flags `Secure` (i.e HTTPS
usage) and `SameSite=None`. [Safari denies requests to
save](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
all third-party cookies by default. The `pylti1p3` library contains
workarounds for such behaviours:

``` python
def login():
    ...
    return oidc_login\
        .enable_check_cookies()\
        .redirect(target_link_uri)
```

After this, the special JS code will try to write and then read test
cookie instead of redirect. The user will see a [special
page](https://raw.githubusercontent.com/dmitry-viskov/repos-assets/master/pylti1p3/examples/cookies-check/001.png)
that will ask them to open the current URL in the new window if cookies
are unavailable. If cookies are allowed, the user will be transparently
redirected to the next page. All texts are configurable with passing
arguments:

``` python
oidc_login.enable_check_cookies(main_msg, click_msg, loading_msg)
```

You may also have troubles with the default framework sessions because
the `pylti1p3` library can\'t control your framework settings connected
with the session ID cookie. Without necessary settings, the user\'s
session could be unavailable in the case of iframe usage. To avoid this,
it is recommended to change the default session adapter to the new cache
adapter (with a memcache/redis backend) and as a consequence, allow the
library to set its own LTI 1.3 session id cookie that will be set with
all necessary params like `Secure` and `SameSite=None`.

## Flask cache data storage

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

# Cache for Public Key

The library will try to fetch the platform\'s public key every time on
the message launch step. This public key may be stored in cache
(memcache/redis) to speed-up the launch process:

``` python
# Django cache storage:
launch_data_storage = DjangoCacheDataStorage()

# Flask cache storage:
launch_data_storage = FlaskCacheDataStorage(cache)

message_launch.set_public_key_caching(launch_data_storage, cache_lifetime=7200)
```

**Important note!** Be careful with using this function because time
period of rotating keys could be less than cache lifetime. For example
D2L appears to expire their keys approximately hourly. You may pass
custom `requests.Session` objects during message launch which allows
caching using HTTP response headers:

``` python
import requests_cache

requests_session = requests_cache.CachedSession('cache')
message_launch = DjangoMessageLaunch(request, tool_conf, requests_session=requests_session)
```

# API to get JWKS

You may generate JWKS from a Tool Config object:

``` python
tool_conf.set_public_key(iss, public_key, client_id=client_id)
jwks_dict = tool_conf.get_jwks()  # {"keys": [{ ... }]}

# or you may specify iss and client_id:
jwks_dict = tool_conf.get_jwks(iss, client_id)  # {"keys": [{ ... }]}
```

Do not forget to set a public key as without it, JWKS cannot be
generated. You may also generate JWK for any public key using the
construction below:

``` python
from pylti1p3.registration import Registration

jwk_dict = Registration.get_jwk(public_key)
# {"e": ..., "kid": ..., "kty": ..., "n": ..., "alg": ..., "use": ...}
```
