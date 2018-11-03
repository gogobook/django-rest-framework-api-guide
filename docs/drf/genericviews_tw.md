> [官方原文鏈接](http://www.django-rest-framework.org/api-guide/generic-views/)

## 通用視圖

基於類的視圖的一個主要優點是它們允許你編寫可重複使用的行為。 REST framework 通過提供大量預構建視圖來提供常用模式，從而充分利用了這一點。

REST framework  提供的通用視圖允許您快速構建緊密映射到數據庫模型的 API 視圖。

如果通用視圖不符合需求，可以使用常規的 `APIView` 類，或者利用 mixin 特性和基類組合出可重用的視圖。

### 舉個栗子

通常，在使用通用視圖時，您需要繼承該視圖，並設置幾個類別屬性。  

``` python
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)
```

對於更復雜的情況，您可能還想重寫視圖類中的各種方法。例如。

``` python
class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser,)

    def list(self, request):
        # 注意使用`get_queryset()`而不是`self.queryset`
        queryset = self.get_queryset()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)
```

對於非常簡單的情況，您可能想要使用 `.as_view()` 方法來傳遞類屬性。例如，您的 URLconf 可能包含類似於以下條目的內容：

``` python
url(r'^/users/', ListCreateAPIView.as_view(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')
```

> 直接在 URLconf 中設置相關屬性參數，這樣連視圖類都不用寫了。



## API 參考

### GenericAPIView

`GenericAPIView` 類繼承於 REST framework 的 `APIView` 類，為標準的列表視圖和詳細視圖添加了常見的行為。

內置的每一個具體的通用視圖都是通過將 `GenericAPIView` 類和一個或多個 minxin 類相互結合來構建的。

#### 屬性

##### 基本設置

使用以下屬性控制基本視圖行為。

* `queryset` - 用於從此視圖返回對象的查詢集。通常，您必須設置此屬性，或覆蓋 `get_queryset()`方法。如果你重寫了一個視圖方法，在視圖方法中，你應該調用 `get_queryset()` 而不是直接訪問這個屬性，這一點很重要！因為 REST 會在內部對 `queryset` 的結果進行緩存用於後續所有請求。
* `serializer_class` - 用於驗證和反序列化輸入以及序列化輸出的序列化類。通常，您必須設置此屬性，或覆蓋 `get_serializer_class()` 方法。
* `lookup_field` - 用於執行各個模型實例的對象查找的模型字段。默認為 `'pk'`。請注意，使用 hyperlinked  API 時，如果需要使用自定義值，則需要確保 API 視圖和序列化類設置了  lookup field。
* `lookup_url_kwarg` - 用於對象查找的 URL 關鍵字參數。URL conf 應該包含與此值相對應的關鍵字參數。如果未設置，則默認使用與 `lookup_field` 相同的值。

關於後兩個參數，我這裡附上對象查詢的源碼大家就應該瞭解了。

省略部分代碼，方便理解

``` python
def get_object(self):
	# 先獲取數據集
    queryset = self.filter_queryset(self.get_queryset())
	# 拿到查詢參數的 key
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field
	# 組裝成 {key:value} 的形式
    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    # 查詢
    obj = get_object_or_404(queryset, **filter_kwargs)
	# 最後返回
    return obj
```

##### 分頁

與列表視圖一起使用時，以下屬性用於控制分頁。

 * `pagination_class` - 對列表進行分頁時使用的分頁類。默認值與 `DEFAULT_PAGINATION_CLASS` 設置相同，即 `rest_framework.pagination.PageNumberPagination`。設置 `pagination_class = None` 將禁用此視圖的分頁。

##### 過濾

 * `filter_backends` - 用於過濾查詢集的過濾器類的列表。默認值與 `DEFAULT_FILTER_BACKENDS` 設置的值相同。


#### 方法

##### 基本方法


`get_queryset(self)`

應該返回列表視圖的查詢集，並應該將其用作查看詳細視圖的基礎。默認返回由 `queryset` 屬性指定的查詢集。

應該始終使用此方法， 而不是直接訪問 `self.queryset`，因為 REST 會在內部對 `self.queryset` 的結果進行緩存用於後續所有請求。

可以覆蓋以提供動態行為，例如針對不同用戶的請求返回不同的數據。

舉個栗子：
``` python
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```

`get_object(self)`

應該返回詳細視圖的對象實例。默認使用 `lookup_field` 參數來過濾基本查詢集。

可以覆蓋以提供更復雜的行為，例如基於多個 URL kwarg 的對象查找。

舉個栗子：
``` python
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```

請注意，如果您的 API 不包含任何對象級權限，您可以選擇排除 `self.check_object_permissions`，並簡單地從 `get_object_or_404` 中查找返回對象。


`filter_queryset(self, queryset)`

給定一個查詢集，使用過濾器進行過濾，返回一個新的查詢集。

舉個栗子：

``` python
def filter_queryset(self, queryset):
    filter_backends = (CategoryFilter,)

    if 'geo_route' in self.request.query_params:
        filter_backends = (GeoRouteFilter, CategoryFilter)
    elif 'geo_point' in self.request.query_params:
        filter_backends = (GeoPointFilter, CategoryFilter)

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
```

`get_serializer_class(self)`

返回用於序列化的類。默認返回 `serializer_class` 屬性。

可以被覆蓋以提供動態行為，例如使用不同的序列化器進行讀寫操作，或為不同類型的用戶提供不同的序列化器。

舉個栗子：

``` python
def get_serializer_class(self):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
```

##### 保存和刪除鉤子（hook）

以下方法由 mixin 類提供，可以很輕鬆的重寫對象的保存和刪除行為。

 * `perform_create(self, serializer)` - 保存新對象實例時由 `CreateModelMixin` 調用。
 * `perform_update(self, serializer)` - 在保存現有對象實例時由 `UpdateModelMixin` 調用。
 * `perform_destroy(self, instance)` - 刪除對象實例時由 `DestroyModelMixin` 調用。

這些鉤子（hook）對設置請求中隱含的但不屬於請求數據的屬性特別有用。例如，您可以根據請求用戶或基於 URL 關鍵字參數在對象上設置屬性。

``` python
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```
這些覆蓋點對於添加保存對象之前或之後發生的行為（如發送確認電子郵件或記錄更新）也特別有用。

``` python
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```

注意：這些方法替代舊式版本2.x `pre_save`，`post_save`，`pre_delete` 和 `post_delete` 方法，這些方法不再可用。


##### 其他方法

通常不需要重寫以下方法，但如果使用 `GenericAPIView` 編寫自定義視圖，則可能需要調用它們。

 * `get_serializer_context(self)` - 返回包含應該提供給序列化的任何額外上下文的字典。默認包括 `'request'`, `'view'` 和 `'format'` 鍵。
 * `get_serializer(self, instance=None, data=None, many=False, partial=False)` - 返回一個序列化器實例。
 * `get_paginated_response(self, data)` - 返回分頁樣式的 `Response` 對象。
 * `paginate_queryset(self, queryset)` - 根據需要為查詢集分頁，或者返回一個頁面對象；如果沒有為該視圖配置分頁，則為 `None`。
 * `filter_queryset(self, queryset)` - 給定一個查詢集，使用過濾器進行過濾，返回一個新的查詢集。


## Mixins 

mixin 類用於提供基本視圖行為的操作。請注意，mixin 類提供了操作方法，而不是直接定義處理方法，如 `.get()` 和 `.post()`。這允許更靈活的行為組合。

mixin 類可以從 `rest_framework.mixins` 中導入。

### ListModelMixin

提供一個 `.list(request, *args, **kwargs)` 方法，實現了列出一個查詢集。

如果查詢集已填充，則返回 `200 OK` 響應，並將 queryset 的序列化表示形式作為響應的主體。響應數據可以設置分頁。


### CreateModelMixin

提供 `.create(request, *args, **kwargs)` 方法，實現創建和保存新模型實例。

如果創建了一個對象，則會返回一個 `201 Created` 響應，並將該對象的序列化表示形式作為響應的主體。如果表示包含名為 url 的鍵，則響應的 `Location` header  將填充該值。

如果為創建對象提供的請求數據無效，則將返回 `400 Bad Request` 響應，並將錯誤細節作為響應的主體。

### RetrieveModelMixin

提供 `.retrieve(request, *args, **kwargs)` 方法，該方法實現在響應中返回現有的模型實例。

如果可以獲取對象，則返回 `200 OK` 響應，並將對象的序列化表示作為響應的主體。否則，將返回一個 `404 Not Found`。



### UpdateModelMixin

提供 `.update(request, *args, **kwargs)` 方法，實現更新和保存現有模型實例。還提供了一個 `.partial_update(request, *args, **kwargs)` 方法，它與更新方法類似，只是更新的所有字段都是可選的。這允許支持 HTTP `PATCH` 請求。

如果成功更新對象，則返回 `200 OK` 響應，並將對象的序列化表示形式作為響應的主體。

如果提供的用於更新對象的請求數據無效，則將返回 `400 Bad Request` 響應，並將錯誤細節作為響應的主體。


### DestroyModelMixin

提供一個 `.destroy(request, *args, **kwargs)` 方法，實現現有模型實例的刪除。

如果一個對象被刪除，則返回一個 `204 No Content` ，否則它將返回一個 `404 Not Found`。



## 內置視圖類列表

以下類是具體的通用視圖。通常情況下，你應該都是使用的它們，除非需要高度的自定義行為。


這些視圖類可以從 `rest_framework.generics` 中導入。


### CreateAPIView

僅用於創建實例。

提供一個 `post` 請求的處理方法。

繼承自： `GenericAPIView `, `CreateModelMixin`

### ListAPIView

僅用於讀取模型實例列表。

提供一個 `get` 請求的處理方法。

繼承自： `GenericAPIView `, `ListModelMixin`

### RetrieveAPIView

僅用於查詢單個模型實例。

提供一個 `get` 請求的處理方法。

繼承自： `GenericAPIView `, `RetrieveModelMixin`

### DestroyAPIView

僅用於刪除單個模型實例。

提供一個 `delete` 請求的處理方法。

繼承自： `GenericAPIView `, `DestroyModelMixin`


### UpdateAPIView

僅用於更新單個模型實例。

提供 `put` 和 `patch` 請求的處理方法。

繼承自： `GenericAPIView `, `UpdateModelMixin`

### ListCreateAPIView

既可以獲取也可以實例集合，也可以創建實例列表

提供 `get` 和 `post` 請求的處理方法。

繼承自： `GenericAPIView `, ` ListModelMixin`，`CreateModelMixin`

### RetrieveUpdateAPIView

既可以查詢也可以更新單個實例

提供 `get`，`put` 和 `patch` 請求的處理方法。

繼承自： `GenericAPIView `, ` RetrieveModelMixin`，`UpdateModelMixin`

### RetrieveDestroyAPIView

既可以查詢也可以刪除單個實例

提供 `get` 和 `delete` 請求的處理方法。

繼承自： `GenericAPIView `, ` RetrieveModelMixin`，`DestroyModelMixin`

### RetrieveUpdateDestroyAPIView

同時支持查詢，更新，刪除

提供 `get`，`put`，`patch` 和 `delete` 請求的處理方法。

繼承自： `GenericAPIView `, ` RetrieveModelMixin`，`UpdateModelMixin`，`DestroyModelMixin`

## 自定義通用視圖類

通常你會想使用現有的通用視圖，然後稍微定製一下行為。如果您發現自己在多個地方重複使用了一些自定義行為，則可能需要將行為重構為普通類，然後根據需要將其應用於任何視圖或視圖集。



### 自定義 mixins

例如，如果您需要根據 URL conf 中的多個字段查找對象，則可以創建一個 mixin 類。

舉個栗子：

``` python
class MultipleFieldLookupMixin(object):
    """
    將此 mixin 應用於任何視圖或視圖集以獲取多個字段過濾
    基於`lookup_fields`屬性，而不是默認的單個字段過濾。
    """
    def get_object(self):
        queryset = self.get_queryset()             # 獲取基本的查詢集
        queryset = self.filter_queryset(queryset)  # 使用過濾器
        filter = {}
        for field in self.lookup_fields:
            if self.kwargs[field]: # 忽略空字段
                filter[field] = self.kwargs[field]
        obj = get_object_or_404(queryset, **filter)  # 查找對象
        self.check_object_permissions(self.request, obj)
        return obj
```

隨後可以在需要應用自定義行為的任​​何時候，將該 mixin 應用於視圖或視圖集。

``` python
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ('account', 'username')
```

如果您需要使用自定義行為，那麼使用自定義 mixins 是一個不錯的選擇。


### 自定義基類

如果您在多個視圖中使用 mixin，您可以進一步創建自己的一組基本視圖，然後在整個項目中使用它們。例如：

``` python
class BaseRetrieveView(MultipleFieldLookupMixin,
                       generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                    generics.RetrieveUpdateDestroyAPIView):
    pass
```

如果自定義行為始終需要在整個項目中的大量視圖中重複使用，那麼使用自定義基類是一個不錯的選擇。


