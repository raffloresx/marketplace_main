# 🚀 SPRINT 1 — Setup + Modelado Base
## 🎯 Objetivo
Tener:
* Proyecto Django funcionando
* App store (tienda)
* Modelos con relaciones:
    - 1:N → Usuario → Producto
    - N:M → Producto ↔ Categoría
    - N:M → Carrito ↔ Producto (con CartItem)
* Admin operativo

## 1 Crear proyecto en django
```bash
django-admin startproject marketplace_main
cd marketplace_main
python manage.py startapp store
```

## 2 Configurar settings.py
📌 Agregar app
```python
INSTALLED_APPS = [
    ...
    'store',
]
```
📌 Usuario personalizado
```python
AUTH_USER_MODEL = 'store.User'
```

## 3 MODELOS (CLAVE DEL PROYECTO)
Diseño de la Base De Datos Relacional en Diagrama Entidad-Relacion
![Diagrama Entidad-Relacion](https://github.com/jcromerohdz/marketplace_main_2026/blob/main/Diagrama_Entidad_Ralacion.png)
📁 marketplace/models.py
```python
import uuid
from django.db import models
from django.contrib.auth.models import AbstractUser

# =========================
# 👤 Usuario
# =========================
class User(AbstractUser):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    is_seller = models.BooleanField(default=False)

    def __str__(self):
        return self.username


# =========================
# 🏷️ Categoría
# =========================
class Category(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=100)
    description = models.TextField(blank=True)

    def __str__(self):
        return self.name


# =========================
# 📦 Producto
# =========================
class Product(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=150)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField(default=0)

    owner = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='products'
    )  # 1:N

    categories = models.ManyToManyField(
        Category,
        related_name='products'
    )  # N:M

    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name


# =========================
# 🛒 Carrito
# =========================
class Cart(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    user = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='carts'
    )  # 1:N

    products = models.ManyToManyField(
        Product,
        through='CartItem',
        related_name='carts'
    )

    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Cart {self.id} - {self.user}"


# =========================
# 🧾 CartItem (tabla intermedia)
# =========================
class CartItem(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    cart = models.ForeignKey(Cart, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)

    quantity = models.PositiveIntegerField(default=1)

    class Meta:
        unique_together = ('cart', 'product')

    def __str__(self):
        return f"{self.product} x {self.quantity}"
```

## 4 Admin de django
📁 store/admin.py
```python
from django.contrib import admin
from .models import User, Category, Product, Cart, CartItem


@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ('username', 'email', 'is_seller')
    list_filter = ('is_seller',)


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display = ('name',)


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display = ('name', 'price', 'stock', 'owner')
    list_filter = ('categories',)
    search_fields = ('name',)


class CartItemInline(admin.TabularInline):
    model = CartItem
    extra = 1


@admin.register(Cart)
class CartAdmin(admin.ModelAdmin):
    list_display = ('id', 'user', 'created_at')
    inlines = [CartItemInline]


@admin.register(CartItem)
class CartItemAdmin(admin.ModelAdmin):
    list_display = ('cart', 'product', 'quantity')
```

## 5 Migraciones
```bash
python manage.py makemigrations
python manage.py migrate
```

## 6 Crear super useario
```bash
python manage.py createsuperuser
```

## 7 Ejecutar Servidor
```bash
python manage.py runserver
```
👉 Ir a:
```bash
http://127.0.0.1:8000/admin/
```

# 🚀 SPRINT 2 — Autenticación + UI Base
## 🎯 Objetivo
* Registro, login, logout
* Layout base con Bootstrap 5.3
* Navbar dinámica (login / logout)
* Listado de productos (cards)
* Home pública

1. URLs del proyecto
📁 config/urls.py
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('store.urls')),
]
```

2. URLs de la app
📁 crea el archivo urls.py en store/urls.py y agrega lo siguiente:
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='home'),
    path('register/', views.register, name='register'),
    path('login/', views.login_view, name='login'),
    path('logout/', views.logout_view, name='logout'),
]
```

