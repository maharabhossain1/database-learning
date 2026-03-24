# Complete Django REST Framework Mastery Guide
**From Basics to Enterprise-Level Backend Engineering**

> A comprehensive guide to master Django REST Framework (DRF) - covering serialization, views, performance optimization, caching, background tasks, and production-ready patterns used by top companies.

---

## Table of Contents

1. [DRF Fundamentals & Architecture](#1-drf-fundamentals--architecture)
2. [Serializers - The Heart of DRF](#2-serializers---the-heart-of-drf)
3. [Views & ViewSets](#3-views--viewsets)
4. [Authentication & Permissions](#4-authentication--permissions)
5. [Filtering, Searching & Ordering](#5-filtering-searching--ordering)
6. [Pagination Strategies](#6-pagination-strategies)
7. [API Performance Optimization](#7-api-performance-optimization)
8. [Caching with Redis](#8-caching-with-redis)
9. [Background Tasks with Celery](#9-background-tasks-with-celery)
10. [Error Handling & Validation](#10-error-handling--validation)
11. [API Versioning](#11-api-versioning)
12. [Testing APIs](#12-testing-apis)
13. [Production Best Practices](#13-production-best-practices)
14. [Common Errors & Solutions](#14-common-errors--solutions)
15. [Real-World Patterns (GTAF)](#15-real-world-patterns)

---

## 1. DRF Fundamentals & Architecture

### 1.1 What is Django REST Framework?

DRF is a powerful toolkit for building Web APIs in Django. It handles:
- **Serialization**: Converting complex Django models to JSON/XML
- **Deserialization**: Converting JSON/XML to Django models
- **Validation**: Ensuring data integrity
- **Authentication & Permissions**: Securing your API
- **Browsable API**: Auto-generated interactive documentation

**The Request-Response Flow:**
```
Client Request (JSON)
    ↓
Authentication/Permission Check
    ↓
View/ViewSet (Business Logic)
    ↓
Serializer (Validate & Transform)
    ↓
Database Query (ORM)
    ↓
Serializer (Transform to JSON)
    ↓
Response (JSON)
```

### 1.2 Installation & Setup

```bash
# Install DRF
pip install djangorestframework
pip install django-filter  # For filtering
pip install djangorestframework-simplejwt  # For JWT auth

# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',
    ],
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

### 1.3 Basic API Example

**models.py:**
```python
from django.db import models

class Product(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField(default=0)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.name
```

**serializers.py:**
```python
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'stock', 'is_active', 'created_at']
        read_only_fields = ['id', 'created_at']
```

**views.py:**
```python
from rest_framework import viewsets
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

**urls.py:**
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet

router = DefaultRouter()
router.register(r'products', ProductViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

**Result:**
```
GET    /api/products/          # List all products
POST   /api/products/          # Create product
GET    /api/products/{id}/     # Retrieve product
PUT    /api/products/{id}/     # Update product
PATCH  /api/products/{id}/     # Partial update
DELETE /api/products/{id}/     # Delete product
```

---

## 2. Serializers - The Heart of DRF

### 2.1 Why Serialization?

**Problem:**
```python
# Django model instance
product = Product.objects.get(id=1)
# Product object is not JSON serializable!
# You can't send it directly to frontend
```

**Solution:**
```python
# Serializer converts model to JSON
serializer = ProductSerializer(product)
serializer.data
# {'id': 1, 'name': 'iPhone', 'price': '999.99', ...}
# Now it's JSON serializable!
```

### 2.2 Serializer Types

#### ModelSerializer (Most Common)

```python
from rest_framework import serializers
from .models import Product, Category

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'  # All fields
        # OR
        fields = ['id', 'name', 'price']  # Specific fields
        # OR
        exclude = ['created_at', 'updated_at']  # Exclude fields
        
        read_only_fields = ['id', 'created_at']
        
        # Extra kwargs for field-level settings
        extra_kwargs = {
            'price': {'min_value': 0},
            'description': {'required': False},
        }
```

#### Serializer (Custom)

```python
class CustomProductSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    name = serializers.CharField(max_length=200)
    price = serializers.DecimalField(max_digits=10, decimal_places=2)
    
    def create(self, validated_data):
        return Product.objects.create(**validated_data)
    
    def update(self, instance, validated_data):
        instance.name = validated_data.get('name', instance.name)
        instance.price = validated_data.get('price', instance.price)
        instance.save()
        return instance
```

### 2.3 Nested Serializers (Relationships)

**Models:**
```python
class Category(models.Model):
    name = models.CharField(max_length=100)

class Product(models.Model):
    name = models.CharField(max_length=200)
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='products')
    price = models.DecimalField(max_digits=10, decimal_places=2)
```

**Method 1: Nested Objects (Read-Only)**
```python
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']

class ProductSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)  # Nested serializer
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'category', 'price']

# Output:
# {
#     "id": 1,
#     "name": "iPhone",
#     "category": {"id": 1, "name": "Electronics"},
#     "price": "999.99"
# }
```

**Method 2: PrimaryKeyRelatedField (Write)**
```python
class ProductSerializer(serializers.ModelSerializer):
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all()
    )
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'category', 'price']

# Input:
# {
#     "name": "iPhone",
#     "category": 1,  # Just the ID
#     "price": "999.99"
# }
```

**Method 3: Read Nested, Write ID (Best Practice)**
```python
class ProductSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True
    )
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'category', 'category_id', 'price']

# Read: Returns nested category object
# Write: Accepts category_id
```

### 2.4 SerializerMethodField (Custom Fields)

```python
class ProductSerializer(serializers.ModelSerializer):
    # Custom computed field
    display_price = serializers.SerializerMethodField()
    is_available = serializers.SerializerMethodField()
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'display_price', 'is_available', 'category_name']
    
    def get_display_price(self, obj):
        """Add currency symbol"""
        return f"${obj.price}"
    
    def get_is_available(self, obj):
        """Check if product is available"""
        return obj.stock > 0 and obj.is_active

# Output:
# {
#     "id": 1,
#     "name": "iPhone",
#     "price": "999.99",
#     "display_price": "$999.99",
#     "is_available": true,
#     "category_name": "Electronics"
# }
```

### 2.5 Performance: N+1 Problem in Serializers

**❌ BAD: Causes N+1 queries**
```python
class ProductSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name')
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'category_name', 'price']

# In view
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()  # N+1 problem!
    serializer_class = ProductSerializer
```

**✅ GOOD: Use select_related**
```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').all()  # Solved!
    serializer_class = ProductSerializer
```

**For Reverse Relations: Use prefetch_related**
```python
class CategorySerializer(serializers.ModelSerializer):
    products = ProductSerializer(many=True, read_only=True)
    product_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'products', 'product_count']

class CategoryViewSet(viewsets.ModelViewSet):
    queryset = Category.objects.prefetch_related('products').annotate(
        product_count=Count('products')
    )
    serializer_class = CategorySerializer
```

### 2.6 Dynamic Fields (Advanced)

```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    """Serializer that allows dynamic field selection"""
    
    def __init__(self, *args, **kwargs):
        # Get fields parameter from context
        fields = kwargs.pop('fields', None)
        super().__init__(*args, **kwargs)
        
        if fields:
            # Drop fields not specified
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)

class ProductSerializer(DynamicFieldsSerializer):
    class Meta:
        model = Product
        fields = '__all__'

# Usage in view:
# GET /api/products/?fields=id,name,price
# Only returns id, name, price
```

### 2.7 Validation (Custom)

```python
class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'
    
    # Field-level validation
    def validate_price(self, value):
        """Validate individual field"""
        if value < 0:
            raise serializers.ValidationError("Price cannot be negative")
        if value > 100000:
            raise serializers.ValidationError("Price too high")
        return value
    
    # Object-level validation
    def validate(self, data):
        """Validate multiple fields together"""
        if data.get('price', 0) > 1000 and data.get('stock', 0) > 100:
            raise serializers.ValidationError(
                "High-priced items cannot have stock > 100"
            )
        return data
    
    # Custom create logic
    def create(self, validated_data):
        # Add custom logic before/after creation
        user = self.context['request'].user
        validated_data['created_by'] = user
        return super().create(validated_data)
    
    # Custom update logic
    def update(self, instance, validated_data):
        # Don't allow price changes for published products
        if instance.is_published:
            validated_data.pop('price', None)
        return super().update(instance, validated_data)
```

### 2.8 Multiple Serializers for Same Model

```python
class ProductListSerializer(serializers.ModelSerializer):
    """Lightweight for listing"""
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'price', 'category_name']

class ProductDetailSerializer(serializers.ModelSerializer):
    """Detailed for single product"""
    category = CategorySerializer(read_only=True)
    reviews = ReviewSerializer(many=True, read_only=True)
    average_rating = serializers.FloatField(read_only=True)
    
    class Meta:
        model = Product
        fields = '__all__'

class ProductCreateSerializer(serializers.ModelSerializer):
    """For creating products"""
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'category', 'stock']

# Usage in ViewSet
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    
    def get_serializer_class(self):
        if self.action == 'list':
            return ProductListSerializer
        elif self.action == 'retrieve':
            return ProductDetailSerializer
        elif self.action == 'create':
            return ProductCreateSerializer
        return ProductDetailSerializer
```

---

## 3. Views & ViewSets

### 3.1 View Types Hierarchy

```
APIView (Most Flexible, Most Code)
    ↓
GenericAPIView (+ Mixins)
    ↓
Concrete Views (ListAPIView, CreateAPIView, etc.)
    ↓
ViewSet (Combines multiple views)
    ↓
ModelViewSet (Least Code, Least Flexible)
```

### 3.2 APIView (Lowest Level)

**When to use:** Maximum control, custom logic

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ProductListAPIView(APIView):
    """
    List all products or create new product
    """
    def get(self, request):
        products = Product.objects.all()
        serializer = ProductSerializer(products, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ProductDetailAPIView(APIView):
    """
    Retrieve, update or delete product
    """
    def get_object(self, pk):
        try:
            return Product.objects.get(pk=pk)
        except Product.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        product = self.get_object(pk)
        serializer = ProductSerializer(product)
        return Response(serializer.data)
    
    def put(self, request, pk):
        product = self.get_object(pk)
        serializer = ProductSerializer(product, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        product = self.get_object(pk)
        product.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)

# urls.py
urlpatterns = [
    path('products/', ProductListAPIView.as_view()),
    path('products/<int:pk>/', ProductDetailAPIView.as_view()),
]
```

### 3.3 Generic Views (Less Code)

**When to use:** Standard CRUD with less boilerplate

```python
from rest_framework import generics

# List all products
class ProductListView(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at']

# Create product
class ProductCreateView(generics.CreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# List and Create in one
class ProductListCreateView(generics.ListCreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# Retrieve single product
class ProductDetailView(generics.RetrieveAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# Update product
class ProductUpdateView(generics.UpdateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# Delete product
class ProductDeleteView(generics.DestroyAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

# Retrieve, Update, Delete in one
class ProductDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

### 3.4 ViewSets (Most Powerful)

**When to use:** RESTful resources with standard CRUD

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    """
    A viewset that provides default `create()`, `retrieve()`,
    `update()`, `partial_update()`, `destroy()` and `list()` actions.
    """
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    # Custom action (non-CRUD)
    @action(detail=False, methods=['get'])
    def featured(self, request):
        """GET /api/products/featured/"""
        featured_products = self.queryset.filter(is_featured=True)
        serializer = self.get_serializer(featured_products, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'])
    def publish(self, request, pk=None):
        """POST /api/products/{id}/publish/"""
        product = self.get_object()
        product.is_published = True
        product.save()
        return Response({'status': 'product published'})
    
    @action(detail=True, methods=['get'])
    def reviews(self, request, pk=None):
        """GET /api/products/{id}/reviews/"""
        product = self.get_object()
        reviews = product.reviews.all()
        serializer = ReviewSerializer(reviews, many=True)
        return Response(serializer.data)
    
    # Override default behavior
    def perform_create(self, serializer):
        """Called when creating object"""
        serializer.save(created_by=self.request.user)
    
    def perform_update(self, serializer):
        """Called when updating object"""
        serializer.save(updated_by=self.request.user)
    
    def get_queryset(self):
        """Dynamically filter queryset"""
        queryset = super().get_queryset()
        
        # Filter by query params
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category__name=category)
        
        # Only show published products for non-staff
        if not self.request.user.is_staff:
            queryset = queryset.filter(is_published=True)
        
        return queryset
    
    def get_serializer_class(self):
        """Use different serializers for different actions"""
        if self.action == 'list':
            return ProductListSerializer
        elif self.action == 'retrieve':
            return ProductDetailSerializer
        return ProductSerializer

# urls.py with router
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'products', ProductViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]

# This generates:
# GET    /api/products/               -> list
# POST   /api/products/               -> create
# GET    /api/products/{id}/          -> retrieve
# PUT    /api/products/{id}/          -> update
# PATCH  /api/products/{id}/          -> partial_update
# DELETE /api/products/{id}/          -> destroy
# GET    /api/products/featured/      -> featured (custom)
# POST   /api/products/{id}/publish/  -> publish (custom)
# GET    /api/products/{id}/reviews/  -> reviews (custom)
```

### 3.5 ReadOnlyModelViewSet

**When to use:** Only need read operations

```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    """
    Only provides 'list' and 'retrieve' actions
    """
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
```

### 3.6 Custom ViewSet Actions

```python
class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def cancel(self, request, pk=None):
        """POST /api/orders/{id}/cancel/"""
        order = self.get_object()
        
        # Check if user owns the order
        if order.user != request.user:
            return Response(
                {'error': 'Not authorized'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        # Check if order can be cancelled
        if order.status not in ['pending', 'confirmed']:
            return Response(
                {'error': 'Order cannot be cancelled'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        order.status = 'cancelled'
        order.save()
        
        # Send notification (celery task)
        send_order_cancelled_email.delay(order.id)
        
        return Response({'status': 'order cancelled'})
    
    @action(detail=False, methods=['get'])
    def my_orders(self, request):
        """GET /api/orders/my_orders/"""
        user_orders = self.queryset.filter(user=request.user)
        page = self.paginate_queryset(user_orders)
        serializer = self.get_serializer(page, many=True)
        return self.get_paginated_response(serializer.data)
    
    @action(detail=False, methods=['post'])
    def bulk_create(self, request):
        """POST /api/orders/bulk_create/"""
        serializer = self.get_serializer(data=request.data, many=True)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### 3.7 Mixing ViewSet with APIView

```python
from rest_framework.views import APIView
from rest_framework.viewsets import ViewSet

class StatisticsViewSet(ViewSet):
    """
    Custom ViewSet with no model
    """
    permission_classes = [IsAuthenticated, IsAdminUser]
    
    def list(self, request):
        """GET /api/statistics/"""
        stats = {
            'total_users': User.objects.count(),
            'total_orders': Order.objects.count(),
            'revenue_today': Order.objects.filter(
                created_at__date=date.today()
            ).aggregate(total=Sum('total_amount'))['total'] or 0
        }
        return Response(stats)
    
    @action(detail=False, methods=['get'])
    def sales_report(self, request):
        """GET /api/statistics/sales_report/?start_date=2024-01-01&end_date=2024-12-31"""
        start_date = request.query_params.get('start_date')
        end_date = request.query_params.get('end_date')
        
        orders = Order.objects.filter(
            created_at__date__range=[start_date, end_date]
        ).values('created_at__date').annotate(
            daily_total=Sum('total_amount'),
            order_count=Count('id')
        ).order_by('created_at__date')
        
        return Response(orders)
```

---

## 4. Authentication & Permissions

### 4.1 Authentication Types

#### Session Authentication (Default)
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}

# Use with cookies, good for web apps
```

#### Token Authentication
```python
# Install
pip install djangorestframework-simplejwt

# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}

# Migrate
python manage.py migrate

# Create tokens
from rest_framework.authtoken.models import Token

token = Token.objects.create(user=user)
print(token.key)

# Usage in requests
# Headers: Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b
```

#### JWT Authentication (Best for Modern APIs)
```python
# Install
pip install djangorestframework-simplejwt

# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
}

# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]

# Usage:
# POST /api/token/ with {"username": "...", "password": "..."}
# Returns: {"access": "...", "refresh": "..."}

# Use access token in headers:
# Authorization: Bearer <access_token>
```

#### Custom JWT Claims
```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # Add custom claims
        token['username'] = user.username
        token['email'] = user.email
        token['is_staff'] = user.is_staff
        
        return token

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

### 4.2 Permission Classes

#### Built-in Permissions

```python
from rest_framework.permissions import (
    IsAuthenticated,
    IsAuthenticatedOrReadOnly,
    AllowAny,
    IsAdminUser,
)

# Apply globally (settings.py)
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# Apply to specific view
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]  # Read: anyone, Write: authenticated

# Apply to specific action
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    def get_permissions(self):
        if self.action in ['create', 'update', 'destroy']:
            return [IsAdminUser()]
        return [AllowAny()]
```

#### Custom Permissions

```python
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission: Object owner can edit, others can only read
    """
    def has_object_permission(self, request, view, obj):
        # Read permissions are allowed for any request
        if request.method in permissions.SAFE_METHODS:  # GET, HEAD, OPTIONS
            return True
        
        # Write permissions only for owner
        return obj.owner == request.user

class IsStaffOrReadOnly(permissions.BasePermission):
    """
    Staff can do anything, others can only read
    """
    def has_permission(self, request, view):
        if request.method in permissions.SAFE_METHODS:
            return True
        return request.user and request.user.is_staff

class IsAuthorOrAdmin(permissions.BasePermission):
    """
    Author of object or admin can access
    """
    def has_object_permission(self, request, view, obj):
        # Admin has all permissions
        if request.user and request.user.is_staff:
            return True
        
        # Author can access their own objects
        return obj.author == request.user

# Usage
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
```

#### Complex Permission Logic

```python
class IsPremiumUser(permissions.BasePermission):
    """
    Check if user has premium subscription
    """
    message = 'You need a premium subscription to access this resource.'
    
    def has_permission(self, request, view):
        return (
            request.user and
            request.user.is_authenticated and
            hasattr(request.user, 'subscription') and
            request.user.subscription.is_active and
            request.user.subscription.plan == 'premium'
        )

class CanAccessCourse(permissions.BasePermission):
    """
    Check if user has purchased or enrolled in course
    """
    def has_object_permission(self, request, view, obj):
        # Admin can access all
        if request.user.is_staff:
            return True
        
        # Check if user purchased the course
        return obj.enrollments.filter(user=request.user, is_paid=True).exists()
```

### 4.3 Throttling (Rate Limiting)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',      # Anonymous users: 100 requests per day
        'user': '1000/day',     # Authenticated users: 1000 requests per day
        'premium': '10000/day', # Premium users: 10000 requests per day
    }
}

# Custom throttle
from rest_framework.throttling import UserRateThrottle

class PremiumUserThrottle(UserRateThrottle):
    scope = 'premium'
    
    def allow_request(self, request, view):
        # Only apply to premium users
        if not hasattr(request.user, 'subscription'):
            return True
        
        if request.user.subscription.plan != 'premium':
            return True
        
        return super().allow_request(request, view)

# Apply to specific view
class PremiumContentViewSet(viewsets.ModelViewSet):
    queryset = PremiumContent.objects.all()
    serializer_class = PremiumContentSerializer
    throttle_classes = [PremiumUserThrottle]
```

---

## 5. Filtering, Searching & Ordering

### 5.1 Django Filter Backend

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}

# Simple filtering
from django_filters.rest_framework import DjangoFilterBackend

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['category', 'is_active', 'price']

# Usage:
# GET /api/products/?category=1
# GET /api/products/?is_active=true
# GET /api/products/?price=999.99
```

### 5.2 Custom FilterSet

```python
import django_filters
from django_filters import rest_framework as filters

class ProductFilter(filters.FilterSet):
    # Exact match
    category = filters.NumberFilter(field_name='category__id')
    
    # Range filters
    min_price = filters.NumberFilter(field_name='price', lookup_expr='gte')
    max_price = filters.NumberFilter(field_name='price', lookup_expr='lte')
    
    # Date filters
    created_after = filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = filters.DateFilter(field_name='created_at', lookup_expr='lte')
    
    # Multiple choice
    status = filters.MultipleChoiceFilter(
        choices=[('active', 'Active'), ('inactive', 'Inactive')]
    )
    
    # Boolean
    is_featured = filters.BooleanFilter(field_name='is_featured')
    
    # Custom method filter
    search = filters.CharFilter(method='filter_search')
    
    def filter_search(self, queryset, name, value):
        """Custom search across multiple fields"""
        return queryset.filter(
            Q(name__icontains=value) |
            Q(description__icontains=value) |
            Q(category__name__icontains=value)
        )
    
    class Meta:
        model = Product
        fields = ['category', 'is_active', 'min_price', 'max_price']

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = ProductFilter

# Usage:
# GET /api/products/?min_price=100&max_price=1000
# GET /api/products/?created_after=2024-01-01
# GET /api/products/?search=iphone
```

### 5.3 Search Filter

```python
from rest_framework import filters

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [filters.SearchFilter]
    search_fields = ['name', 'description', 'category__name']

# Usage:
# GET /api/products/?search=iphone
# Searches in name, description, and category name

# Advanced search configurations:
search_fields = [
    'name',                    # Exact match (contains)
    '^name',                   # Starts with
    '=name',                   # Exact match
    '@description',            # Full text search (PostgreSQL)
    '$name',                   # Regex search
]
```

### 5.4 Ordering Filter

```python
from rest_framework import filters

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['price', 'created_at', 'name']
    ordering = ['-created_at']  # Default ordering

# Usage:
# GET /api/products/?ordering=price         # Ascending
# GET /api/products/?ordering=-price        # Descending
# GET /api/products/?ordering=price,-created_at  # Multiple
```

### 5.5 Combining All Filters

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').all()
    serializer_class = ProductSerializer
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter,
    ]
    filterset_class = ProductFilter
    search_fields = ['name', 'description']
    ordering_fields = ['price', 'created_at', 'name']
    ordering = ['-created_at']

# Usage (combine all):
# GET /api/products/?category=1&min_price=100&search=phone&ordering=-price
```

---

## 6. Pagination Strategies

### 6.1 PageNumberPagination (Default)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20
}

# Usage:
# GET /api/products/?page=1
# GET /api/products/?page=2

# Response:
# {
#     "count": 100,
#     "next": "http://api.example.com/products/?page=2",
#     "previous": null,
#     "results": [...]
# }
```

### 6.2 Custom PageNumberPagination

```python
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'  # Allow client to set page size
    max_page_size = 100
    
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'results': data
        })

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    pagination_class = StandardResultsSetPagination

# Usage:
# GET /api/products/?page=1&page_size=50
```

### 6.3 LimitOffsetPagination

```python
from rest_framework.pagination import LimitOffsetPagination

class StandardLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    pagination_class = StandardLimitOffsetPagination

# Usage:
# GET /api/products/?limit=20&offset=40
# Skip 40 items, return next 20
```

### 6.4 CursorPagination (Best for Large Datasets)

```python
from rest_framework.pagination import CursorPagination

class ProductCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Must have ordering
    cursor_query_param = 'cursor'

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    pagination_class = ProductCursorPagination

# Usage:
# GET /api/products/
# Response includes encoded cursor for next page
# GET /api/products/?cursor=cD0yMDI0LTA...

# Benefits:
# - O(1) performance (doesn't count total)
# - Works well with real-time data
# - Prevents duplicate records during pagination
```

### 6.5 No Pagination for Specific Endpoints

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    def get_pagination_class(self):
        if self.action == 'list_all':
            return None  # No pagination
        return StandardResultsSetPagination
    
    @action(detail=False, methods=['get'])
    def list_all(self, request):
        """GET /api/products/list_all/ - Returns all products without pagination"""
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

---

## 7. API Performance Optimization

### 7.1 Select & Prefetch Related (Critical!)

```python
# ❌ BAD: N+1 queries
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()  # This causes N+1!
    serializer_class = ProductSerializer

class ProductSerializer(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name')
    manufacturer_name = serializers.CharField(source='manufacturer.name')

# ✅ GOOD: Optimized
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related(
        'category',
        'manufacturer'
    ).all()
    serializer_class = ProductSerializer

# ✅ GOOD: For reverse relations
class CategoryViewSet(viewsets.ModelViewSet):
    queryset = Category.objects.prefetch_related(
        'products'
    ).annotate(
        product_count=Count('products')
    )
    serializer_class = CategorySerializer
```

### 7.2 Use only() and defer()

```python
# Load only needed fields
class ProductListViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Product.objects.only(
        'id', 'name', 'price', 'category_id'
    ).select_related('category')
    serializer_class = ProductListSerializer

# Defer heavy fields
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.defer(
        'description',  # Large text field
        'specifications'  # JSON field
    )
    serializer_class = ProductSerializer
```

### 7.3 Database Indexing

```python
# models.py
class Product(models.Model):
    name = models.CharField(max_length=200, db_index=True)
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    price = models.DecimalField(max_digits=10, decimal_places=2, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['category', 'price']),  # Composite index
            models.Index(fields=['-created_at']),  # For ordering
            models.Index(
                fields=['price'],
                name='active_product_price_idx',
                condition=Q(is_active=True)  # Partial index
            ),
        ]
```

### 7.4 Use values() or values_list() When Possible

```python
# ❌ SLOW: Full model instances
products = Product.objects.all()
data = [{'id': p.id, 'name': p.name} for p in products]

# ✅ FAST: Direct dictionary
data = Product.objects.values('id', 'name')

# ✅ FAST: Tuples
data = Product.objects.values_list('id', 'name')

# In ViewSet
class ProductNamesViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Product.objects.values('id', 'name', 'price')
    serializer_class = None  # No serializer needed
    
    def list(self, request):
        queryset = self.filter_queryset(self.get_queryset())
        return Response(list(queryset))
```

### 7.5 Batch Database Operations

```python
# ❌ SLOW: N queries
for item in items:
    Product.objects.create(**item)

# ✅ FAST: 1 query
Product.objects.bulk_create([
    Product(**item) for item in items
], batch_size=500)

# ✅ FAST: Bulk update
products = Product.objects.filter(category_id=1)
for product in products:
    product.is_featured = True
Product.objects.bulk_update(products, ['is_featured'], batch_size=500)
```

### 7.6 Use Aggregation at Database Level

```python
# ❌ SLOW: Python-level calculation
products = Product.objects.all()
total = sum(p.price for p in products)

# ✅ FAST: Database-level
total = Product.objects.aggregate(total=Sum('price'))['total']

# ✅ Complex aggregations
stats = Product.objects.aggregate(
    total_products=Count('id'),
    avg_price=Avg('price'),
    max_price=Max('price'),
    total_value=Sum(F('price') * F('stock'))
)
```

### 7.7 Connection Pooling

```python
# Install
pip install django-db-connection-pool

# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'dj_db_conn_pool.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,
            'MAX_OVERFLOW': 10,
            'RECYCLE': 24 * 60 * 60,  # 24 hours
        }
    }
}
```

### 7.8 Query Debugging & Monitoring

```python
# Count queries in view
from django.db import connection
from django.test.utils import CaptureQueriesContext

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    def list(self, request, *args, **kwargs):
        with CaptureQueriesContext(connection) as queries:
            response = super().list(request, *args, **kwargs)
        
        # Log query count
        print(f"Queries executed: {len(queries)}")
        for query in queries:
            print(query['sql'])
        
        return response

# Use Django Debug Toolbar for development
# See all queries, cache hits, timing, etc.
```

---

## 8. Caching with Redis

### 8.1 Setup Redis

```bash
# Install Redis
# Ubuntu
sudo apt install redis-server

# macOS
brew install redis

# Start Redis
redis-server

# Install Python packages
pip install redis django-redis
```

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myapp',
        'TIMEOUT': 60 * 15,  # 15 minutes default
    }
}

# Session cache
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

### 8.2 View-Level Caching

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from rest_framework.decorators import action

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    # Cache for 15 minutes
    @method_decorator(cache_page(60 * 15))
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
    
    # Cache specific action
    @method_decorator(cache_page(60 * 30))  # 30 minutes
    @action(detail=False, methods=['get'])
    def featured(self, request):
        products = self.queryset.filter(is_featured=True)
        serializer = self.get_serializer(products, many=True)
        return Response(serializer.data)
```

### 8.3 Manual Caching with Redis

```python
from django.core.cache import cache
from rest_framework.response import Response

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    def list(self, request, *args, **kwargs):
        # Build cache key based on query params
        cache_key = f"products_list_{request.query_params.urlencode()}"
        
        # Try to get from cache
        cached_data = cache.get(cache_key)
        if cached_data:
            return Response(cached_data)
        
        # If not in cache, get from database
        response = super().list(request, *args, **kwargs)
        
        # Store in cache for 15 minutes
        cache.set(cache_key, response.data, 60 * 15)
        
        return response
    
    def retrieve(self, request, *args, **kwargs):
        # Cache individual product
        product_id = kwargs.get('pk')
        cache_key = f"product_{product_id}"
        
        cached_product = cache.get(cache_key)
        if cached_product:
            return Response(cached_product)
        
        response = super().retrieve(request, *args, **kwargs)
        cache.set(cache_key, response.data, 60 * 60)  # 1 hour
        
        return response
```

### 8.4 Cache Invalidation

```python
from django.core.cache import cache
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    """Invalidate cache when product is saved"""
    # Clear specific product cache
    cache.delete(f"product_{instance.id}")
    
    # Clear list cache (all variations)
    cache.delete_pattern("products_list_*")
    
    # Clear category cache if needed
    cache.delete(f"category_{instance.category_id}_products")

@receiver(post_delete, sender=Product)
def invalidate_product_cache_on_delete(sender, instance, **kwargs):
    """Invalidate cache when product is deleted"""
    cache.delete(f"product_{instance.id}")
    cache.delete_pattern("products_list_*")

# In ViewSet
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    
    def perform_create(self, serializer):
        """Invalidate cache after creating"""
        instance = serializer.save()
        cache.delete_pattern("products_list_*")
        return instance
    
    def perform_update(self, serializer):
        """Invalidate cache after updating"""
        instance = serializer.save()
        cache.delete(f"product_{instance.id}")
        cache.delete_pattern("products_list_*")
        return instance
```

### 8.5 Caching Serializer Results

```python
class CachedProductSerializer(serializers.ModelSerializer):
    category_name = serializers.SerializerMethodField()
    
    class Meta:
        model = Product
        fields = '__all__'
    
    def get_category_name(self, obj):
        # Cache category name lookup
        cache_key = f"category_name_{obj.category_id}"
        category_name = cache.get(cache_key)
        
        if not category_name:
            category_name = obj.category.name
            cache.set(cache_key, category_name, 60 * 60)  # 1 hour
        
        return category_name
```

### 8.6 Query Result Caching

```python
from django.core.cache import cache

def get_featured_products():
    """Get featured products with caching"""
    cache_key = 'featured_products'
    products = cache.get(cache_key)
    
    if not products:
        products = list(
            Product.objects.filter(is_featured=True)
            .select_related('category')
            .values('id', 'name', 'price', 'category__name')
        )
        cache.set(cache_key, products, 60 * 15)  # 15 minutes
    
    return products

class ProductViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=['get'])
    def featured(self, request):
        products = get_featured_products()
        return Response(products)
```

### 8.7 Cache Warming (Preload Cache)

```python
from django.core.management.base import BaseCommand
from django.core.cache import cache
from myapp.models import Product, Category

class Command(BaseCommand):
    help = 'Warm up cache with frequently accessed data'
    
    def handle(self, *args, **options):
        # Cache featured products
        featured = list(
            Product.objects.filter(is_featured=True)
            .select_related('category')
            .values()
        )
        cache.set('featured_products', featured, 60 * 60)
        
        # Cache popular categories
        categories = list(Category.objects.all().values())
        cache.set('all_categories', categories, 60 * 60)
        
        # Cache individual popular products
        popular_products = Product.objects.filter(
            view_count__gt=1000
        ).select_related('category')
        
        for product in popular_products:
            serializer = ProductSerializer(product)
            cache.set(f"product_{product.id}", serializer.data, 60 * 60)
        
        self.stdout.write(self.style.SUCCESS('Cache warmed successfully'))

# Run with: python manage.py warm_cache
# Schedule with cron or Celery Beat
```

---

## 9. Background Tasks with Celery

### 9.1 Setup Celery

```bash
# Install
pip install celery redis
```

```python
# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# myproject/__init__.py
from .celery import app as celery_app

__all__ = ('celery_app',)

# settings.py
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60  # 30 minutes
```

```bash
# Start Celery worker
celery -A myproject worker --loglevel=info

# Start Celery beat (for scheduled tasks)
celery -A myproject beat --loglevel=info
```

### 9.2 Basic Task Example

```python
# myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail
from django.conf import settings

@shared_task
def send_welcome_email(user_id):
    """Send welcome email to new user"""
    from django.contrib.auth import get_user_model
    User = get_user_model()
    
    try:
        user = User.objects.get(id=user_id)
        send_mail(
            subject='Welcome to Our Platform!',
            message=f'Hi {user.username}, welcome!',
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[user.email],
            fail_silently=False,
        )
        return f"Email sent to {user.email}"
    except User.DoesNotExist:
        return f"User {user_id} not found"

@shared_task
def process_order(order_id):
    """Process order in background"""
    from .models import Order
    
    order = Order.objects.get(id=order_id)
    
    # Process payment
    payment_result = process_payment(order)
    
    # Update inventory
    update_inventory(order)
    
    # Send confirmation email
    send_order_confirmation_email.delay(order.id)
    
    return f"Order {order_id} processed"
```

### 9.3 Using Tasks in API Views

```python
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .tasks import send_welcome_email, process_order

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    
    def perform_create(self, serializer):
        """Create user and send welcome email asynchronously"""
        user = serializer.save()
        
        # Send email in background (non-blocking)
        send_welcome_email.delay(user.id)
        
        return user

class OrderViewSet(viewsets.ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    
    @action(detail=True, methods=['post'])
    def process(self, request, pk=None):
        """Process order asynchronously"""
        order = self.get_object()
        
        if order.status != 'pending':
            return Response(
                {'error': 'Order already processed'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Process in background
        task = process_order.delay(order.id)
        
        return Response({
            'message': 'Order processing started',
            'task_id': task.id
        })
    
    @action(detail=False, methods=['get'])
    def task_status(self, request):
        """Check task status"""
        task_id = request.query_params.get('task_id')
        
        from celery.result import AsyncResult
        task = AsyncResult(task_id)
        
        return Response({
            'task_id': task_id,
            'status': task.status,
            'result': task.result if task.ready() else None
        })
```

### 9.4 Email Sending with Celery

```python
# myapp/tasks.py
from celery import shared_task
from django.core.mail import send_mail, EmailMultiAlternatives
from django.template.loader import render_to_string
from django.utils.html import strip_tags

@shared_task
def send_order_confirmation_email(order_id):
    """Send order confirmation email"""
    from .models import Order
    
    order = Order.objects.select_related('user').prefetch_related(
        'items__product'
    ).get(id=order_id)
    
    # Render HTML email
    html_content = render_to_string('emails/order_confirmation.html', {
        'order': order,
        'user': order.user,
        'items': order.items.all()
    })
    text_content = strip_tags(html_content)
    
    email = EmailMultiAlternatives(
        subject=f'Order Confirmation #{order.order_number}',
        body=text_content,
        from_email=settings.DEFAULT_FROM_EMAIL,
        to=[order.user.email]
    )
    email.attach_alternative(html_content, "text/html")
    email.send()
    
    return f"Confirmation email sent for order {order_id}"

@shared_task
def send_bulk_emails(user_ids, subject, message):
    """Send email to multiple users"""
    from django.contrib.auth import get_user_model
    User = get_user_model()
    
    users = User.objects.filter(id__in=user_ids)
    emails_sent = 0
    
    for user in users:
        send_mail(
            subject=subject,
            message=message,
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=[user.email],
            fail_silently=True,
        )
        emails_sent += 1
    
    return f"Sent {emails_sent} emails"
```

### 9.5 SMS Sending with Celery

```python
# myapp/tasks.py
from celery import shared_task
import requests

@shared_task
def send_sms(phone_number, message):
    """Send SMS using external API"""
    # Using Twilio example
    from twilio.rest import Client
    
    client = Client(settings.TWILIO_ACCOUNT_SID, settings.TWILIO_AUTH_TOKEN)
    
    try:
        message = client.messages.create(
            body=message,
            from_=settings.TWILIO_PHONE_NUMBER,
            to=phone_number
        )
        return f"SMS sent: {message.sid}"
    except Exception as e:
        return f"SMS failed: {str(e)}"

@shared_task
def send_order_sms_notification(order_id):
    """Send SMS notification for order"""
    from .models import Order
    
    order = Order.objects.select_related('user').get(id=order_id)
    
    message = f"Your order #{order.order_number} has been confirmed. Total: ${order.total_amount}"
    
    if order.user.phone_number:
        send_sms.delay(order.user.phone_number, message)
    
    return f"SMS notification sent for order {order_id}"

# In ViewSet
class OrderViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def confirm(self, request, pk=None):
        order = self.get_object()
        order.status = 'confirmed'
        order.save()
        
        # Send email and SMS asynchronously
        send_order_confirmation_email.delay(order.id)
        send_order_sms_notification.delay(order.id)
        
        return Response({'status': 'order confirmed'})
```

### 9.6 Scheduled Tasks (Periodic)

```python
# myproject/celery.py
from celery import Celery
from celery.schedules import crontab

app = Celery('myproject')

app.conf.beat_schedule = {
    # Runs every day at midnight
    'cleanup-expired-sessions': {
        'task': 'myapp.tasks.cleanup_expired_sessions',
        'schedule': crontab(hour=0, minute=0),
    },
    # Runs every hour
    'update-product-recommendations': {
        'task': 'myapp.tasks.update_recommendations',
        'schedule': crontab(minute=0),
    },
    # Runs every Monday at 9 AM
    'send-weekly-report': {
        'task': 'myapp.tasks.send_weekly_report',
        'schedule': crontab(hour=9, minute=0, day_of_week=1),
    },
    # Runs every 15 minutes
    'sync-inventory': {
        'task': 'myapp.tasks.sync_inventory',
        'schedule': 900.0,  # seconds
    },
}

# myapp/tasks.py
@shared_task
def cleanup_expired_sessions():
    """Clean up expired sessions daily"""
    from django.contrib.sessions.models import Session
    from django.utils import timezone
    
    Session.objects.filter(expire_date__lt=timezone.now()).delete()
    return "Expired sessions cleaned"

@shared_task
def update_recommendations():
    """Update product recommendations hourly"""
    from .models import Product
    from .utils import calculate_recommendations
    
    products = Product.objects.filter(is_active=True)
    for product in products:
        recommendations = calculate_recommendations(product)
        # Update recommendations
    
    return "Recommendations updated"
```

### 9.7 Task Chaining & Groups

```python
from celery import chain, group, chord

# Chain: Execute tasks in sequence
result = chain(
    task1.s(arg1),
    task2.s(),
    task3.s()
).apply_async()

# Group: Execute tasks in parallel
result = group(
    send_email.s(user1.email),
    send_email.s(user2.email),
    send_email.s(user3.email)
).apply_async()

# Chord: Execute tasks in parallel, then callback
result = chord([
    process_item.s(item1),
    process_item.s(item2),
    process_item.s(item3),
])(finalize_processing.s())

# Example: Process order with multiple steps
@shared_task
def validate_order(order_id):
    # Validate order
    return order_id

@shared_task
def process_payment(order_id):
    # Process payment
    return order_id

@shared_task
def update_inventory(order_id):
    # Update inventory
    return order_id

@shared_task
def send_notifications(order_id):
    # Send email and SMS
    return order_id

# Chain them
def process_order_pipeline(order_id):
    chain(
        validate_order.s(order_id),
        process_payment.s(),
        update_inventory.s(),
        send_notifications.s()
    ).apply_async()
```

---

## 10. Error Handling & Validation

### 10.1 Custom Exception Handler

```python
# myapp/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import APIException
from rest_framework import status

def custom_exception_handler(exc, context):
    """Custom exception handler for better error messages"""
    # Call DRF's default exception handler first
    response = exception_handler(exc, context)
    
    if response is not None:
        # Customize error format
        custom_response = {
            'error': {
                'status_code': response.status_code,
                'message': response.data.get('detail', 'An error occurred'),
                'errors': response.data if isinstance(response.data, dict) else {}
            }
        }
        response.data = custom_response
    
    return response

# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'myapp.exceptions.custom_exception_handler',
}
```

### 10.2 Custom Exceptions

```python
from rest_framework.exceptions import APIException

class InsufficientStockException(APIException):
    status_code = 400
    default_detail = 'Insufficient stock available'
    default_code = 'insufficient_stock'

class PaymentFailedException(APIException):
    status_code = 402
    default_detail = 'Payment processing failed'
    default_code = 'payment_failed'

class OrderNotFoundException(APIException):
    status_code = 404
    default_detail = 'Order not found'
    default_code = 'order_not_found'

# Usage in views
class OrderViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def purchase(self, request, pk=None):
        order = self.get_object()
        
        # Check stock
        for item in order.items.all():
            if item.product.stock < item.quantity:
                raise InsufficientStockException(
                    detail=f"Insufficient stock for {item.product.name}"
                )
        
        # Process payment
        payment_result = process_payment(order)
        if not payment_result.success:
            raise PaymentFailedException(
                detail=payment_result.error_message
            )
        
        return Response({'status': 'order completed'})
```

### 10.3 Validation Errors

```python
from rest_framework import serializers
from rest_framework.exceptions import ValidationError

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'
    
    def validate_price(self, value):
        """Field-level validation"""
        if value < 0:
            raise ValidationError("Price cannot be negative")
        if value > 1000000:
            raise ValidationError("Price exceeds maximum allowed")
        return value
    
    def validate(self, data):
        """Object-level validation"""
        if data.get('price', 0) > 10000 and not data.get('requires_approval'):
            raise ValidationError({
                'requires_approval': 'High-value items require approval'
            })
        
        if data.get('stock', 0) < 0:
            raise ValidationError({
                'stock': 'Stock cannot be negative'
            })
        
        # Check if product name already exists
        if self.instance is None:  # Creating new product
            if Product.objects.filter(name=data['name']).exists():
                raise ValidationError({
                    'name': 'Product with this name already exists'
                })
        
        return data

# Response format for validation errors:
# {
#     "error": {
#         "status_code": 400,
#         "message": "Validation failed",
#         "errors": {
#             "price": ["Price cannot be negative"],
#             "stock": ["Stock cannot be negative"]
#         }
#     }
# }
```

### 10.4 Try-Except in Views

```python
from django.db import transaction
from rest_framework.exceptions import ValidationError

class OrderViewSet(viewsets.ModelViewSet):
    @transaction.atomic
    @action(detail=False, methods=['post'])
    def create_order(self, request):
        try:
            # Validate input
            serializer = self.get_serializer(data=request.data)
            serializer.is_valid(raise_exception=True)
            
            # Create order
            order = serializer.save(user=request.user)
            
            # Process items
            for item_data in request.data.get('items', []):
                product = Product.objects.select_for_update().get(
                    id=item_data['product_id']
                )
                
                # Check stock
                if product.stock < item_data['quantity']:
                    raise InsufficientStockException(
                        detail=f"Insufficient stock for {product.name}"
                    )
                
                # Update stock
                product.stock -= item_data['quantity']
                product.save()
                
                # Create order item
                OrderItem.objects.create(
                    order=order,
                    product=product,
                    quantity=item_data['quantity'],
                    price=product.price
                )
            
            # Send confirmation
            send_order_confirmation_email.delay(order.id)
            
            return Response(
                OrderSerializer(order).data,
                status=status.HTTP_201_CREATED
            )
        
        except Product.DoesNotExist:
            raise ValidationError({'product_id': 'Product not found'})
        
        except Exception as e:
            # Log error
            logger.error(f"Order creation failed: {str(e)}")
            raise APIException('Order creation failed. Please try again.')
```

### 10.5 Error Logging

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.FileHandler',
            'filename': 'logs/api_errors.log',
            'formatter': 'verbose',
        },
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file', 'console'],
            'level': 'ERROR',
            'propagate': True,
        },
        'myapp': {
            'handlers': ['file', 'console'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

# In views
import logging

logger = logging.getLogger('myapp')

class ProductViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        try:
            product = serializer.save()
            logger.info(f"Product created: {product.id}")
            return product
        except Exception as e:
            logger.error(f"Product creation failed: {str(e)}", exc_info=True)
            raise
```

---

## 11. API Versioning

### 11.1 URL Path Versioning (Recommended)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
}

# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import ProductViewSetV1, ProductViewSetV2

router_v1 = DefaultRouter()
router_v1.register(r'products', ProductViewSetV1, basename='product')

router_v2 = DefaultRouter()
router_v2.register(r'products', ProductViewSetV2, basename='product')

urlpatterns = [
    path('api/v1/', include(router_v1.urls)),
    path('api/v2/', include(router_v2.urls)),
]

# Usage:
# GET /api/v1/products/
# GET /api/v2/products/
```

### 11.2 Different Serializers per Version

```python
# serializers.py
class ProductSerializerV1(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ['id', 'name', 'price']

class ProductSerializerV2(serializers.ModelSerializer):
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Product
        fields = ['id', 'name', 'description', 'price', 'category_name', 'stock']

# views.py
class ProductViewSetV1(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializerV1

class ProductViewSetV2(viewsets.ModelViewSet):
    queryset = Product.objects.select_related('category').all()
    serializer_class = ProductSerializerV2
```

### 11.3 Version-Based Logic in Same ViewSet

```python
class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    
    def get_serializer_class(self):
        if self.request.version == 'v1':
            return ProductSerializerV1
        return ProductSerializerV2
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        if self.request.version == 'v2':
            queryset = queryset.select_related('category')
        
        return queryset
```

### 11.4 Accept Header Versioning

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
}

# Usage:
# Headers: Accept: application/json; version=1.0
# Headers: Accept: application/json; version=2.0
```

---

## 12. Testing APIs

### 12.1 Test Setup

```python
# tests/test_products.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from django.contrib.auth import get_user_model
from myapp.models import Product, Category

User = get_user_model()

class ProductAPITestCase(APITestCase):
    def setUp(self):
        """Run before each test"""
        # Create test user
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        
        # Create test category
        self.category = Category.objects.create(name='Electronics')
        
        # Create test products
        self.product1 = Product.objects.create(
            name='iPhone',
            category=self.category,
            price=999.99,
            stock=10
        )
        self.product2 = Product.objects.create(
            name='Samsung',
            category=self.category,
            price=899.99,
            stock=5
        )
        
        # Setup API client
        self.client = APIClient()
```

### 12.2 Testing CRUD Operations

```python
class ProductAPITestCase(APITestCase):
    def setUp(self):
        # ... setup code ...
        pass
    
    def test_list_products(self):
        """Test GET /api/products/"""
        response = self.client.get('/api/products/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 2)
        self.assertEqual(response.data['count'], 2)
    
    def test_retrieve_product(self):
        """Test GET /api/products/{id}/"""
        response = self.client.get(f'/api/products/{self.product1.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['name'], 'iPhone')
        self.assertEqual(float(response.data['price']), 999.99)
    
    def test_create_product(self):
        """Test POST /api/products/"""
        # Login first
        self.client.force_authenticate(user=self.user)
        
        data = {
            'name': 'MacBook',
            'category': self.category.id,
            'price': 1999.99,
            'stock': 3
        }
        
        response = self.client.post('/api/products/', data)
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Product.objects.count(), 3)
        self.assertEqual(response.data['name'], 'MacBook')
    
    def test_update_product(self):
        """Test PUT /api/products/{id}/"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'name': 'iPhone Pro',
            'category': self.category.id,
            'price': 1099.99,
            'stock': 15
        }
        
        response = self.client.put(f'/api/products/{self.product1.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.product1.refresh_from_db()
        self.assertEqual(self.product1.name, 'iPhone Pro')
        self.assertEqual(float(self.product1.price), 1099.99)
    
    def test_partial_update_product(self):
        """Test PATCH /api/products/{id}/"""
        self.client.force_authenticate(user=self.user)
        
        data = {'stock': 20}
        
        response = self.client.patch(f'/api/products/{self.product1.id}/', data)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.product1.refresh_from_db()
        self.assertEqual(self.product1.stock, 20)
        self.assertEqual(self.product1.name, 'iPhone')  # Unchanged
    
    def test_delete_product(self):
        """Test DELETE /api/products/{id}/"""
        self.client.force_authenticate(user=self.user)
        
        response = self.client.delete(f'/api/products/{self.product1.id}/')
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertEqual(Product.objects.count(), 1)
```

### 12.3 Testing Authentication & Permissions

```python
class AuthenticationTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.admin = User.objects.create_user(
            username='admin',
            password='adminpass123',
            is_staff=True
        )
    
    def test_authentication_required(self):
        """Test that authentication is required"""
        response = self.client.post('/api/products/', {})
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_authenticated_user_can_create(self):
        """Test authenticated user can create"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'name': 'Test Product',
            'price': 99.99,
            'stock': 10
        }
        
        response = self.client.post('/api/products/', data)
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
    
    def test_admin_can_delete(self):
        """Test only admin can delete"""
        product = Product.objects.create(name='Test', price=99.99)
        
        # Regular user cannot delete
        self.client.force_authenticate(user=self.user)
        response = self.client.delete(f'/api/products/{product.id}/')
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
        
        # Admin can delete
        self.client.force_authenticate(user=self.admin)
        response = self.client.delete(f'/api/products/{product.id}/')
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
```

### 12.4 Testing Filters & Search

```python
class FilteringTestCase(APITestCase):
    def setUp(self):
        self.category1 = Category.objects.create(name='Electronics')
        self.category2 = Category.objects.create(name='Books')
        
        Product.objects.create(
            name='iPhone',
            category=self.category1,
            price=999.99
        )
        Product.objects.create(
            name='Samsung',
            category=self.category1,
            price=899.99
        )
        Product.objects.create(
            name='Python Book',
            category=self.category2,
            price=49.99
        )
    
    def test_filter_by_category(self):
        """Test filtering by category"""
        response = self.client.get(f'/api/products/?category={self.category1.id}')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 2)
    
    def test_filter_by_price_range(self):
        """Test filtering by price range"""
        response = self.client.get('/api/products/?min_price=100&max_price=1000')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 2)
    
    def test_search(self):
        """Test search functionality"""
        response = self.client.get('/api/products/?search=iphone')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data['results']), 1)
        self.assertEqual(response.data['results'][0]['name'], 'iPhone')
    
    def test_ordering(self):
        """Test ordering results"""
        response = self.client.get('/api/products/?ordering=-price')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        prices = [float(item['price']) for item in response.data['results']]
        self.assertEqual(prices, sorted(prices, reverse=True))
```

### 12.5 Testing Custom Actions

```python
class CustomActionsTestCase(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.product = Product.objects.create(
            name='iPhone',
            price=999.99,
            is_featured=False
        )
    
    def test_featured_products(self):
        """Test GET /api/products/featured/"""
        # Create featured products
        Product.objects.create(name='Featured 1', price=99.99, is_featured=True)
        Product.objects.create(name='Featured 2', price=199.99, is_featured=True)
        
        response = self.client.get('/api/products/featured/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(len(response.data), 2)
    
    def test_publish_product(self):
        """Test POST /api/products/{id}/publish/"""
        self.client.force_authenticate(user=self.user)
        
        response = self.client.post(f'/api/products/{self.product.id}/publish/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.product.refresh_from_db()
        self.assertTrue(self.product.is_published)
```

### 12.6 Running Tests

```bash
# Run all tests
python manage.py test

# Run specific test file
python manage.py test myapp.tests.test_products

# Run specific test class
python manage.py test myapp.tests.test_products.ProductAPITestCase

# Run specific test method
python manage.py test myapp.tests.test_products.ProductAPITestCase.test_list_products

# Run with verbose output
python manage.py test --verbosity=2

# Run with coverage
pip install coverage
coverage run --source='.' manage.py test
coverage report
coverage html  # Generate HTML report
```

---

## 13. Production Best Practices

### 13.1 Security Settings

```python
# settings.py (Production)
DEBUG = False
ALLOWED_HOSTS = ['api.yoursite.com', 'yoursite.com']

# HTTPS
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# CORS
CORS_ALLOWED_ORIGINS = [
    "https://yourfrontend.com",
]
CORS_ALLOW_CREDENTIALS = True

# Security headers
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # ... other middleware ...
]

# Rate limiting
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },
}
```

### 13.2 Database Connection Pooling

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'localhost',
        'PORT': '5432',
        'CONN_MAX_AGE': 600,  # Connection pooling
        'OPTIONS': {
            'connect_timeout': 10,
        }
    }
}
```

### 13.3 Gunicorn Configuration

```bash
# Install
pip install gunicorn

# gunicorn.conf.py
bind = '0.0.0.0:8000'
workers = 4  # 2-4 x number of CPU cores
worker_class = 'sync'
worker_connections = 1000
timeout = 30
keepalive = 2

# Max requests before restart (prevent memory leaks)
max_requests = 1000
max_requests_jitter = 50

# Logging
accesslog = '/var/log/gunicorn/access.log'
errorlog = '/var/log/gunicorn/error.log'
loglevel = 'info'

# Run
gunicorn myproject.wsgi:application -c gunicorn.conf.py
```

### 13.4 Nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp
upstream django {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name api.yoursite.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.yoursite.com;
    
    ssl_certificate /etc/ssl/certs/yoursite.crt;
    ssl_certificate_key /etc/ssl/private/yoursite.key;
    
    client_max_body_size 10M;
    
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    location /static/ {
        alias /var/www/myapp/static/;
        expires 30d;
    }
    
    location /media/ {
        alias /var/www/myapp/media/;
        expires 30d;
    }
}
```

### 13.5 Monitoring & Logging

```python
# Install Sentry
pip install sentry-sdk

# settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn="https://your-sentry-dsn",
    integrations=[DjangoIntegration()],
    traces_sample_rate=1.0,
    send_default_pii=True
)

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/myapp/django.log',
            'maxBytes': 1024 * 1024 * 10,  # 10MB
            'backupCount': 10,
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

### 13.6 Health Check Endpoint

```python
# views.py
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import AllowAny
from rest_framework.response import Response
from django.db import connection

@api_view(['GET'])
@permission_classes([AllowAny])
def health_check(request):
    """Health check endpoint for load balancers"""
    try:
        # Check database
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        
        # Check cache
        from django.core.cache import cache
        cache.set('health_check', 'ok', 1)
        assert cache.get('health_check') == 'ok'
        
        return Response({
            'status': 'healthy',
            'database': 'ok',
            'cache': 'ok'
        })
    except Exception as e:
        return Response({
            'status': 'unhealthy',
            'error': str(e)
        }, status=503)

# urls.py
urlpatterns = [
    path('health/', health_check),
]
```

---

## 14. Common Errors & Solutions

### 14.1 "Detail: Not found."

**Problem:**
```python
response = self.client.get(f'/api/products/999/')
# Response: {"detail": "Not found."}
```

**Solution:**
```python
# Check if object exists before accessing
try:
    product = Product.objects.get(id=999)
except Product.DoesNotExist:
    return Response(
        {'error': 'Product not found'},
        status=status.HTTP_404_NOT_FOUND
    )

# Or use get_object_or_404
from django.shortcuts.get_object_or_404
product = get_object_or_404(Product, id=999)
```

### 14.2 "Authentication credentials were not provided."

**Problem:**
```python
# Trying to access protected endpoint without auth
response = self.client.post('/api/products/', data)
# Response: {"detail": "Authentication credentials were not provided."}
```

**Solution:**
```python
# Include JWT token in headers
headers = {'Authorization': f'Bearer {access_token}'}
response = self.client.post('/api/products/', data, headers=headers)

# Or force authenticate in tests
self.client.force_authenticate(user=self.user)
```

### 14.3 "Invalid page."

**Problem:**
```python
# Requesting page that doesn't exist
response = self.client.get('/api/products/?page=9999')
# Response: {"detail": "Invalid page."}
```

**Solution:**
```python
# Handle in custom pagination
class SafePageNumberPagination(PageNumberPagination):
    def get_paginated_response(self, data):
        try:
            return super().get_paginated_response(data)
        except EmptyPage:
            return Response({
                'count': self.page.paginator.count,
                'next': None,
                'previous': None,
                'results': []
            })
```

### 14.4 N+1 Query Problem

**Problem:**
```python
# This causes N+1 queries!
products = Product.objects.all()
for product in products:
    print(product.category.name)  # New query for each product
```

**Solution:**
```python
# Use select_related for ForeignKey
products = Product.objects.select_related('category').all()

# Use prefetch_related for reverse relations
categories = Category.objects.prefetch_related('products').all()
```

### 14.5 "Method Not Allowed"

**Problem:**
```python
# Trying to use wrong HTTP method
response = self.client.post('/api/products/1/')  # Should be PUT/PATCH
# Response: {"detail": "Method \"POST\" not allowed."}
```

**Solution:**
```python
# Use correct method
response = self.client.put(f'/api/products/{product.id}/', data)
# Or
response = self.client.patch(f'/api/products/{product.id}/', data)
```

### 14.6 Serializer Validation Errors

**Problem:**
```python
serializer = ProductSerializer(data={'name': ''})  # Empty name
serializer.is_valid()
# False
print(serializer.errors)
# {'name': ['This field may not be blank.']}
```

**Solution:**
```python
# Always check is_valid() and handle errors
if serializer.is_valid():
    serializer.save()
else:
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

# Or use raise_exception=True
serializer.is_valid(raise_exception=True)
```

### 14.7 CORS Errors

**Problem:**
```
Access to fetch at 'http://api.example.com/products/' from origin 'http://localhost:3000' 
has been blocked by CORS policy
```

**Solution:**
```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    # ...
]

# Development
CORS_ALLOW_ALL_ORIGINS = True

# Production
CORS_ALLOWED_ORIGINS = [
    "https://yourfrontend.com",
]
```

---

## 15. Real-World Patterns (GTAF)

### 15.1 Quran API with Translations

```python
# models.py
class Surah(models.Model):
    number = models.IntegerField(unique=True)
    name_arabic = models.CharField(max_length=100)
    name_english = models.CharField(max_length=100)
    verses_count = models.IntegerField()

class Verse(models.Model):
    surah = models.ForeignKey(Surah, on_delete=models.CASCADE, related_name='verses')
    verse_number = models.IntegerField()
    text_arabic = models.TextField()
    
    class Meta:
        unique_together = ('surah', 'verse_number')
        indexes = [
            models.Index(fields=['surah', 'verse_number']),
        ]

class Translation(models.Model):
    verse = models.ForeignKey(Verse, on_delete=models.CASCADE, related_name='translations')
    language = models.CharField(max_length=10)
    translator = models.CharField(max_length=100)
    text = models.TextField()
    
    class Meta:
        unique_together = ('verse', 'language', 'translator')

# serializers.py
class TranslationSerializer(serializers.ModelSerializer):
    class Meta:
        model = Translation
        fields = ['language', 'translator', 'text']

class VerseDetailSerializer(serializers.ModelSerializer):
    translations = TranslationSerializer(many=True, read_only=True)
    surah_name = serializers.CharField(source='surah.name_arabic', read_only=True)
    
    class Meta:
        model = Verse
        fields = ['verse_number', 'text_arabic', 'surah_name', 'translations']

# views.py
class VerseViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Verse.objects.select_related('surah').prefetch_related(
        Prefetch(
            'translations',
            queryset=Translation.objects.all()
        )
    )
    serializer_class = VerseDetailSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter]
    filterset_fields = ['surah__number', 'verse_number']
    search_fields = ['text_arabic', 'translations__text']
    
    @action(detail=False, methods=['get'])
    def by_surah(self, request):
        """GET /api/verses/by_surah/?surah=2&language=en"""
        surah_number = request.query_params.get('surah')
        language = request.query_params.get('language', 'en')
        
        if not surah_number:
            return Response(
                {'error': 'surah parameter is required'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        verses = self.queryset.filter(
            surah__number=surah_number
        ).prefetch_related(
            Prefetch(
                'translations',
                queryset=Translation.objects.filter(language=language)
            )
        )
        
        serializer = self.get_serializer(verses, many=True)
        return Response(serializer.data)

# Usage:
# GET /api/verses/?surah__number=2&verse_number=255
# GET /api/verses/by_surah/?surah=1&language=en
# GET /api/verses/?search=paradise
```

### 15.2 Prayer Times API with Caching

```python
# models.py
class City(models.Model):
    name = models.CharField(max_length=100)
    country = models.CharField(max_length=100)
    latitude = models.DecimalField(max_digits=9, decimal_places=6)
    longitude = models.DecimalField(max_digits=9, decimal_places=6)

class PrayerTime(models.Model):
    city = models.ForeignKey(City, on_delete=models.CASCADE)
    date = models.DateField()
    fajr = models.TimeField()
    dhuhr = models.TimeField()
    asr = models.TimeField()
    maghrib = models.TimeField()
    isha = models.TimeField()
    
    class Meta:
        unique_together = ('city', 'date')
        indexes = [
            models.Index(fields=['city', 'date']),
        ]

# views.py
from django.core.cache import cache

class PrayerTimeViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = PrayerTime.objects.select_related('city')
    serializer_class = PrayerTimeSerializer
    
    @action(detail=False, methods=['get'])
    def today(self, request):
        """GET /api/prayer-times/today/?city=1"""
        city_id = request.query_params.get('city')
        
        if not city_id:
            return Response(
                {'error': 'city parameter is required'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Try cache first
        cache_key = f"prayer_times_{city_id}_{date.today()}"
        cached_times = cache.get(cache_key)
        
        if cached_times:
            return Response(cached_times)
        
        # Get from database
        prayer_time = PrayerTime.objects.filter(
            city_id=city_id,
            date=date.today()
        ).select_related('city').first()
        
        if not prayer_time:
            return Response(
                {'error': 'Prayer times not found for this city'},
                status=status.HTTP_404_NOT_FOUND
            )
        
        serializer = self.get_serializer(prayer_time)
        
        # Cache for 1 hour
        cache.set(cache_key, serializer.data, 60 * 60)
        
        return Response(serializer.data)
```

### 15.3 Donation API with Celery

```python
# models.py
class Donation(models.Model):
    donor_name = models.CharField(max_length=100)
    donor_email = models.EmailField()
    donor_phone = models.CharField(max_length=20)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_method = models.CharField(max_length=20)  # bkash, nagad, card
    payment_status = models.CharField(max_length=20, default='pending')
    payment_id = models.CharField(max_length=100, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

# tasks.py
@shared_task
def send_donation_receipt_email(donation_id):
    donation = Donation.objects.get(id=donation_id)
    
    send_mail(
        subject='Donation Receipt',
        message=f'Thank you for your donation of ${donation.amount}',
        from_email=settings.DEFAULT_FROM_EMAIL,
        recipient_list=[donation.donor_email],
    )

@shared_task
def send_donation_sms(donation_id):
    donation = Donation.objects.get(id=donation_id)
    
    if donation.donor_phone:
        send_sms(
            phone_number=donation.donor_phone,
            message=f'Your donation of ${donation.amount} has been received. Thank you!'
        )

# views.py
class DonationViewSet(viewsets.ModelViewSet):
    queryset = Donation.objects.all()
    serializer_class = DonationSerializer
    
    @action(detail=True, methods=['post'])
    def confirm_payment(self, request, pk=None):
        """POST /api/donations/{id}/confirm_payment/"""
        donation = self.get_object()
        
        if donation.payment_status != 'pending':
            return Response(
                {'error': 'Donation already processed'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        # Update status
        donation.payment_status = 'completed'
        donation.save()
        
        # Send email and SMS asynchronously
        send_donation_receipt_email.delay(donation.id)
        send_donation_sms.delay(donation.id)
        
        return Response({
            'message': 'Payment confirmed. Receipt will be sent shortly.'
        })
```

---

## Summary: From Beginner to Advanced

### 🎯 **Learning Path**

**Week 1-2: Basics**
- ✅ DRF setup & configuration
- ✅ Serializers (ModelSerializer)
- ✅ APIView & Generic Views
- ✅ Basic CRUD operations

**Week 3-4: Intermediate**
- ✅ ViewSets & Routers
- ✅ Authentication (JWT)
- ✅ Permissions & Throttling
- ✅ Filtering & Pagination

**Week 5-6: Advanced**
- ✅ N+1 problem solutions
- ✅ Performance optimization
- ✅ Redis caching
- ✅ Celery background tasks

**Week 7-8: Production**
- ✅ API versioning
- ✅ Testing
- ✅ Error handling
- ✅ Deployment (Gunicorn + Nginx)

### 🚀 **Enterprise-Level Checklist**

✅ **Performance**
- Use `select_related()` & `prefetch_related()`
- Implement caching (Redis)
- Database indexing
- Connection pooling

✅ **Security**
- JWT authentication
- Permission classes
- Rate limiting
- HTTPS only

✅ **Scalability**
- Background tasks (Celery)
- Async operations
- Database optimization
- Load balancing

✅ **Reliability**
- Comprehensive testing
- Error handling
- Logging & monitoring
- Health check endpoints

✅ **Best Practices**
- API versioning
- Documentation (Swagger)
- Clean code structure
- Type hints

---

**Created for Maharab - Backend Engineer at Greentech Apps Foundation**
*Master Django REST Framework from Basics to Enterprise Level*

🎉 **You're now ready to build world-class APIs!**
