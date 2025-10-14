# understanding_django





# Django MVC/MVT Architecture & Multi-App Structure Guide

## Table of Contents
1. [MVC vs MVT Pattern](#mvc-vs-mvt-pattern)
2. [Views as Controllers in Django](#views-as-controllers-in-django)
3. [Complete Request Flow](#complete-request-flow)
4. [Multi-App Architecture](#multi-app-architecture)
5. [API Versioning Strategy](#api-versioning-strategy)
6. [Decision Heuristics](#decision-heuristics)

---

## 🏗️ MVC vs MVT Pattern

### Traditional MVC (Node.js/Express)

```
┌─────────────────────────────────────────┐
│           MVC PATTERN (Node.js)         │
├─────────────────────────────────────────┤
│                                         │
│  Model ──────► Controller ──────► View │
│  (Data)       (Logic)          (UI)    │
│                                         │
│  Prisma       Express          React   │
│  Schema       Routes           JSX     │
│               Services                  │
└─────────────────────────────────────────┘
```

### Django MVT Pattern

```
┌──────────────────────────────────────────┐
│        MVT PATTERN (Django)              │
├──────────────────────────────────────────┤
│                                          │
│  Model ──────► View ──────► Template    │
│  (Data)       (Logic)       (UI)        │
│                                          │
│  Django       Django        HTML/Django │
│  ORM          Views         Templates   │
│               (Serializers)             │
└──────────────────────────────────────────┘
```

### Django REST Framework (API-Only)

```
┌──────────────────────────────────────────┐
│    Django REST Framework (API)           │
├──────────────────────────────────────────┤
│                                          │
│  Model ──► Serializer ──► View ──► JSON │
│  (Data)    (DTO)         (Logic)  (API) │
│                                          │
│  ORM       Validate      ViewSet   DRF  │
│            Transform     Router    Auto │
└──────────────────────────────────────────┘
```

---

## 📊 Component Comparison

| Component | Node.js/Express | Django REST Framework |
|-----------|-----------------|----------------------|
| **Model** | Prisma Schema | Django Model |
| **Controller** | Route Handler + Service | **View/ViewSet** ← This is the controller! |
| **DTO** | Custom DTO classes | **Serializer** |
| **View** | React Component | (Handled by React separately) |
| **Router** | Express Router | DRF Router + urls.py |

---

## 🎯 Views = Controllers in Django

In Django REST Framework, **Views ARE the Controllers!**

### Node.js/Express Controller Pattern

```javascript
// ============================================
//           NODE.JS / EXPRESS
// ============================================

// 1. MODEL (Prisma Schema)
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  isActive  Boolean  @default(true)
  posts     Post[]   // One-to-many
}

// 2. DTO (Data Transfer Object)
class UserDTO {
  id: number;
  name: string;
  email: string;
  isActive: boolean;
}

// 3. DAL (Data Access Layer)
class UserRepository {
  async findAll() {
    return await prisma.user.findMany();
  }
  
  async findById(id: number) {
    return await prisma.user.findUnique({ where: { id } });
  }
  
  async create(data: CreateUserDTO) {
    return await prisma.user.create({ data });
  }
}

// 4. SERVICE (Business Logic)
class UserService {
  constructor(private repo: UserRepository) {}
  
  async getUsers() {
    const users = await this.repo.findAll();
    return users.map(u => new UserDTO(u));
  }
  
  async getUser(id: number) {
    const user = await this.repo.findById(id);
    if (!user) throw new Error('Not found');
    return new UserDTO(user);
  }
}

// 5. CONTROLLER (HTTP Layer)
class UserController {
  constructor(private service: UserService) {}
  
  async listUsers(req: Request, res: Response) {
    try {
      const users = await this.service.getUsers();
      res.json(users);
    } catch (error) {
      res.status(500).json({ error: error.message });
    }
  }
  
  async getUser(req: Request, res: Response) {
    try {
      const id = parseInt(req.params.id);
      const user = await this.service.getUser(id);
      res.json(user);
    } catch (error) {
      res.status(404).json({ error: 'Not found' });
    }
  }
  
  async createUser(req: Request, res: Response) {
    try {
      const user = await this.service.createUser(req.body);
      res.status(201).json(user);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}

// 6. ROUTER
const userController = new UserController(userService);
router.get('/users', userController.listUsers);
router.get('/users/:id', userController.getUser);
router.post('/users', userController.createUser);
```

### Django REST Framework ViewSet Pattern

```python
# ============================================
#       DJANGO REST FRAMEWORK
# ============================================

# 1. MODEL (Django ORM) - Like Prisma Schema
class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    # posts relationship via ForeignKey in Post model
    
    class Meta:
        db_table = 'users'
        ordering = ['-created_at']

# 2. SERIALIZER (DTO + Validation) - Like DTO
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'name', 'email', 'is_active', 'created_at']
        read_only_fields = ['id', 'created_at']

# 3. VIEW (Controller + Service + DAL combined!)
class UserViewSet(viewsets.ModelViewSet):
    """
    This ONE class replaces:
    - Repository (DAL)
    - Service (Business Logic)  
    - Controller (HTTP Handlers)
    - Router (URL patterns)
    
    All CRUD operations automatically generated!
    """
    queryset = User.objects.all()  # ← DAL: Database query
    serializer_class = UserSerializer  # ← DTO: Data transformation
    
    # DRF automatically provides these methods:
    # - list(request)      → GET /users/
    # - retrieve(request, pk) → GET /users/{id}/
    # - create(request)    → POST /users/
    # - update(request, pk) → PUT /users/{id}/
    # - partial_update(request, pk) → PATCH /users/{id}/
    # - destroy(request, pk) → DELETE /users/{id}/
    
    # Custom action (like custom controller method)
    @action(detail=True, methods=['get'])
    def posts(self, request, pk=None):
        """GET /api/users/{id}/posts/"""
        user = self.get_object()  # ← Get user by ID
        posts = user.posts.all()   # ← Query related posts
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

# 4. ROUTER (Auto-generates URL patterns)
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet)
# Auto-creates all CRUD routes!
```

---

## 🔄 Complete Request Flow

### Scenario: GET all users

```
CLIENT REQUEST:
GET http://127.0.0.1:8000/api/users/
```

#### 1️⃣ URL Routing

```python
# blogproject/urls.py (Main Router)
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),  # Routes to api app
]

# api/urls.py (DRF Router)
router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')
urlpatterns = [
    path('', include(router.urls)),
]
```

#### 2️⃣ View/Controller

```python
# api/views.py
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()  # ← DATABASE QUERY (DAL)
    serializer_class = UserSerializer  # ← DATA TRANSFORMATION (DTO)
    
    # DRF calls this method automatically:
    def list(self, request):  # ← CONTROLLER METHOD
        queryset = self.get_queryset()  # Get all users from DB
        serializer = self.get_serializer(queryset, many=True)  # Serialize
        return Response(serializer.data)  # Return JSON response
```

#### 3️⃣ Model (Database Query)

```python
# User.objects.all() translates to:
SELECT * FROM users ORDER BY created_at DESC;
# Returns: [User(id=1, name='John'...), User(id=2, name='Jane'...)]
```

#### 4️⃣ Serializer (Data Transformation)

```python
# api/serializers.py
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'name', 'email', 'is_active', 'created_at']

# Transforms Python objects to JSON:
# User(id=1, name='John', email='john@...')
# ↓
# {"id": 1, "name": "John", "email": "john@..."}
```

#### 5️⃣ JSON Response

```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com",
      "is_active": true,
      "created_at": "2025-10-15T10:30:00Z"
    },
    {
      "id": 2,
      "name": "Jane Smith",
      "email": "jane@example.com",
      "is_active": true,
      "created_at": "2025-10-14T15:20:00Z"
    }
  ]
}
```

---

## 🎨 Visual Flow Diagram

```
┌────────────────────────────────────────────────────────────┐
│                  CLIENT (React/Postman)                    │
│              GET http://127.0.0.1:8000/api/users/         │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  1. URL ROUTING (blogproject/urls.py)                     │
│     path('api/', include('api.urls'))                     │
│     → Routes to api app                                    │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  2. DRF ROUTER (api/urls.py)                              │
│     router.register('users', UserViewSet)                 │
│     → Maps to UserViewSet.list() method                   │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  3. VIEW/CONTROLLER (api/views.py)                        │
│     class UserViewSet(viewsets.ModelViewSet):             │
│         queryset = User.objects.all()  ← DAL              │
│         serializer_class = UserSerializer  ← DTO          │
│                                                            │
│     Automatically provides:                                │
│     - list()     → GET /users/                            │
│     - retrieve() → GET /users/{id}/                       │
│     - create()   → POST /users/                           │
│     - update()   → PUT /users/{id}/                       │
│     - destroy()  → DELETE /users/{id}/                    │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  4. MODEL/ORM (api/models.py)                             │
│     User.objects.all()                                     │
│     → SELECT * FROM users ORDER BY created_at DESC;       │
│     → Returns QuerySet of User objects                     │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  5. SERIALIZER/DTO (api/serializers.py)                   │
│     UserSerializer(users, many=True)                       │
│     → Converts Python objects to JSON                      │
│     → Validates data                                       │
│     → Applies field rules (read_only, required, etc.)     │
└──────────────────────────┬─────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│  6. JSON RESPONSE                                          │
│     {                                                      │
│       "count": 2,                                         │
│       "results": [                                        │
│         {"id": 1, "name": "John", ...},                  │
│         {"id": 2, "name": "Jane", ...}                   │
│       ]                                                    │
│     }                                                      │
└────────────────────────────────────────────────────────────┘
```

---

## 🔥 Why ViewSets are Powerful

### Node.js: Lots of Boilerplate (≈50 lines)

```javascript
// You have to write ALL of this manually:

router.get('/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

router.get('/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({ where: { id: req.params.id } });
  if (!user) return res.status(404).json({ error: 'Not found' });
  res.json(user);
});

router.post('/users', async (req, res) => {
  const user = await prisma.user.create({ data: req.body });
  res.status(201).json(user);
});

router.put('/users/:id', async (req, res) => {
  const user = await prisma.user.update({
    where: { id: req.params.id },
    data: req.body
  });
  res.json(user);
});

router.delete('/users/:id', async (req, res) => {
  await prisma.user.delete({ where: { id: req.params.id } });
  res.status(204).send();
});
```

### Django REST Framework: Automatic (3 lines)

```python
# All of the above in just 3 lines:

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

# Done! 🎉
# - GET /users/
# - GET /users/{id}/
# - POST /users/
# - PUT /users/{id}/
# - PATCH /users/{id}/
# - DELETE /users/{id}/
# All automatically generated!
```

---

## 🎯 Summary: Views = Controllers

| Concept | Node.js/Express | Django REST Framework |
|---------|-----------------|----------------------|
| **Controller** | Route handlers in `controller.ts` | **View/ViewSet** in `views.py` |
| **Service** | Business logic in `service.ts` | Built into ViewSet |
| **Repository/DAL** | Database queries in `repository.ts` | `queryset` in ViewSet |
| **DTO** | Custom DTO classes | **Serializer** |
| **Router** | Express Router | DRF Router + `urls.py` |
| **Validation** | Manual (Zod, Joi, etc.) | Built into Serializer |

**In Django: View = Controller + Service + Repository combined!** 🚀

---

# 🏗️ Multi-Vendor Home Services Platform Architecture

## Real-World Example: Home Services Booking App
(Like Upwork, but specifically for home services - electricians, plumbers, cleaners, etc.)

---

## 🎯 When to Create Separate Django Apps

### Decision Tree

```
Should I create a new app?
│
├─ Does it represent a distinct business domain? → YES = New App
├─ Does it have its own models/data? → YES = New App  
├─ Can it be reused in other projects? → YES = New App
├─ Will it grow complex over time? → YES = New App
└─ Is it just 1-2 models with simple logic? → NO = Add to existing app
```

---

## 📦 Recommended App Structure for Home Services Platform

```
home_services_project/          ← Main project folder
│
├── manage.py
├── requirements.txt
├── README.md
│
├── config/                     ← Project configuration (settings, urls)
│   ├── __init__.py
│   ├── settings/
│   │   ├── base.py            ← Common settings
│   │   ├── development.py     ← Dev settings
│   │   ├── production.py      ← Prod settings
│   │   └── staging.py
│   ├── urls.py                ← Main URL routing with versioning
│   ├── wsgi.py
│   └── asgi.py
│
├── apps/                       ← All Django apps here
│   │
│   ├── users/                  ← User management app
│   │   ├── models.py          (User, Profile, Address)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── permissions.py
│   │   └── managers.py
│   │
│   ├── providers/              ← Service provider app
│   │   ├── models.py          (Provider, ProviderProfile, Availability)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── signals.py
│   │
│   ├── services/               ← Services catalog app
│   │   ├── models.py          (Category, Service, ServicePricing)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── filters.py
│   │
│   ├── bookings/               ← Booking/appointment app
│   │   ├── models.py          (Booking, BookingStatus, TimeSlot)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── tasks.py           (Celery tasks)
│   │   └── state_machine.py
│   │
│   ├── reviews/                ← Reviews and ratings app
│   │   ├── models.py          (Review, Rating)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── signals.py
│   │
│   ├── payments/               ← Payment processing app
│   │   ├── models.py          (Payment, Transaction, Wallet)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── integrations/      (Stripe, PayPal)
│   │   └── webhooks.py
│   │
│   ├── notifications/          ← Notification app
│   │   ├── models.py          (Notification, PushToken)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── tasks.py
│   │   └── channels/          (WebSocket)
│   │
│   ├── chat/                   ← Messaging app
│   │   ├── models.py          (Conversation, Message)
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── consumers.py
│   │
│   └── core/                   ← Shared utilities app
│       ├── models.py          (Abstract base models)
│       ├── permissions.py
│       ├── pagination.py
│       ├── exceptions.py
│       └── utils.py
│
├── static/                     ← Static files
├── media/                      ← User uploaded files
└── logs/                       ← Application logs
```

---

## 🎯 Why This Many Apps? (Decision Breakdown)

| App | Why Separate? | Models | Reusable? |
|-----|---------------|--------|-----------|
| **users** | Core authentication, used everywhere | User, Profile, Address | ✅ Yes (any project) |
| **providers** | Distinct business domain, complex logic | Provider, Availability, Documents | ✅ Yes (gig platforms) |
| **services** | Service catalog management | Category, Service, Pricing | ✅ Yes (marketplaces) |
| **bookings** | Core business logic, complex state machine | Booking, TimeSlot, Status | ✅ Yes (appointment apps) |
| **reviews** | Independent feature, can be reused | Review, Rating | ✅ Yes (any marketplace) |
| **payments** | Complex integrations, security critical | Payment, Transaction, Wallet | ✅ Yes (e-commerce) |
| **notifications** | Independent service, multiple channels | Notification, PushToken | ✅ Yes (any app) |
| **chat** | Real-time feature, WebSocket | Conversation, Message | ✅ Yes (social apps) |
| **core** | Shared utilities across apps | Abstract models, helpers | ✅ Yes (framework) |

---

## 🔢 API Versioning Structure (`/api/v1/`)

### Main `config/urls.py` with Versioning

```python
# config/urls.py (Main project URLs with versioning)

from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # Admin panel
    path('admin/', admin.site.urls),
    
    # API Version 1 (Current stable version)
    path('api/v1/', include([
        path('users/', include('apps.users.urls')),
        path('providers/', include('apps.providers.urls')),
        path('services/', include('apps.services.urls')),
        path('bookings/', include('apps.bookings.urls')),
        path('reviews/', include('apps.reviews.urls')),
        path('payments/', include('apps.payments.urls')),
        path('notifications/', include('apps.notifications.urls')),
        path('chat/', include('apps.chat.urls')),
    ])),
    
    # API Version 2 (Beta - new features)
    # path('api/v2/', include([
    #     path('users/', include('apps.users.urls_v2')),
    #     path('bookings/', include('apps.bookings.urls_v2')),
    #     # Only changed endpoints in v2
    # ])),
    
    # Health check endpoint (no versioning)
    path('health/', include('apps.core.urls')),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### Resulting URL Structure

```
USERS:
POST   /api/v1/users/register/
POST   /api/v1/users/login/
GET    /api/v1/users/me/
PUT    /api/v1/users/me/
POST   /api/v1/users/change-password/

PROVIDERS:
GET    /api/v1/providers/
GET    /api/v1/providers/{id}/
POST   /api/v1/providers/register/
GET    /api/v1/providers/{id}/services/
GET    /api/v1/providers/{id}/reviews/
GET    /api/v1/providers/{id}/availability/

SERVICES:
GET    /api/v1/services/
GET    /api/v1/services/{id}/
GET    /api/v1/services/categories/
GET    /api/v1/services/search/?category=plumbing&location=NYC

BOOKINGS:
GET    /api/v1/bookings/
POST   /api/v1/bookings/
GET    /api/v1/bookings/{id}/
PATCH  /api/v1/bookings/{id}/cancel/
PATCH  /api/v1/bookings/{id}/accept/
PATCH  /api/v1/bookings/{id}/complete/

REVIEWS:
GET    /api/v1/reviews/
POST   /api/v1/reviews/
GET    /api/v1/reviews/{id}/

PAYMENTS:
POST   /api/v1/payments/create-intent/
POST   /api/v1/payments/confirm/
GET    /api/v1/payments/history/
POST   /api/v1/payments/webhook/

NOTIFICATIONS:
GET    /api/v1/notifications/
PATCH  /api/v1/notifications/{id}/read/

CHAT:
GET    /api/v1/chat/conversations/
POST   /api/v1/chat/conversations/
GET    /api/v1/chat/conversations/{id}/messages/
POST   /api/v1/chat/conversations/{id}/messages/
```

---

## 📱 Complete Booking Flow (How Apps Work Together)

### Models from Different Apps

```python
# -----------------------------
# 1. USERS APP
# -----------------------------
class User(AbstractUser):
    """Extended user model"""
    USER_TYPE_CHOICES = [
        ('customer', 'Customer'),
        ('provider', 'Service Provider'),
    ]
    user_type = models.CharField(max_length=20, choices=USER_TYPE_CHOICES)
    phone = models.CharField(max_length=20, blank=True)
    profile_picture = models.ImageField(upload_to='profiles/', null=True)
    is_verified = models.BooleanField(default=False)


class Address(models.Model):
    """User addresses"""
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='addresses')
    street = models.CharField(max_length=255)
    city = models.CharField(max_length=100)
    state = models.CharField(max_length=100)
    zip_code = models.CharField(max_length=20)
    is_default = models.BooleanField(default=False)


# -----------------------------
# 2. PROVIDERS APP
# -----------------------------
class Provider(models.Model):
    """Service provider profile"""
    user = models.OneToOneField('users.User', on_delete=models.CASCADE)
    business_name = models.CharField(max_length=200)
    description = models.TextField()
    experience_years = models.IntegerField(default=0)
    hourly_rate = models.DecimalField(max_digits=10, decimal_places=2)
    is_available = models.BooleanField(default=True)
    average_rating = models.DecimalField(max_digits=3, decimal_places=2, default=0)
    total_bookings = models.IntegerField(default=0)
    is_verified = models.BooleanField(default=False)


# -----------------------------
# 3. SERVICES APP
# -----------------------------
class ServiceCategory(models.Model):
    """Service categories (Plumbing, Electrical, etc.)"""
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(unique=True)
    icon = models.ImageField(upload_to='categories/', null=True)
    is_active = models.BooleanField(default=True)


class Service(models.Model):
    """Individual services offered"""
    provider = models.ForeignKey('providers.Provider', on_delete=models.CASCADE)
    category = models.ForeignKey(ServiceCategory, on_delete=models.PROTECT)
    name = models.CharField(max_length=200)
    description = models.TextField()
    base_price = models.DecimalField(max_digits=10, decimal_places=2)
    duration_minutes = models.IntegerField()


# -----------------------------
# 4. BOOKINGS APP
# -----------------------------
class Booking(models.Model):
    """Service booking/appointment"""
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('accepted', 'Accepted'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
        ('cancelled', 'Cancelled'),
        ('refunded', 'Refunded'),
    ]
    
    booking_number = models.CharField(max_length=20, unique=True)
    customer = models.ForeignKey('users.User', on_delete=models.PROTECT)
    provider = models.ForeignKey('providers.Provider', on_delete=models.PROTECT)
    service = models.ForeignKey('services.Service', on_delete=models.PROTECT)
    address = models.ForeignKey('users.Address', on_delete=models.PROTECT)
    
    scheduled_date = models.DateField()
    scheduled_time = models.TimeField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    
    service_fee = models.DecimalField(max_digits=10, decimal_places=2)
    platform_fee = models.DecimalField(max_digits=10, decimal_places=2)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)


# -----------------------------
# 5. REVIEWS APP
# -----------------------------
class Review(models.Model):
    """Customer reviews for providers"""
    booking = models.OneToOneField('bookings.Booking', on_delete=models.CASCADE)
    customer = models.ForeignKey('users.User', on_delete=models.CASCADE)
    provider = models.ForeignKey('providers.Provider', on_delete=models.CASCADE)
    
    rating = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    comment = models.TextField()
    professionalism = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    punctuality = models.IntegerField(choices=[(i, i) for i in range(1, 6)])
    quality = models.IntegerField(choices=[(i, i) for i in range(1, 6)])


# -----------------------------
# 6. PAYMENTS APP
# -----------------------------
class Payment(models.Model):
    """Payment transactions"""
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('completed', 'Completed'),
        ('failed', 'Failed'),
        ('refunded', 'Refunded'),
    ]
    
    booking = models.ForeignKey('bookings.Booking', on_delete=models.PROTECT)
    customer = models.ForeignKey('users.User', on_delete=models.PROTECT)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    payment_method = models.CharField(max_length=20)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    transaction_id = models.CharField(max_length=100, unique=True)


# -----------------------------
# 7. NOTIFICATIONS APP
# -----------------------------
class Notification(models.Model):
    """User notifications"""
    user = models.ForeignKey('users.User', on_delete=models.CASCADE)
    notification_type = models.CharField(max_length=50)
    title = models.CharField(max_length=200)
    message = models.TextField()
    is_read = models.BooleanField(default=False)
```

---

## 🔄 Complete Booking Flow

```python
# STEP 1: Customer searches for services
# GET /api/v1/services/?category=plumbing&city=NYC
# → services app returns available services

# STEP 2: Customer views provider profile
# GET /api/v1/providers/123/
# → providers app returns provider details
# → reviews app shows provider ratings

# STEP 3: Customer creates booking
# POST /api/v1/bookings/
# → bookings app creates booking
# → payments app creates pending payment
# → notifications app sends notification to provider

# STEP 4: Provider accepts booking
# PATCH /api/v1/bookings/1/accept/
# → bookings app updates status to 'accepted'
# → notifications app notifies customer

# STEP 5: Customer makes payment
# POST /api/v1/payments/confirm/
# → payments app processes payment via Stripe
# → bookings app updates payment status

# STEP 6: Service completed
# PATCH /api/v1/bookings/1/complete/
# → bookings app updates status to 'completed'
# → payments app releases funds to provider
# → notifications app prompts customer to leave review

# STEP 7: Customer leaves review
# POST /api/v1/reviews/
# → reviews app creates review
# → providers app updates average rating (via signal)
# → notifications app notifies provider
```

---

## 🎯 Decision Heuristics: When to Create New Apps

### ✅ CREATE NEW APP when:

1. **Distinct Business Domain**
   - users (authentication, profiles)
   - bookings (appointment management)
   - payments (financial transactions)

2. **Independent Functionality**
   - notifications (can work standalone)
   - reviews (can be reused in other projects)
   - chat (independent messaging feature)

3. **Will Grow Complex**
   - providers (verification, availability, ratings, etc.)
   - services (categories, pricing, search, filters)

4. **Reusable Across Projects**
   - core (abstract models, utilities)
   - payments (payment gateway integrations)

5. **Different Team Ownership**
   - Team A works on bookings
   - Team B works on payments

---

### ❌ DON'T CREATE NEW APP when:

1. **Too Simple (1-2 models with basic CRUD)**
   - ❌ Don't create separate "posts" app for simple blog
   - ✅ Add to existing "content" app

2. **Tightly Coupled to Another App**
   - ❌ Don't create "booking_status" app
   - ✅ Keep BookingStatus model in bookings app

3. **No Clear Business Boundary**
   - ❌ Don't create "helpers" app
   - ✅ Use core app for utilities

---

## 📊 Comparison: Simple vs Complex Projects

### Simple Blog Project (Current)

```
❌ Overkill to create separate apps:
   - users app (just User model)
   - posts app (just Post model)

✅ Better approach:
blogproject/
    api/                    ← Single app is fine
        models.py          (User + Post models)
        serializers.py
        views.py
```

**Why?** Only 2 models, simple relationships, won't grow complex.

---

### Home Services Platform (Complex)

```
✅ Need multiple apps:
home_services/
    users/          ← Auth + profiles (will grow)
    providers/      ← Provider management (complex)
    services/       ← Service catalog (searchable)
    bookings/       ← Booking workflow (state machine)
    reviews/        ← Rating system (reusable)
    payments/       ← Payment processing (3rd party)
    notifications/  ← Multi-channel notifications
    chat/           ← Real-time messaging
```

**Why?** Each domain is complex, independent, and will grow over time.

---

## 🔢 API Versioning Best Practices

### When to Use `/api/v1/`

```python
# config/urls.py
urlpatterns = [
    path('api/v1/', include([
        path('bookings/', include('apps.bookings.urls')),
        # ... other apps
    ])),
]
```

### Why Version APIs?

1. **Backward Compatibility**
   ```
   /api/v1/bookings/     ← Mobile app v1.0 uses this
   /api/v2/bookings/     ← Mobile app v2.0 uses this (new fields)
   ```

2. **Breaking Changes**
   ```python
   # v1: Booking model
   status = models.CharField(choices=['pending', 'completed'])
   
   # v2: Breaking change (more statuses)
   status = models.CharField(
       choices=['pending', 'accepted', 'in_progress', 'completed']
   )
   ```

3. **Gradual Migration**
   ```
   /api/v1/  ← Old clients keep working
   /api/v2/  ← New clients use new features
   ```

---

### Version Migration Strategy

```python
# apps/bookings/urls.py (v1)
from rest_framework.routers import DefaultRouter
from .views import BookingViewSet

router = DefaultRouter()
router.register('', BookingViewSet, basename='booking')
urlpatterns = router.urls


# apps/bookings/urls_v2.py (v2)
from rest_framework.routers import DefaultRouter
from .views_v2 import BookingViewSetV2  # New version

router = DefaultRouter()
router.register('', BookingViewSetV2, basename='booking-v2')
urlpatterns = router.urls


# config/urls.py
urlpatterns = [
    path('api/v1/bookings/', include('apps.bookings.urls')),      # Old
    path('api/v2/bookings/', include('apps.bookings.urls_v2')),   # New
]
```

---

## 📋 Quick Decision Matrix

| Scenario | Create New App? | Reasoning |
|----------|-----------------|-----------|
| Blog with Posts & Users | ❌ No | Too simple, keep in one `api` app |
| E-commerce (Products, Orders, Payments) | ✅ Yes | 3 distinct domains, will grow complex |
| Social media (Posts, Comments, Likes) | ❌ No (initially) | Keep in `posts` app until complex |
| Marketplace (Vendors, Products, Orders, Reviews) | ✅ Yes | Multi-vendor requires separation |
| Booking platform (Services, Bookings, Payments) | ✅ Yes | Complex workflows, integrations |
| Admin dashboard with reports | ❌ No | Add to `core` or existing admin |
| Real-time chat | ✅ Yes | Independent feature, WebSocket |
| Email notifications | ❌ Maybe | If simple → `core`, if complex → separate |

---

## 🎯 Final Summary

### Your Current Project (Simple Blog)
```
✅ Single `api` app is perfect
   - 2 simple models (User, Post)
   - Basic CRUD operations
   - Won't grow complex
```

### Home Services Platform (Complex)
```
✅ Need 8+ apps
   - users, providers, services, bookings
   - reviews, payments, notifications, chat
   - Each domain is complex and independent
```

### API Versioning
```
✅ Use /api/v1/ from the start
   - Easy to add /api/v2/ later
   - Protects existing clients
   - Allows breaking changes
```

---

## Key Principles

1. **Start simple** - Don't over-engineer early
2. **Refactor when needed** - Split apps as complexity grows
3. **Think reusability** - Can this app work in other projects?
4. **Business boundaries** - Each app should represent a clear domain
5. **Team scalability** - Separate apps allow parallel development
6. **Version from day one** - `/api/v1/` prevents future pain

---

**Remember:** The goal is maintainability and scalability, not just organization!