3. Formularios
📁 crea el archivo forms.py en store/forms.py y agregua lo siguiente:
```python
from django import forms
from django.contrib.auth.forms import UserCreationForm
from .models import User

class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)
    is_seller = forms.BooleanField(required=False)

    class Meta:
        model = User
        fields = ('username', 'email', 'is_seller', 'password1', 'password2')
```

4. Vistas
📁 marketplace/views.py agrega lo siguiente:
```python
from django.shortcuts import render, redirect
from django.contrib.auth import login, authenticate, logout
from .forms import RegisterForm
from .models import Product


def home(request):
    products = Product.objects.select_related('owner').prefetch_related('categories').all()
    return render(request, 'marketplace/home.html', {'products': products})


def register(request):
    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            return redirect('home')
    else:
        form = RegisterForm()

    return render(request, 'marketplace/register.html', {'form': form})


def login_view(request):
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')

        user = authenticate(request, username=username, password=password)

        if user:
            login(request, user)
            return redirect('home')

    return render(request, 'marketplace/login.html')


def logout_view(request):
    logout(request)
    return redirect('home')
```

5. Templates (Bootstrap 5.3)
📁 Crear la carpeta templates y dentro la carpeta store para agregar lo siguientes archivos de la Estructura que se muestra
```bash
templates/
 └── store/
     ├── base.html
     ├── home.html
     ├── login.html
     └── register.html
```

🧩 base.html
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Marketplace</title>

    <!-- Bootstrap 5.3 -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>

<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <div class="container">
    <a class="navbar-brand" href="/">Marketplace</a>

    <div>
      {% if user.is_authenticated %}
        <span class="text-white me-3">Hola {{ user.username }}</span>
        <a href="{% url 'logout' %}" class="btn btn-outline-light btn-sm">Logout</a>
      {% else %}
        <a href="{% url 'login' %}" class="btn btn-outline-light btn-sm me-2">Login</a>
        <a href="{% url 'register' %}" class="btn btn-primary btn-sm">Registro</a>
      {% endif %}
    </div>
  </div>
</nav>

<div class="container mt-4">
    {% block content %}{% endblock %}
</div>

</body>
</html>
```

🏠 home.html
```bash
{% extends 'store/base.html' %}

{% block content %}

<h2 class="mb-4">Productos</h2>

<div class="row">
    {% for product in products %}
    <div class="col-md-4">
        <div class="card mb-4 shadow-sm">
            <div class="card-body">
                <h5>{{ product.name }}</h5>
                <p>{{ product.description|truncatechars:80 }}</p>

                <p><strong>$ {{ product.price }}</strong></p>

                <small class="text-muted">
                    Vendedor: {{ product.owner.username }}
                </small>

                <div class="mt-2">
                    {% for cat in product.categories.all %}
                        <span class="badge bg-secondary">{{ cat.name }}</span>
                    {% endfor %}
                </div>

            </div>
        </div>
    </div>
    {% empty %}
        <p>No hay productos aún.</p>
    {% endfor %}
</div>

{% endblock %}
```

🔐 login.html
```html
{% extends 'marketplace/base.html' %}

{% block content %}

<h2>Login</h2>

<form method="POST">
    {% csrf_token %}
    <input type="text" name="username" placeholder="Usuario" class="form-control mb-2">
    <input type="password" name="password" placeholder="Contraseña" class="form-control mb-2">

    <button class="btn btn-primary">Ingresar</button>
</form>

{% endblock %}
```

📝 register.html
```html
{% extends 'marketplace/base.html' %}

{% block content %}

<h2>Registro</h2>

<form method="POST">
    {% csrf_token %}
    {{ form.as_p }}

    <button class="btn btn-success">Registrarse</button>
</form>

