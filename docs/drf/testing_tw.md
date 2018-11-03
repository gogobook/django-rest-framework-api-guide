> [官方原文鏈接](http://www.django-rest-framework.org/api-guide/testing/)  


# 測試


REST framework 包含一些擴展 Django 現有測試框架的助手類，並改進對 API 請求的支持。

# APIRequestFactory

擴展了 Django 現有的 `RequestFactory` 類。

## 創建測試請求

`APIRequestFactory` 類支持與 Django 的標準 `RequestFactory` 類幾乎完全相同的 API。這意味著標準的 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`, `.head()` 和 `.options()` 方法都可用。

``` python
from rest_framework.test import APIRequestFactory

# Using the standard RequestFactory API to create a form POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'})
```

#### 使用 `format` 參數

創建請求主體（如 `post`，`put` 和 `patch`）的方法包括 `format` 參數，這使得使用除 multipart 表單數據以外的內容類型生成請求變得容易。例如：

``` python
# Create a JSON POST request
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'}, format='json')
```

默認情況下，可用的格式是 `'multipart'` 和 `'json'` 。為了與 Django 現有的 `RequestFactory` 兼容，默認格式是 `'multipart'`。


#### 顯式編碼請求主體

如果你需要顯式編碼請求正文，則可以通過設置 `content_type` 標誌來完成。例如：

``` python
request = factory.post('/notes/', json.dumps({'title': 'new idea'}), content_type='application/json')
```

#### PUT 和 PATCH 與表單數據

Django 的 `RequestFactory` 和 REST framework 的 `APIRequestFactory` 之間值得注意的一個區別是 multipart 表單數據將被編碼為除 `.post()` 以外的方法。

例如，使用 `APIRequestFactory`，你可以像這樣做一個表單 PUT 請求：

``` python
factory = APIRequestFactory()
request = factory.put('/notes/547/', {'title': 'remember to email dave'})
```

使用 Django 的 `RequestFactory`，你需要自己顯式編碼數據：

``` python
from django.test.client import encode_multipart, RequestFactory

factory = RequestFactory()
data = {'title': 'remember to email dave'}
content = encode_multipart('BoUnDaRyStRiNg', data)
content_type = 'multipart/form-data; boundary=BoUnDaRyStRiNg'
request = factory.put('/notes/547/', content, content_type=content_type)
```

## 強制認證

當使用請求工廠直接測試視圖時，能夠直接驗證請求通常很方便，而不必構造正確的驗證憑證。

要強制驗證請求，請使用 `force_authenticate()` 方法。

``` python
from rest_framework.test import force_authenticate

factory = APIRequestFactory()
user = User.objects.get(username='olivia')
view = AccountDetail.as_view()

# Make an authenticated request to the view...
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user)
response = view(request)
```

該方法的簽名是 `force_authenticate(request, user=None, token=None)`。調用時，可以設置 user 和 token 中的任一個或兩個。

例如，當使用令牌強行進行身份驗證時，你可能會執行以下操作：

``` python
user = User.objects.get(username='olivia')
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user, token=user.auth_token)
```

---

**注意**: `force_authenticate` 直接將 `request.user` 設置為內存中的 `user` 實例。如果跨多個測試重新使用同一個 `user` 實例來更新已保存的 `user` 狀態，則可能需要在測試之間調用 `refresh_from_db()`。

---

**注意**: 使用 `APIRequestFactory` 時，返回的對象是 Django 的標準 `HttpRequest`，而不是 REST framework 的 `Request` 對象，只有在調用視圖後才會生成該對象。

這意味著直接在請求對象上設置屬性可能並不總是有你期望的效果。例如，直接設置 `.token` 將不起作用，並且僅在使用會話身份驗證時直接設置 `.user` 才會起作用。

``` python
# Request will only authenticate if `SessionAuthentication` is in use.
request = factory.get('/accounts/django-superstars/')
request.user = user
response = view(request)
```

---

## 強制 CSRF 驗證

默認情況下，使用 `APIRequestFactory` 創建的請求在傳遞給 REST framework 視圖時不會應用 CSRF 驗證。如果你需要明確打開 CSRF 驗證，則可以通過在實例化工廠時設置 `enforce_csrf_checks` 標誌來實現。

``` python
factory = APIRequestFactory(enforce_csrf_checks=True)
```

---

**注意**: 值得注意的是，Django 的標準 `RequestFactory` 不需要包含這個選項，因為當使用常規的 Django 時，CSRF 驗證發生在中間件中，當直接測試視圖時該中間件不運行。在使用 REST framework 時，CSRF 驗證發生在視圖內，因此請求工廠需要禁用視圖級 CSRF 檢查。

---

# APIClient

擴展了 Django 現有的 `Client` 類。

## 發出請求

`APIClient` 類支持與 Django 標準 `Client` 類相同的請求接口。這意味著標準的 `.get()`, `.post()`, `.put()`, `.patch()`, `.delete()`, `.head()` 和 `.options()` 方法都可用。例如：

``` python
from rest_framework.test import APIClient

