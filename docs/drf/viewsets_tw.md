> [官方原文鏈接](http://www.django-rest-framework.org/api-guide/viewsets/)

## 視圖集

> 在路由決定了哪個控制器用於請求後，控制器負責理解請求併產生適當的輸出。     
> — *Ruby on Rails 文檔*

Django REST framework 允許將一組相關視圖的邏輯組合到一個稱為 `ViewSet` 的類中。在其他框架中，您可能會發現概念上類似的實現，名為 “Resources” 或 “Controllers” 。

`ViewSet` 類只是一種基於類的 View，它不提供任何處理方法，如 `.get()` 或 `.post()`，而是提供諸如  `.list()` 和 `.create()` 之類的操作。

`ViewSet` 只在用 `.as_view()` 方法綁定到最終化視圖時做一些相應操作。

通常，不是在 urlconf 中的視圖集中明確註冊視圖，而是使用路由器類註冊視圖集，這會自動為您確定 urlconf。

### 舉個栗子

定義一個簡單的視圖集，可以用來列出或檢索系統中的所有用戶。  

``` python
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):

    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

如果需要，可以將這個視圖集合成兩個單獨的視圖，如下所示：

``` python
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```

通常情況下，我們不會這樣做，而是用路由器註冊視圖集，並允許自動生成 urlconf。

``` python
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, base_name='user')
urlpatterns = router.urls
```

不用自己編寫視圖集，通常使用默認提供的現有基類。例如：

``` python
class UserViewSet(viewsets.ModelViewSet):
    """
    用於查看和編輯用戶實例的視圖。
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

使用 `ViewSet` 類比使用 View 類有兩個主要優點。

 * 重複的邏輯可以合併成一個類。在上面的例子中，我們只需要指定一次查詢集，它將在多個視圖中使用。
 * 通過使用 routers，我們不再需要處理自己的 URL 配置。

這兩者各有優缺點。使用常規視圖和 URL 配置文件更加明確，併為您提供更多控制。如果想要更快速的開發出一個應用，或者需要使大型 API 的 URL 配置始終保持一致，視圖集會非常有用。

### 操作視圖集

REST framework 中包含的默認 routes  將為一組標準的  create / retrieve / update / destroy  風格 action 提供路由，如下所示：

``` python
class UserViewSet(viewsets.ViewSet):
    """
    這些方法將由路由器負責處理。

    如果要使用後綴，請確保加上 `format = None` 關鍵字參數
    """

    def list(self, request):
        pass

    def create(self, request):
        pass

    def retrieve(self, request, pk=None):
        pass

    def update(self, request, pk=None):
        pass

    def partial_update(self, request, pk=None):
        pass

    def destroy(self, request, pk=None):
        pass
```

在調度期間，當前 action 的名稱可以通過 `.action` 屬性獲得。您可以檢查 `.action` 以根據當前 action 調整行為。

例如，您可以將權限限制為只有 admin 才能訪問 `list` 以外的其他 action，如下所示：

``` python
def get_permissions(self):
    """
    實例化並返回此視圖所需的權限列表。
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```

### 標記額外的路由行為

如果需要路由特定方法，則可以用 `@detail_route` 或 `@list_route` 裝飾器進行修飾。

`@detail_route` 裝飾器在其 URL 模式中包含 `pk`，用於支持需要獲取單個實例的方法。`@list_route` 修飾器適用於在對象列表上操作的方法。

舉個栗子：

``` python
from django.contrib.auth.models import User
from rest_framework import status
from rest_framework import viewsets
from rest_framework.decorators import detail_route, list_route
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    提供標準操作的視圖集
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @detail_route(methods=['post'])
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @list_route()
    def recent_users(self, request):
        recent_users = User.objects.all().order('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```

另外，裝飾器可以為路由視圖設置額外的參數。例如...

``` python
    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
       ...
```

這些裝飾器默認路由 `GET` 請求，但也可以使用 `methods` 參數接受其他 HTTP 方法。例如：

``` python
    @detail_route(methods=['post', 'delete'])
    def unset_password(self, request, pk=None):
       ...
```

這兩個新操作將在 urls  `^users/{pk}/set_password/$` 和 `^users/{pk}/unset_password/$` 上。

### action 跳轉

如果你需要獲取 action 的 URL ，請使用 `.reverse_action()` 方法。這是 `.reverse()` 的一個便捷包裝，它會自動傳遞視圖的請求對象，並將 `url_name` 與 `.basename` 屬性掛接。

請注意，`basename` 是在 `ViewSet` 註冊過程中由路由器提供的。如果您不使用路由器，則必須提供`.as_view()` 方法的 `basename` 參數。

使用上一節中的示例：

``` python
>>> view.reverse_action('set-password', args=['1'])
'http://localhost:8000/api/users/1/set_password'
```

`url_name` 參數應該與 `@list_route` 和 `@detail_route` 裝飾器的相同參數匹配。另外，這可以用來反轉默認 `list` 和 `detail` 路由。


## API 參考


### ViewSet

`ViewSet` 類繼承自 `APIView`。您可以使用任何標準屬性（如 `permission_classes`，`authentication_classes`）來控制視圖上的 API 策略。

`ViewSet` 類不提供任何 action 的實現。為了使用 `ViewSet` 類，必須繼承該類並明確定義 action 實現。

### GenericViewSet

`GenericViewSet` 類繼承自 `GenericAPIView`，並提供默認的 `get_object`，`get_queryset` 方法和其他通用視圖基礎行為，但默認情況下不包含任何操作。

為了使用 `GenericViewSet` 類，必須繼承該類並混合所需的 mixin 類，或明確定義操作實現。

### ModelViewSet

`ModelViewSet` 類繼承自 `GenericAPIView`，並通過混合各種 mixin 類的行為來包含各種操作的實現。

`ModelViewSet` 提供的操作有 `.list()` , `.retrieve()` , `.create()` , `.update()` , `.partial_update()`, 和 `.destroy()` 。


舉個栗子：

由於 `ModelViewSet` 類繼承自 `GenericAPIView`，因此通常需要提供至少 `queryset` 和 `serializer_class` 屬性。例如：

``` python
class AccountViewSet(viewsets.ModelViewSet):
    """
    用於查看和編輯 Account
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```

請注意，您可以覆蓋 `GenericAPIView` 提供的任何標準屬性或方法。例如，要動態確定它應該操作的查詢集的`ViewSet`，可以這樣做：

``` python
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing the accounts
    associated with the user.
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```


但請注意，從 `ViewSet` 中刪除 `queryset` 屬性後，任何關聯的 router 將無法自動導出模型的 `base_name`，因此您必須將 `base_name` kwarg 指定為 router 註冊的一部分。

還要注意，雖然這個類默認提供了完整的  create / list / retrieve / update / destroy  操作集，但您可以通過使用標準權限類來限制可用操作。


### ReadOnlyModelViewSet

 `ReadOnlyModelViewSet` 類也從 `GenericAPIView` 繼承。與 `ModelViewSet` 一樣，它也包含各種操作的實現，但與 `ModelViewSet` 不同的是它只提供 “只讀” 操作，`.list()` 和 `.retrieve()`。

舉個栗子：

與 `ModelViewSet` 一樣，您通常需要提供至少 `queryset` 和 `serializer_class` 屬性。例如：

``` python
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A simple ViewSet for viewing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```

同樣，與 `ModelViewSet` 一樣，您可以覆蓋`GenericAPIView` 可用的任何標準屬性和方法。


## 自定義視圖集基類


您可能需要使用沒有完整 `ModelViewSet` 操作集的自定義 `ViewSet` 類，或其他自定義行為。

舉個栗子：

要創建提供 `create`， `list` 和 `retrieve` 操作的基本視圖集類，請從 `GenericViewSet` 繼承，並混合（mixin ）所需的操作：

``` python
from rest_framework import mixins

class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                mixins.ListModelMixin,
                                mixins.RetrieveModelMixin,
                                viewsets.GenericViewSet):
    """
    A viewset that provides `retrieve`, `create`, and `list` actions.

    To use it, override the class and set the `.queryset` and
    `.serializer_class` attributes.
    """
    pass
```

通過創建自己的基本 `ViewSet` 類，能夠提供可在 API 中的多個視圖集中重用的常見行為。