{% endblock %}
```

6. Ejecutar el proyecto
```bash
python manage.py runserver
```

🧪 7. Flujo de prueba
1. Ir a /register/
2. Crear usuario
3. Login automático
4. Ver productos en /
5. Logout desde navbar

✅ Resultado del Sprint 2
- ✔ Autenticación completa
- ✔ UI base profesional con Bootstrap
- ✔ Navbar dinámica
- ✔ Listado de productos
- ✔ Estructura lista para escalar

# 🚀 SPRINT 3 — Gestión de Productos + Dashboard
## 🎯 Objetivo
* CRUD completo de productos
* Dashboard para vendedores
* Control de permisos (solo dueño edita)
* UI tipo panel (estilo Amazon Seller)

## 🧱 1. Actualizar URLs
📁 store/urls.py
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='home'),

    # Auth
    path('register/', views.register, name='register'),
    path('login/', views.login_view, name='login'),
    path('logout/', views.logout_view, name='logout'),

    # Dashboard
    path('dashboard/', views.dashboard, name='dashboard'),

    # Productos CRUD
    path('products/create/', views.product_create, name='product_create'),
    path('products/<uuid:pk>/edit/', views.product_update, name='product_update'),
    path('products/<uuid:pk>/delete/', views.product_delete, name='product_delete'),
]
```

## 🧠 2. Formulario de Producto
📁 store/forms.py crear este archivo en la aplicación store
```python
from django import forms
from .models import Product

class ProductForm(forms.ModelForm):
    class Meta:
        model = Product
        fields = ['name', 'description', 'price', 'stock', 'categories']
        widgets = {
            'categories': forms.CheckboxSelectMultiple()
        }
```

## 🧠 3. Vistas-Views (LÓGICA DE NEGOCIO)
📁 store/views.py
```python
from django.contrib.auth.decorators import login_required
from django.shortcuts import get_object_or_404
from django.http import HttpResponseForbidden
from .forms import ProductForm
from .models import Product

# =========================
# 📊 Dashboard
# =========================
@login_required
def dashboard(request):
    if not request.user.is_seller:
        return HttpResponseForbidden("No tienes permisos")

    products = Product.objects.filter(owner=request.user)

    return render(request, 'store/dashboard.html', {
        'products': products
    })


# =========================
# ➕ Crear producto
# =========================
@login_required
def product_create(request):
    if not request.user.is_seller:
        return HttpResponseForbidden("Solo vendedores")

    form = ProductForm(request.POST or None)

    if form.is_valid():
        product = form.save(commit=False)
        product.owner = request.user
        product.save()
        form.save_m2m()

        return redirect('dashboard')

    return render(request, 'store/product_form.html', {'form': form})


# =========================
# ✏️ Editar producto
# =========================
@login_required
def product_update(request, pk):
    product = get_object_or_404(Product, pk=pk)

    if product.owner != request.user:
        return HttpResponseForbidden("No puedes editar este producto")

    form = ProductForm(request.POST or None, instance=product)

    if form.is_valid():
        form.save()
        return redirect('dashboard')

    return render(request, 'store/product_form.html', {'form': form})


# =========================
# 🗑️ Eliminar producto
# =========================
@login_required
def product_delete(request, pk):
    product = get_object_or_404(Product, pk=pk)

    if product.owner != request.user:
        return HttpResponseForbidden("No puedes eliminar este producto")

    if request.method == 'POST':
        product.delete()
        return redirect('dashboard')

    return render(request, 'store/product_confirm_delete.html', {'product': product})
```