client = APIClient()
client.post('/notes/', {'title': 'new idea'}, format='json')
```


## 認證

#### .login(**kwargs)

`login` 方法的功能與 Django 的常規 `Client` 類一樣。這使你可以對任何包含 `SessionAuthentication` 的視圖進行身份驗證。

``` python
# Make all requests in the context of a logged in session.
client = APIClient()
client.login(username='lauren', password='secret')
```

要登出，請照常調用 `logout` 方法。

``` python
# Log out
client.logout()
```

`login` 方法適用於測試使用會話認證的 API，例如包含 AJAX 與 API 交互的網站。

#### .credentials(**kwargs)

`credentials` 方法可用於設置 header，這些 header 將包含在測試客戶端的所有後續請求中。

``` python
from rest_framework.authtoken.models import Token
from rest_framework.test import APIClient

# Include an appropriate `Authorization:` header on all requests.
token = Token.objects.get(user__username='lauren')
client = APIClient()
client.credentials(HTTP_AUTHORIZATION='Token ' + token.key)
```

請注意，第二次調用 `credentials` 會覆蓋任何現有憑證。你可以通過調用沒有參數的方法來取消任何現有的憑證。

``` python
# Stop including any credentials
client.credentials()
```

`credentials` 方法適用於測試需要驗證 header 的 API，例如 basic 驗證，OAuth1 和 OAuth2 驗證以及簡單令牌驗證方案。

#### .force_authenticate(user=None, token=None)

有時你可能想完全繞過認證，強制測試客戶端的所有請求被自動視為已認證。

如果你正在測試 API 但是不想構建有效的身份驗證憑據以進行測試請求，則這可能是一個有用的捷徑。

``` python
user = User.objects.get(username='lauren')
client = APIClient()
client.force_authenticate(user=user)
```

要對後續請求進行身份驗證，請調用 `force_authenticate` 將 user 和(或) token 設置為 `None`。

``` python
client.force_authenticate(user=None)
```

## CSRF 驗證

默認情況下，使用 `APIClient` 時不應用 CSRF 驗證。如果你需要明確啟用 CSRF 驗證，則可以通過在實例化客戶端時設置 `enforce_csrf_checks` 標誌來實現。

``` python
client = APIClient(enforce_csrf_checks=True)
```

像往常一樣，CSRF 驗證將僅適用於任何會話驗證視圖。這意味著 CSRF 驗證只有在客戶端通過調用 `login()` 登錄後才會發生。

---

# RequestsClient

REST framework 還包含一個客戶端，用於使用流行的 Python 庫 `requests` 與應用程序進行交互。 這可能是有用的，如果：

* 你期望主要從另一個 Python 服務與 API 進行交互，並且希望在與客戶端相同的級別測試該服務。
* 你希望以這樣的方式編寫測試，以便它們也可以在分段或實時環境中運行。 

它暴露了與直接使用請求會話完全相同的接口。

``` python
client = RequestsClient()
response = client.get('http://testserver/users/')
assert response.status_code == 200
```

請注意，requests client 要求你傳遞完全限定的 URL。

## `RequestsClient` 與數據庫一起工作

如果你想編寫僅與服務接口交互的測試，則 `RequestsClient` 類很有用。這比使用標準的 Django 測試客戶端要嚴格一些，因為這意味著所有的交互必須通過 API。

如果你使用的是 `RequestsClient`，你需要確保測試設置和結果斷言以常規 API 調用的方式執行，而不是直接與數據庫模型進行交互。例如，不是檢查 `Customer.objects.count() == 3`，而是列出 customers 端點，並確保它包含三條記錄。

## Headers & 身份驗證

自定義 header 和身份驗證憑證的提供方式與使用標準 `requests.Session` 實例時的方式相同。

``` python
from requests.auth import HTTPBasicAuth

client.auth = HTTPBasicAuth('user', 'pass')
client.headers.update({'x-test': 'true'})
```

## CSRF

如果你使用 `SessionAuthentication` ，那麼你需要為 `POST`, `PUT`, `PATCH` 或 `DELETE` 請求包含一個 CSRF 令牌。

你可以通過遵循基於 JavaScript 的客戶端使用的相同流程來實現。首先進行 `GET` 請求以獲取 CRSF 令牌，然後在以下請求中呈現該令牌。

例如...

``` python
client = RequestsClient()

# Obtain a CSRF token.
response = client.get('/homepage/')
assert response.status_code == 200
csrftoken = response.cookies['csrftoken']

