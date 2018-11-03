> [官方原文鏈接](http://www.django-rest-framework.org/api-guide/views/)

## 基於類的視圖

REST framework  提供了一個 `APIView` 類，它繼承於 Django 的 `View` 類。

`APIView` 類與一般的 `View` 類有以下的不同：  

* 傳遞給處理方法的 request 對象是 REST framework 的 `Request` 實例，而不是 Django 的 `HttpRequest` 實例。
* 處理方法可能是返回 REST framework 的 `Response`，而不是 Django 的 `HttpResponse` 。該視圖將管理內容協商，並在響應中設置正確的渲染器。
* 任何 `APIException` 異常都會被捕獲並進行適當的響應。
* 傳入的請求會進行認證，在請求分派給處理方法之前將進行適當的權限檢查（允許或限制）。


像往常一樣，使用 `API​​View` 類與使用常規 `View` 類非常相似，傳入的請求被分派到適當的處理方法，如 `.get()` 或`.post()`。此外，可以在類上設置許多屬性（AOP）。

舉個栗子：
``` python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import authentication, permissions
from django.contrib.auth.models import User

class ListUsers(APIView):
    """
    列出系統中的所有用戶

    * 需要 token 認證。
    * 只有 admin 用戶才能訪問此視圖。
    """
    authentication_classes = (authentication.TokenAuthentication,)
    permission_classes = (permissions.IsAdminUser,)

    def get(self, request, format=None):
        """
        Return a list of all users.
        """
        usernames = [user.username for user in User.objects.all()]
        return Response(usernames)
```

> **注意**：REST Framework 中的 `APIView`，`GenericAPIView`，各種 `Mixins` 和 `Viewsets` 包含許多方法和屬性，剛開始要全部理解是比較困難的。這裡除了文檔，有一個 [Classy Django REST Framework](http://www.cdrf.co/) 資源，它提供了一個可以在線瀏覽的參照，包含所有屬性和方法。

### API 策略屬性（policy attributes）

以下屬性用於增加擴展視圖的功能，AOP。

.renderer_classes  
設置渲染器

.parser_classes  
設置解析器

.authentication_classes  
設置認證器

.throttle_classes

.permission_classes  
設置權限驗證器

.content_negotiation_class

### API 策略實例方法（policy instantiation methods）

以下策略實例方法通常不需要我們重寫。

.get_renderers(self)

.get_parsers(self)

.get_authenticators(self)

.get_throttles(self)

.get_permissions(self)

.get_content_negotiator(self)

.get_exception_handler(self)


### API 策略實現方法（policy implementation methods）

在分派到處理方法之前會調用以下方法。

.check_permissions(self, request)

.check_throttles(self, request)

.perform_content_negotiation(self, request, force=False)


### 調度方法

以下方法由視圖的 `.dispatch()` 方法直接調用。它們在調用處理方法（`.get()`, `.post()`, `put()`, `patch()` 和 `.delete()`）之前或者之後被調用。

**.initial(self, request, \*args, \*\*kwargs)**

用於執行處理方法被調用之前需要的任何操作。此方法用於強制執行權限和限流，並執行內容協商。

**.handle_exception(self, exc)**

處理方法拋出的任何異常都將傳遞給此方法，該方法返回一個 `Response` 實例，或者重新引發異常。

默認實現處理 `rest_framework.exceptions.APIException` 的任何子類，以及 Django 的 `Http404` 和`PermissionDenied` 異常，並返回相應的錯誤響應。

如果需要自定義 API 返回的錯誤響應，應該重寫此方法。


**.initialize_request(self, request, \*args, \*\*kwargs)**

確保傳遞給處理方法的請求對象是 `Request` 的一個實例，而不是通常的 Django `HttpRequest`。

通常不需要重寫此方法。

**.finalize_response(self, request, response, \*args, \*\*kwargs)**

確保從處理方法返回的任何 `Response` 對象將被呈現為正確的內容類型，這由內容協商確定。

通常不需要重寫此方法。



## 基於方法的視圖(Function Based Views)

REST framework 也允許使用基於函數的視圖。它提供了一套簡單的裝飾器來包裝你的函數視圖，以確保它們接收 `Request`（而不是 Django `HttpRequest`）實例並允許它們返回 `Response`（而不是 Django `HttpResponse`），並允許你配置該請求的處理方式。

### @api_view()

簽名：`@api_view(http_method_names=['GET'])`

`api_view` 是一個裝飾器，用 `http_method_names` 來設置視圖允許響應的 HTTP 方法列表，舉個栗子，編寫一個簡單的視圖，手動返回一些數據。

``` python
from rest_framework.decorators import api_view

@api_view()
def hello_world(request):
    return Response({"message": "Hello, world!"})
```

該視圖將使用 `settings` 中指定的默認渲染器，解析器，認證類等。

默認情況下，只有 `GET` 方法會被接受。其他方法將以 `"405 Method Not Allowed"` 進行響應。要改變這種行為，請指定視圖允許的方法，如下所示：

``` python
@api_view(['GET', 'POST'])
def hello_world(request):
    if request.method == 'POST':
        return Response({"message": "Got some data!", "data": request.data})
    return Response({"message": "Hello, world!"})
```


### API 策略裝飾器 (policy decorators)

為了覆蓋默認設置，REST framework 提供了一系列可以添加到視圖中的附加裝飾器。這些必須在 `@api_view` 裝飾器之後（下方）。例如，要創建一個使用 `throttle` 來確保它每天只能由特定用戶調用一次的視圖，請使用 `@throttle_classes` 裝飾器，傳遞一個 `throttle` 類列表：

``` python
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle

class OncePerDayUserThrottle(UserRateThrottle):
        rate = '1/day'

@api_view(['GET'])
@throttle_classes([OncePerDayUserThrottle])
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

這些裝飾器對應於 `APIView` 上設置的策略屬性。

可用的裝飾器有：

 * `@renderer_classes(...)`
 * `@parser_classes(...)`
 * `@authentication_classes(...)`
 * `@throttle_classes(...)`
 * `@permission_classes(...)`


每個裝飾器都有一個參數，它必須是一個類列表或者一個類元組。


### View schema 裝飾器

要覆蓋函數視圖的默認 模式生成（schema generation），可以使用 `@schema` 裝飾器。這必須在 `@api_view` 裝飾器之後（下方）。例如：
``` python
from rest_framework.decorators import api_view, schema
from rest_framework.schemas import AutoSchema

class CustomAutoSchema(AutoSchema):
    def get_link(self, path, method, base_url):
        # override view introspection here...

@api_view(['GET'])
@schema(CustomAutoSchema())
def view(request):
    return Response({"message": "Hello for today! See you tomorrow!"})
```

該裝飾器將採用一個 `AutoSchema` 實例，一個 `AutoSchema` 子類實例或 `ManualSchema` 實例，如[ Schemas 文檔](http://www.django-rest-framework.org/api-guide/schemas/)（先放官鏈）中所述。您也可以傳 `None` 以從 模式生成（schema generation） 中排除視圖。

``` python
@api_view(['GET'])
@schema(None)
def view(request):
    return Response({"message": "Will not appear in schema!"})
```