## 🎨 4. Dashboard UI (tipo Amazon simple)
📁 templates/store/dashboard.html crear este template en la runta indicada.
```html
{% extends 'store/base.html' %}

{% block content %}

<div class="d-flex justify-content-between align-items-center mb-4">
    <h2>Mi Dashboard</h2>
    <a href="{% url 'product_create' %}" class="btn btn-success">+ Nuevo Producto</a>
</div>

<table class="table table-striped">
    <thead>
        <tr>
            <th>Nombre</th>
            <th>Precio</th>
            <th>Stock</th>
            <th>Acciones</th>
        </tr>
    </thead>
    <tbody>

    {% for product in products %}
        <tr>
            <td>{{ product.name }}</td>
            <td>$ {{ product.price }}</td>
            <td>{{ product.stock }}</td>
            <td>
                <a href="{% url 'product_update' product.id %}" class="btn btn-warning btn-sm">Editar</a>

                <a href="{% url 'product_delete' product.id %}" class="btn btn-danger btn-sm">Eliminar</a>
            </td>
        </tr>
    {% endfor %}

    </tbody>
</table>

{% endblock %}
```
## 🧾 5. Formulario UI
📁 templates/store/product_form.html crear este template en la runta indicada.
```html
{% extends 'store/base.html' %}

{% block content %}

<h2>Producto</h2>

<form method="POST">
    {% csrf_token %}
    {{ form.as_p }}

    <button class="btn btn-primary">Guardar</button>
</form>

{% endblock %}
```

## ⚠️ 6. Confirmación de eliminación de producto
📁 templates/store/product_confirm_delete.html crear este template en la runta indicada.
```html
{% extends 'marketplace/base.html' %}

{% block content %}

<h3>¿Eliminar "{{ product.name }}"?</h3>

<form method="POST">
    {% csrf_token %}
    <button class="btn btn-danger">Sí, eliminar</button>
    <a href="{% url 'dashboard' %}" class="btn btn-secondary">Cancelar</a>
</form>

{% endblock %}
```
## 🎯 7. Mejora UX en Navbar
📁 En templates/store/product_confirm_delete.html agrega lo siguiente a la barra de navegación.
```html
{% if user.is_authenticated and user.is_seller %}
    <a href="{% url 'dashboard' %}" class="btn btn-warning btn-sm me-2">Dashboard</a>
{% endif %}
```

## 🧪 8. Flujo de prueba
1. Crear usuario vendedor (is_seller = True)
2. Ir a /dashboard/
3. Crear producto
4. Editar producto
5. Eliminar producto
6. Ver reflejado en home

# 🚀 SPRINT 4 — Carrito de Compras Completo
## 🎯 Objetivos
Implementar:
* ✔ Agregar productos al carrito
* ✔ Actualizar cantidades
* ✔ Eliminar productos
* ✔ Calcular subtotal y total
* ✔ Persistencia en base de datos
* ✔ UI Bootstrap 5.3
* ✔ Seguridad por usuario autenticado

## 🧱 Arquitectura del carrito
Ya tienes estos modelos:
```bash
Product
Cart
CartItem
```
Ahora agregaremos:

* lógica de negocio
* vistas
* templates
* cálculo de totales

## 🧠 1. Mejorar modelos (IMPORTANTE)
📁 store/models.py actualizar lo siguiente:
```python
class CartItem(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    cart = models.ForeignKey(Cart, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)

    quantity = models.PositiveIntegerField(default=1)

    class Meta:
        unique_together = ('cart', 'product')

    def __str__(self):
        return f"{self.product} x {self.quantity}"

    # Actualizar
    @property
        def subtotal(self):
            return self.product.price * self.quantity

        def __str__(self):
            return f"{self.product} x {self.quantity}"
```
Continuemos con 🧠 Agregar total al carrito, dentro del modelo Cart:
```python
class Cart(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    user = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='carts'
    )  # 1:N

    products = models.ManyToManyField(
        Product,
        through='CartItem',
        related_name='carts'
    )

    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Cart {self.id} - {self.user}"
    
    # Actualizar    
    @property
    def total(self):
        return sum(item.subtotal for item in self.cartitem_set.all())
```

## ⚙️ 2. Migraciones de los modelos es decir las actualizaciones de la Base de Datos
```bash
python manage.py makemigrations
python manage.py migrate
```

## 🧱 3. URLs del carrito
En 📁 store/urls.py actualizar debajo de CRUD de productos con lo siguiente:
```python
# Cart
path('cart/', views.cart_detail, name='cart_detail'),
path('cart/add/<uuid:product_id>/', views.add_to_cart, name='add_to_cart'),
path('cart/remove/<uuid:item_id>/', views.remove_from_cart, name='remove_from_cart'),
path('cart/update/<uuid:item_id>/', views.update_cart_item, name='update_cart_item'),
```

