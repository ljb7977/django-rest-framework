# 튜토리얼 6: 뷰셋과 라우터

REST 프레임워크에는 개발자가 API의 상태 및 상호작용을 모델링하는 데 집중하고, 일반적인 컨벤션에 따라 URL 구성을 자동으로 처리해 주는 `ViewSet`을 위한 추상화가 있습니다.

`ViewSet` 클래스는 `get` 또는 `put`과 같은 메소드 핸들러가 아니라 `read` 또는 `update`와 같은 연산을 제공한다는 점을 제외하면 `View` 클래스와 거의 동일합니다.

`ViewSet` 클래스는 마지막에 뷰들의 집합으로 인스턴스화될 때 오직 메소드 핸들러들에만 바인딩됩니다. 이때 보통 복잡한 URLconf 정의를 처리해 주는 `Router` 클래스를 사용합니다.

## 뷰셋 리팩토링하기

현재의 뷰들을 가져와서 뷰셋으로 리팩토링합시다.

우선 우리의 `UserList`와 `UserDetail` 뷰를 하나의 `UserViewSet`로 리팩토링해 봅시다. 두 뷰를 제거하고 하나의 클래스로 대체할 수 있습니다.
```python
from rest_framework import viewsets

class UserViewSet(viewsets.ReadOnlyModelViewSet):
    """
    이 뷰셋은 자동으로 `list`와 `detail` 액션을 제공합니다.
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
여기서 우리는 `ReadOnlyModelViewSet` 클래스를 사용하여 기본적인 '읽기 전용' 연산을 자동으로 만들었습니다. 일반 뷰를 사용할 때와 같이 여전히 `queryset`과 `serializer_class` 속성을 설정하고 있지만, 더이상 다른 두 클래스에 동일한 정보를 제공하지 않아도 됩니다.

다음으로 `SnippetList`, `SnippetDetail` 및 `SnippetHighlight` 뷰 클래스를 대체할 것입니다. 이 세 개의 뷰를 제거하고 다시 하나의 클래스로 대체할 수 있습니다.
```python
from rest_framework.decorators import action
from rest_framework.response import Response

class SnippetViewSet(viewsets.ModelViewSet):
    """
    This viewset automatically provides `list`, `create`, `retrieve`,
    `update` and `destroy` actions.

    Additionally we also provide an extra `highlight` action.
    """
    queryset = Snippet.objects.all()
    serializer_class = SnippetSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                            IsOwnerOrReadOnly]

    @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
    def highlight(self, request, *args, **kwargs):
        snippet = self.get_object()
        return Response(snippet.highlighted)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

이번에는 완전한 읽기 및 쓰기 연산 세트를 얻기 위해 `ModelViewSet` 클래스를 사용했습니다.

`@action` 데코레이터를 사용하여 `highlight`이라는 커스텀 액션를 만든 것을 보세요. 이 데코레이터는 표준 `create`/`update`/`delete` 스타일에 맞지 않는 커스텀 엔드포인트를 추가하는 데에 사용할 수 있습니다.

`@action` 데코레이터를 사용하는 커스텀 액션은 기본적으로 `GET` 요청에 응답합니다. `POST` 요청에 응답하는 액션을 원하면 `methods` 인수를 사용할 수 있습니다.

The URLs for custom actions by default depend on the method name itself. If you want to change the way url should be constructed, you can include `url_path` as a decorator keyword argument.
기본적으로 커스텀 액션의 URL은 메소드 이름에 따라 달라집니다. url의 구성 방식을 변경하려면 `url_path`를 데코레이터 키워드 인수에 포함시키면 됩니다.

## 뷰셋을 URL에 명시적으로 바인딩하기

핸들러 메소드는 URLConf를 정의할 때만 액션에 바인딩됩니다. 그 안에서 무슨 일이 일어나고 있는지 보려면 먼저 우리의 ViewSets으로부터 뷰들을 명시적으로 작성해 보겠습니다.

`snippets/urls.py` 파일 안에서, 우리는 `ViewSet` 클래스를 구체적인 뷰들의 모음에 바인딩합니다.
```python
from snippets.views import SnippetViewSet, UserViewSet, api_root
from rest_framework import renderers

snippet_list = SnippetViewSet.as_view({
    'get': 'list',
    'post': 'create'
})
snippet_detail = SnippetViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'patch': 'partial_update',
    'delete': 'destroy'
})
snippet_highlight = SnippetViewSet.as_view({
    'get': 'highlight'
}, renderer_classes=[renderers.StaticHTMLRenderer])
user_list = UserViewSet.as_view({
    'get': 'list'
})
user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})
```

HTTP 메소드를 각 뷰의 필요한 액션에 바인딩하여 각 `ViewSet` 클래스로부터 여러 개의 뷰를 만드는 방법에 주목하십시오.

리소스를 구체적 뷰에 바인딩했으므로 평소처럼 URL conf로 뷰를 등록할 수 있습니다.
```python
urlpatterns = format_suffix_patterns([
    path('', api_root),
    path('snippets/', snippet_list, name='snippet-list'),
    path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
    path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
    path('users/', user_list, name='user-list'),
    path('users/<int:pk>/', user_detail, name='user-detail')
])
```
## 라우터 사용하기

`View` 클래스 대신 `ViewSet` 클래스를 사용하기 때문에 실제로 URL conf를 직접 디자인할 필요는 없습니다. 뷰와 URL에 리소스를 연결하는 규칙은 `Router` 클래스를 사용하여 자동으로 처리할 수 ​​있습니다. 라우터에 적절한 뷰셋을 등록하면 나머지는 모두 라우터가 해 줍니다.

재연결한 `snippets/urls.py` 파일은 다음과 같습니다.
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from snippets import views

# Create a router and register our viewsets with it.
router = DefaultRouter()
router.register(r'snippets', views.SnippetViewSet)
router.register(r'users', views.UserViewSet)

# The API URLs are now determined automatically by the router.
urlpatterns = [
    path('', include(router.urls)),
]
```
라우터에 뷰셋을 등록하는 것은 urlpattern을 제공하는 것과 유사합니다. 라우터에는 인수로 뷰의 URL 접두사와 뷰셋을 전달해야 합니다.

우리가 사용하는 `DefaultRouter` 클래스도 자동으로 API 루트 뷰를 생성하므로 이제 `views` 모듈에서 `api_root` 메소드를 삭제할 수 있습니다.

## 뷰와 뷰셋 사이의 트레이드오프

뷰셋을 사용하는 것은 정말 유용한 추상화일 수 있습니다. 이를 통해 API 전체에서 URL 규칙의 일관성을 유지하고, 작성해야 하는 코드의 양을 최소화하며, URL conf의 세부 사항보다는 API가 제공하는 상호작용 및 표현에 집중할 수 있습니다.

하지만 뷰셋이 항상 올바른 접근 방식인 것은 아닙니다. 뷰셋에는 함수 기반 뷰 대신 클래스 기반 뷰를 사용할 때와 비슷한 단점이 있습니다. 뷰셋을 사용하는 것은 뷰를 개별적으로 작성하는 것보다 덜 명시적입니다.
