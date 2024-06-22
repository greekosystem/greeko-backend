<!-- DISCLAIMER: Some parts of this document are AI Generated -->

# Coding Standards for Django + DRF + Python

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Python and Django Style Guide](#2-python-and-django-style-guide)
3. [API Design](#3-api-design)
4. [Models and Database](#4-models-and-database)
5. [Serializers](#5-serializers)
6. [Views and ViewSets](#6-views-and-viewsets)
7. [URLs and Routing](#7-urls-and-routing)
8. [Authentication and Permissions](#8-authentication-and-permissions)
9. [Testing](#9-testing)
10. [Documentation](#10-documentation)
11. [Security](#11-security)
12. [Performance](#12-performance)

## 1. Project Structure

Use the following project structure for our Django API project:

```bash
src/
├── manage.py
├── requirements.txt
├── .gitignore
├── README.md
├── .env
├── mypy.ini
├── project_name/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
└── api/
    ├── __init__.py
    ├── apps.py
    ├── models.py
    ├── serializers.py
    ├── views.py
    ├── urls.py
    ├── permissions.py
    ├── filters.py
    ├── utils.py
    └── tests/
        ├── __init__.py
        ├── test_models.py
        ├── test_views.py
        └── test_serializers.py
```

- All API-related code resides in the `api` app.
- Use separate files for models, serializers, views, etc., within the `api` app.
- Group all tests in a `tests` directory within the `api` app.

## 2. Python and Django Style Guide

- Follow PEP 8 style guide for Python code.
- Use 4 spaces for indentation (no tabs).
- Limit lines to 80 characters maximum.
- Use `snake_case` for function and variable names.
- Use `PascalCase` for class names.
- Use `UPPERCASE` for constants.

```python
# Good
def calculate_total(price: float, quantity: int) -> float:
    return price * quantity

class ItemManager:
    def get_discounted_price(self, item: 'Item') -> float:
        # Implementation here
        pass

MAX_ITEMS_PER_ORDER = 10
```

- Use type hints for all function arguments and return values.
- Use docstrings for all classes, methods, and functions.

```python
def get_active_items() -> list[Item]:
    """
    Retrieve all active items from the database.

    Returns:
        list[Item]: A list of active Item instances.
    """
    return Item.objects.filter(is_active=True)
```

## 3. API Design

- Use REST principles for API design.
- Use plural nouns for resource endpoints (e.g., `/api/v1/items/` not `/api/v1/item/`).
- Use appropriate HTTP methods:
  - GET for retrieving resources
  - POST for server actions
  - PUT for creating or updating resources
  - DELETE for removing resources
- Use proper HTTP status codes:
  - 200 for successful GET, PUT or DELETE
  - 201 for successful POST
  - 204 for successful DELETE with no content returned
  - 400 for bad requests
  - 401 for unauthorized access
  - 403 for forbidden actions
  - 404 for resources not found
  - 500 for server errors
- Implement versioning in the URL (e.g., `/api/v1/`).
- Use query parameters for filtering, sorting, and pagination.
- Use nested routes for related resources when appropriate.

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ItemViewSet, OrderViewSet

router = DefaultRouter()
router.register(r'items', ItemViewSet)
router.register(r'orders', OrderViewSet)

urlpatterns = [
    path('v1/', include(router.urls)),
]
```

## 4. Models and Database

- Use meaningful and descriptive names for models and fields.
- Implement proper indexes for frequently queried fields.
- Use appropriate field types and constraints.
- Implement custom model methods and properties when needed.
- Use model managers for complex queries.

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Item(models.Model):
    name = models.CharField(max_length=100, db_index=True)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    is_active = models.BooleanField(default=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return self.name

    @property
    def is_discounted(self):
        return self.discount_set.filter(is_active=True).exists()

class ItemManager(models.Manager):
    def active(self):
        return self.filter(is_active=True)

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    items = models.ManyToManyField(Item, through='OrderItem')
    total = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

    objects = models.Manager()
    active_orders = ItemManager()
```

## 5. Serializers

- Use Django Rest Framework serializers for data validation and formatting.
- Create nested serializers for related objects when appropriate.
- Use custom methods for complex data transformations.

```python
# api/serializers.py
from rest_framework import serializers
from .models import Item, Order

class ItemSerializer(serializers.ModelSerializer):
    is_discounted = serializers.BooleanField(read_only=True)

    class Meta:
        model = Item
        fields = ['id', 'name', 'description', 'price', 'is_active', 'is_discounted', 'created_at']
        read_only_fields = ['created_at']

class OrderSerializer(serializers.ModelSerializer):
    items = ItemSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'user', 'items', 'total', 'created_at']
        read_only_fields = ['created_at']
```

## 6. Views and ViewSets

- Use Django Rest Framework's ViewSets for consistent API actions.
- Implement custom actions using `@action` decorator when needed.
- Use appropriate permission classes for views.
- Utilize DRF's built-in features like pagination, filtering, and sorting.

```python
# api/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Item, Order
from .serializers import ItemSerializer, OrderSerializer
from .permissions import IsAdminOrReadOnly

class ItemViewSet(viewsets.ModelViewSet):
    queryset = Item.objects.all()
    serializer_class = ItemSerializer
    permission_classes = [IsAdminOrReadOnly]
    filterset_fields = ['is_active', 'price']
    search_fields = ['name', 'description']
    ordering_fields = ['created_at', 'price']

    @action(detail=True, methods=['post'])
    def discount(self, request, pk=None):
        item = self.get_object()
        # Apply discount logic here
        return Response({'status': 'discount applied'})

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    # Implement appropriate permissions and filters
```

## 7. URLs and Routing

- Use Django Rest Framework's DefaultRouter for automatic URL routing.
- Group API URLs under a common prefix (e.g., `/api/v1/`).
- Use include() for modular URL configurations.

```python
# project_name/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/', include('api.urls')),
]

# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ItemViewSet, OrderViewSet

router = DefaultRouter()
router.register(r'items', ItemViewSet)
router.register(r'orders', OrderViewSet)

urlpatterns = [
    path('v1/', include(router.urls)),
]
```

## 8. Authentication and Permissions

- Use Django Rest Framework's built-in authentication classes
(e.g., TokenAuthentication, JWTAuthentication).
- Implement custom permission classes when necessary.
- Use Django's built-in User model or a custom user model.

```python
# api/permissions.py
from rest_framework import permissions

class IsAdminOrReadOnly(permissions.BasePermission):
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff
```

## 9. Testing

- Write unit tests for models, serializers, and views.
- Use Django's TestCase for tests that require database access.
- Use APITestCase for testing API endpoints.
- Aim for high test coverage, especially for core functionality.

```python
# api/tests/test_views.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.urls import reverse
from .factories import ItemFactory

class ItemViewSetTestCase(APITestCase):
    def setUp(self):
        self.item = ItemFactory()

    def test_list_items(self):
        url = reverse('item-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 1)

    def test_create_item(self):
        url = reverse('item-list')
        data = {'name': 'New Item', 'price': '10.00'}
        response = self.client.post(url, data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
```

## 10. Documentation

- Use docstrings for all classes, methods, and functions.
- Maintain a comprehensive README.md file with setup instructions and API overview.
- Use tools like drf-yasg for automatic API documentation.

```python
# api/views.py
from drf_yasg.utils import swagger_auto_schema
from drf_yasg import openapi

class ItemViewSet(viewsets.ModelViewSet):
    """
    ViewSet for viewing and editing Item instances.
    """

    @swagger_auto_schema(
        operation_description="List all items",
        responses={200: ItemSerializer(many=True)}
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

## 11. Security

- Keep Django and all dependencies up-to-date.
- Use environment variables for sensitive information.
- Implement proper CORS settings.
- Use HTTPS in production.
- Implement rate limiting to prevent abuse.

```python
# settings.py
import os
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', cast=lambda v: [s.strip() for s in v.split(',')])

# CORS settings
CORS_ALLOW_ALL_ORIGINS = False
CORS_ALLOWED_ORIGINS = config('CORS_ALLOWED_ORIGINS', cast=lambda v: [s.strip() for s in v.split(',')])

# Ensure HTTPS in production
if not DEBUG:
    SECURE_SSL_REDIRECT = True
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
```

## 12. Performance

- Use Django's caching framework.
- Optimize database queries (select_related, prefetch_related).
- Use pagination for large result sets.
- Implement database indexing wisely.
- Use asynchronous tasks for long-running operations (e.g., Celery).

```python
# api/views.py
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page

class ItemViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Item.objects.select_related('category').all()
    serializer_class = ItemSerializer
    pagination_class = StandardResultsSetPagination

    @method_decorator(cache_page(60 * 15))  # Cache for 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```
