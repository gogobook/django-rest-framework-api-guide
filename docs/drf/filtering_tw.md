> [官方原文鏈接](http://www.django-rest-framework.org/api-guide/filtering/)  


# 過濾


REST framework 的通用列表視圖的默認行為是從模型管理器返回整個查詢集。通常你會希望 API 限制查詢集返回的條目。

篩選 `GenericAPIView` 子類的查詢集的最簡單方法是重寫 `.get_queryset()` 方法。

重寫此方法允許你以多種不同方式自定義視圖返回的查詢集。

## 根據當前用戶進行過濾

你可能需要過濾查詢集，以確保只返回與當前通過身份驗證的用戶發出的請求相關的結果。

你可以基於 `request.user` 的值進行篩選來完成此操作。

例如：

``` python
from myapp.models import Purchase
from myapp.serializers import PurchaseSerializer
from rest_framework import generics

class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases
        for the currently authenticated user.
        """
        user = self.request.user
        return Purchase.objects.filter(purchaser=user)
```


## 根據 URL 進行過濾

另一種過濾方式可能涉及基於 URL 的某個部分限制查詢集。

例如，如果你的 URL 配置包含這樣的條目：

``` python
url('^purchases/(?P<username>.+)/$', PurchaseList.as_view()),
```

然後，你可以編寫一個視圖，返回由 URL 的用戶名部分過濾的 purchase 查詢集：

``` python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases for
        the user as determined by the username portion of the URL.
        """
        username = self.kwargs['username']
        return Purchase.objects.filter(purchaser__username=username)
```

## 根據查詢參數進行過濾

過濾初始查詢集的最後一個例子是根據 url 中的查詢參數確定初始查詢集。

我們可以覆蓋 `.get_queryset()` 來處理諸如 `http://example.com/api/purchases?username=denvercoder9` 的URL，並且只有在 URL 中包含 `username` 參數時才過濾查詢集：

``` python
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned purchases to a given user,
        by filtering against a `username` query parameter in the URL.
        """
        queryset = Purchase.objects.all()
        username = self.request.query_params.get('username', None)
        if username is not None:
            queryset = queryset.filter(purchaser__username=username)
        return queryset
```

---

# 通用過濾器

除了能夠覆蓋默認的查詢集外，REST framework 還包括對通用過濾後端的支​​持，使你可以輕鬆構建複雜的搜索和過濾器。

通用過濾器也可以在可瀏覽的 API 和管理 API 中將自己渲染為 HTML 控件。

![Filter Example](http://www.django-rest-framework.org/img/filter-controls.png)

## 設置過濾器後端

可以使用 `DEFAULT_FILTER_BACKENDS` setting 全局設置默認的過濾器後端。例如。

``` python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

你還可以使用基於 `GenericAPIView` 類的視圖，在每個視圖或視圖集的基礎上設置過濾器後端。

``` python
import django_filters.rest_framework
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (django_filters.rest_framework.DjangoFilterBackend,)
```

## 過濾和對象查找

請注意，如果為一個視圖配置了一個過濾器後端，那麼除了用於篩選列表視圖之外，它還將用於篩選返回單個對象的查詢集。

例如，根據前面的示例以及 ID 為 `4675` 的產品，以下 URL 將返回相應的對象，或返回 404 響應，具體取決於給定產品實例是否滿足過濾條件：

``` python
http://example.com/api/products/4675/?category=clothing&max_price=10.00
```

## 覆蓋初始查詢集

請注意，你可以同時重寫的 `.get_queryset()` 和通用過濾，並且所有內容都將按預期工作。例如，如果產品與用戶具有多對多關係，則可能需要編寫一個如下所示的視圖：

``` python
class PurchasedProductsList(generics.ListAPIView):
    """
    Return a list of all the products that the authenticated
    user has ever purchased, with optional filtering.
    """
    model = Product
    serializer_class = ProductSerializer
    filter_class = ProductFilter

    def get_queryset(self):
        user = self.request.user
        return user.purchase_set.all()
```

---

# API 參考

## DjangoFilterBackend

`django-filter` 庫包含一個 `DjangoFilterBackend` 類，它支持 REST framework 對字段過濾進行高度定製。

要使用 `DjangoFilterBackend`，首先安裝 `django-filter`。然後將 `django_filters` 添加到 Django 的 `INSTALLED_APPS` 中

``` python
pip install django-filter
```

你現在應該將過濾器後端添加到設置中：

``` python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

或者將過濾器後端添加到單個視圖或視圖集。

``` python
from django_filters.rest_framework import DjangoFilterBackend

class UserListView(generics.ListAPIView):
    ...
    filter_backends = (DjangoFilterBackend,)
```

如果你只需要簡單的基於等式的過濾，則可以在視圖或視圖集上設置 `filter_fields` 屬性，列出你要過濾的一組字段。

``` python
class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (DjangoFilterBackend,)
    filter_fields = ('category', 'in_stock')
```

這將自動為給定字段創建一個 `FilterSet` 類，並允許你發出如下請求：

``` python
http://example.com/api/products?category=clothing&in_stock=True
```

對於更高級的過濾要求，你應該在視圖上在指定 `FilterSet` 類。你可以在 [django-filter 文檔](https://django-filter.readthedocs.io/en/latest/index.html)中閱讀有關 `FilterSet` 的更多信息。還建議你閱讀 [DRF integration](https://django-filter.readthedocs.io/en/latest/guide/rest_framework.html)。


## SearchFilter

`SearchFilter` 類支持簡單的基於單個查詢參數的搜索，並且基於 Django 管理員的搜索功能。

在使用時，可瀏覽的 API 將包含一個 `SearchFilter` 控件：

![Search Filter](http://www.django-rest-framework.org/img/search-filter.png)

`SearchFilter` 類將僅在視圖具有 `search_fields` 屬性集的情況下應用。`search_fields` 屬性應該是模型上文本類型字段的名稱列表，例如 `CharField` 或 `TextField`。

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.SearchFilter,)
    search_fields = ('username', 'email')
```

這將允許客戶端通過查詢來過濾列表中的項目，例如：

```
http://example.com/api/users?search=russell
```

你還可以使用查找 API 雙下劃線表示法對 ForeignKey 或 ManyToManyField 執行相關查找：

``` python
search_fields = ('username', 'email', 'profile__profession')
```

默認情況下，搜索將使用不區分大小寫的部分匹配。搜索參數可能包含多個搜索詞，它們應該是空格和（或）逗號分隔的。如果使用多個搜索條件，則只有在所有提供的條件匹配的情況下，對象才會返回到列表中。

搜索行為可以通過將各種字符預先添加到 `search_fields` 來限制。

* '^' 匹配起始部分。
* '=' 完全匹配。
* '@' 全文搜索。（目前只支持 Django 的 MySQL 後端。）
* '$' 正則匹配。

例如：

``` python
search_fields = ('=username', '=email')
```

默認情況下，搜索參數被命名為 `'search'` ，但這可能會被 `SEARCH_PARAM` setting 覆蓋。

有關更多詳細信息，請參閱 [Django 文檔](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)。

---

## OrderingFilter

`OrderingFilter` 類支持簡單查詢參數控制結果的排序。

![Ordering Filter](http://www.django-rest-framework.org/img/ordering-filter.png)

默認情況下，查詢參數被命名為 `'ordering'`，但這可能會被 `ORDERING_PARAM` setting 覆蓋。

例如，要通過 username 對用戶排序：

```
http://example.com/api/users?ordering=username
```

客戶端也可以通過在字段名稱前加 ' - ' 來指定反向排序，如下所示：

```
http://example.com/api/users?ordering=-username 
```

也可以指定多個排序：

```
http://example.com/api/users?ordering=account,username
```

### 指定可以根據哪些字段進行排序

建議你明確指定 API 應該允許在排序過濾器中使用哪些字段。你可以通過在視圖上設置一個 `ordering_fields` 屬性來完成此操作，如下所示：

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
```

這有助於防止意外的數據洩漏，例如允許用戶根據密碼哈希字段或其他敏感數據進行排序。

如果你未在視圖上指定 `ordering_fields` 屬性，則過濾器類將默認允許用戶過濾由 `serializer_class` 屬性指定的序列化類中的任何可讀字段。

如果你確信視圖使用的查詢集不包含任何敏感數據，則還可以通過使用特殊值 `'__all__'` 明確指定視圖允許在任何模型字段或查詢集聚合上進行排序。

``` python
class BookingsListView(generics.ListAPIView):
    queryset = Booking.objects.all()
    serializer_class = BookingSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = '__all__'
```

### 指定默認順序

如果在視圖上設置了 `ordering` 屬性，則將用作默認排序。

通常情況下，你應該通過在初始查詢集上設置 `order_by` 來控制此操作，但是通過在視圖上使用 `ordering` 參數，你可以指定排序方式，然後可以將其作為上下文自動傳遞到渲染的模板。這可以自動渲染列標題，如果它們用於排序結果。

``` python
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
    ordering = ('username',)
```

`ordering` 屬性可以是一個字符串或者字符串列表（元組）。

---

## DjangoObjectPermissionsFilter

`DjangoObjectPermissionsFilter` 旨在與 [`django-guardian`](https://django-guardian.readthedocs.io/) 軟件包一起使用，添加了自定義 `'view'` 的權限。過濾器將確保查詢集僅返回用戶具有適當查看權限的對象。

如果你使用的是 `DjangoObjectPermissionsFilter`，那麼你可能還需要添加適當的對象權限類，以確保用戶只有在具有適當對象權限的情況下才能對實例進行操作。做到這一點的最簡單方法是繼承 `DjangoObjectPermissions` 併為 `perms_map` 屬性添加 `'view'` 權限。

使用 `DjangoObjectPermissionsFilter` 和 `DjangoObjectPermissions` 的完整示例可能如下所示。

**permissions.py**:

``` python
class CustomObjectPermissions(permissions.DjangoObjectPermissions):
    """
    Similar to `DjangoObjectPermissions`, but adding 'view' permissions.
    """
    perms_map = {
        'GET': ['%(app_label)s.view_%(model_name)s'],
        'OPTIONS': ['%(app_label)s.view_%(model_name)s'],
        'HEAD': ['%(app_label)s.view_%(model_name)s'],
        'POST': ['%(app_label)s.add_%(model_name)s'],
        'PUT': ['%(app_label)s.change_%(model_name)s'],
        'PATCH': ['%(app_label)s.change_%(model_name)s'],
        'DELETE': ['%(app_label)s.delete_%(model_name)s'],
    }
```

**views.py**:

``` python
class EventViewSet(viewsets.ModelViewSet):
    """
    Viewset that only lists events if user has 'view' permissions, and only
    allows operations on individual events if user has appropriate 'view', 'add',
    'change' or 'delete' permissions.
    """
    queryset = Event.objects.all()
    serializer_class = EventSerializer
    filter_backends = (filters.DjangoObjectPermissionsFilter,)
    permission_classes = (myapp.permissions.CustomObjectPermissions,)
```



---

# 自定義通用過濾器

你還可以提供自己的通用過濾器後端，或者編寫一個可供其他開發人員使用的可安裝應用程序。

為此，請繼承 `BaseFilterBackend`，並覆蓋 `.filter_queryset(self, request, queryset, view)` 方法。該方法應該返回一個新的，過濾的查詢集。

除了允許客戶端執行搜索和過濾外，通用過濾器後端可用於限制哪些對象應該對給定的請求或用戶可見。

## 舉個栗子

你可能需要限制用戶只能看到他們創建的對象。

``` python
class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """
    Filter that only allows users to see their own objects.
    """
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(owner=request.user)
```

我們可以通過覆蓋視圖上的 `get_queryset()`來實現相同的行為，但使用過濾器後端允許你更輕鬆地將此限制添加到多個視圖，或者將其應用於整個 API。

## 自定義接口

通用過濾器也可以在可瀏覽的 API 中渲染接口。為此，你應該實現一個 `to_html()` 方法，該方法返回過濾器的渲染 HTML 表示。此方法應具有以下簽名：

`to_html(self, request, queryset, view)`

該方法應該返回一個渲染的 HTML 字符串。

## Pagination & schemas

通過實現 `get_schema_fields()` 方法，你還可以使過濾器控件可用於 REST framework 提供的模式自動生成。此方法應具有以下簽名：

`get_schema_fields(self, view)`

該方法應該返回一個 `coreapi.Field` 實例列表。