# Interact with the API.
response = client.post('/organisations/', json={
    'name': 'MegaCorp',
    'status': 'active'
}, headers={'X-CSRFToken': csrftoken})
assert response.status_code == 200
```

## Live tests

使用 `RequestsClient` 和 `CoreAPIClient` 可以編寫在開發環境中運行的測試用例，也可以直接根據測試服務器或生產環境運行測試用例。

使用這種風格來創建幾個核心功能的基本測試是驗證你的實時服務的有效方法。這樣做可能需要仔細注意安裝和卸載（setup and teardown），以確保測試的運行方式不會直接影響客戶數據。

---

# CoreAPIClient

CoreAPIClient 允許你使用 Python `coreapi` 客戶端庫與你的 API 進行交互。

``` python
# Fetch the API schema
client = CoreAPIClient()
schema = client.get('http://testserver/schema/')

# Create a new organisation
params = {'name': 'MegaCorp', 'status': 'active'}
client.action(schema, ['organisations', 'create'], params)

# Ensure that the organisation exists in the listing
data = client.action(schema, ['organisations', 'list'])
assert(len(data) == 1)
assert(data == [{'name': 'MegaCorp', 'status': 'active'}])
```

## Headers & 身份驗證

自定義 header 和身份驗證可以與 `RequestsClient` 類似的方式和 `CoreAPIClient` 一起使用。

``` python
from requests.auth import HTTPBasicAuth

client = CoreAPIClient()
client.session.auth = HTTPBasicAuth('user', 'pass')
client.session.headers.update({'x-test': 'true'})
```

---

# API Test cases

REST framework 包含以下測試用例類，它們類似現有的 Django 測試用例類，但使用 `API​​Client` 而不是 Django 的默認 `Client`。

* `APISimpleTestCase`
* `APITransactionTestCase`
* `APITestCase`
* `APILiveServerTestCase`

## 舉個栗子

你可以像使用常規 Django 測試用例類一樣使用任何 REST framework 的測試用例類。 `self.client` 屬性將是一個 `APIClient` 實例。

``` python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase
from myproject.apps.core.models import Account

class AccountTests(APITestCase):
    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        data = {'name': 'DabApps'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Account.objects.count(), 1)
        self.assertEqual(Account.objects.get().name, 'DabApps')
```

---

# URLPatternsTestCase

REST framework 還提供了一個用於隔離每個類的 `urlpatterns` 的測試用例類。請注意，它繼承自 Django 的 `SimpleTestCase`，並且很可能需要與另一個測試用例類混合使用。

## 例如

``` python
from django.urls import include, path, reverse
from rest_framework.test import APITestCase, URLPatternsTestCase


class AccountTests(APITestCase, URLPatternsTestCase):
    urlpatterns = [
        path('api/', include('api.urls')),
    ]

    def test_create_account(self):
        """
        Ensure we can create a new account object.
        """
        url = reverse('account-list')
        response = self.client.get(url, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)
```

---

# 測試響應

## 檢查響應數據

在檢查測試響應的有效性時，檢查響應的創建數據通常比較方便，而不是檢查完全渲染的響應。

例如，檢查 `response.data` 更容易：

``` python
response = self.client.get('/users/4/')
self.assertEqual(response.data, {'id': 4, 'username': 'lauren'})
```

而不是檢查解析 `response.content` 的結果：

``` python
response = self.client.get('/users/4/')
self.assertEqual(json.loads(response.content), {'id': 4, 'username': 'lauren'})
```

## 渲染響應

如果你使用 `API​​RequestFactory` 直接測試視圖，則返回的響應將不會渲染，因為模板響應的渲染由 Django 的內部請求 - 響應循環執行。為了訪問 `response.content`，你首先需要渲染響應。

``` python
view = UserDetail.as_view()
request = factory.get('/users/4')
response = view(request, pk='4')
response.render()  # Cannot access `response.content` without this.
self.assertEqual(response.content, '{"username": "lauren", "id": 4}')
```

---

# 配置

## 設置默認格式

用於創建測試請求的默認格式可以使用 `TEST_REQUEST_DEFAULT_FORMAT` setting key 進行設置。例如，默認情況下總是對測試請求使用 JSON 而不是標準的 multipart 表單請求，請在 `settings.py` 文件中設置以下內容：

``` python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_DEFAULT_FORMAT': 'json'
}
```

## 設置可用的格式

如果你需要使用除 multipart 或 json 請求之外的其他方法來測試請求，則可以通過設置 `TEST_REQUEST_RENDERER_CLASSES` setting 來完成。

例如，要在測試請求中添加對 `format ='html'` 的支持，您可能在 `settings.py` 文件中有這樣的內容。

``` python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework.renderers.MultiPartRenderer',
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.TemplateHTMLRenderer'
    )
}
```