## 🧠 4. Vistas del carrito
📁 store/views.py actualizar para la funcionalidad de carrito
```python
from .models import Cart, CartItem


# =========================
# 🛒 Ver carrito
# =========================
@login_required
def cart_detail(request):
    cart = Cart.objects.get(user=request.user)

    return render(request, 'marketplace/cart_detail.html', {
        'cart': cart
    })


# =========================
# ➕ Agregar producto
# =========================
@login_required
def add_to_cart(request, product_id):
    cart = Cart.objects.get(user=request.user)

    product = get_object_or_404(Product, id=product_id)

    cart_item, created = CartItem.objects.get_or_create(
        cart=cart,
        product=product
    )

    if not created:
        cart_item.quantity += 1
        cart_item.save()

    return redirect('cart_detail')


# =========================
# ❌ Eliminar item
# =========================
@login_required
def remove_from_cart(request, item_id):
    item = get_object_or_404(
        CartItem,
        id=item_id,
        cart__user=request.user
    )

    item.delete()

    return redirect('cart_detail')


# =========================
# 🔄 Actualizar cantidad
# =========================
@login_required
def update_cart_item(request, item_id):
    item = get_object_or_404(
        CartItem,
        id=item_id,
        cart__user=request.user
    )

    if request.method == 'POST':
        quantity = int(request.POST.get('quantity'))

        if quantity > 0:
            item.quantity = quantity
            item.save()
        else:
            item.delete()

    return redirect('cart_detail')
```

## 🎨 5. UI — Carrito
📁 En templates/store/cart_detail.html crear este archivo y agregar:
```python
{% extends 'store/base.html' %}

{% block content %}

<h2 class="mb-4">Mi Carrito</h2>

{% if cart.cartitem_set.all %}

<table class="table table-bordered align-middle">
    <thead>
        <tr>
            <th>Producto</th>
            <th>Precio</th>
            <th>Cantidad</th>
            <th>Subtotal</th>
            <th></th>
        </tr>
    </thead>

    <tbody>

    {% for item in cart.cartitem_set.all %}
        <tr>

            <td>{{ item.product.name }}</td>

            <td>$ {{ item.product.price }}</td>

            <td>
                <form method="POST"
                      action="{% url 'update_cart_item' item.id %}">
                    {% csrf_token %}

                    <input type="number"
                           name="quantity"
                           value="{{ item.quantity }}"
                           min="1"
                           class="form-control"
                           style="width:90px;">

                    <button class="btn btn-sm btn-primary mt-2">
                        Actualizar
                    </button>
                </form>
            </td>

            <td>
                $ {{ item.subtotal }}
            </td>

            <td>
                <a href="{% url 'remove_from_cart' item.id %}"
                   class="btn btn-danger btn-sm">
                    Eliminar
                </a>
            </td>

        </tr>
    {% endfor %}

    </tbody>
</table>

<div class="text-end">
    <h3>Total: $ {{ cart.total }}</h3>
</div>

{% else %}

<div class="alert alert-info">
    Tu carrito está vacío
</div>

{% endif %}

{% endblock %}
```
## 🛍️ 6. Botón “Agregar al carrito”
📁 home.html Dentro de cada card actualiza:
```html
{% if user.is_authenticated %}
    <a href="{% url 'add_to_cart' product.id %}"
       class="btn btn-success mt-3">
       Agregar al carrito
    </a>
{% endif %}
```

##🧭 7. Navbar con carrito
📁 base.html actualiza con:
```html
<a href="{% url 'cart_detail' %}"
   class="btn btn-outline-light btn-sm me-2">
   Carrito
</a>
```
## 🧪 9. Flujo de prueba
1 Crear productos

2 Iniciar sesión

3 Agregar productos al carrito

4 Actualizar cantidades

5 Eliminar productos

6 Ver total dinámico
