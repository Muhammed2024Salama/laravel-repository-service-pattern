![Laravel](https://img.shields.io/badge/Laravel-11-red)
![PHP](https://img.shields.io/badge/PHP-8.3-blue)
![License](https://img.shields.io/badge/license-MIT-green)

# Laravel Clean Architecture (Repository + Service Pattern)  
# مثال: معمارية نظيفة (Repository + Service) في Laravel

This repository demonstrates how to structure a Laravel application using **Repository Pattern**, **Service Layer**, **Form Requests**, **API Resources**, and **Dependency Injection**.  
الهدف: فصل المسؤوليات (Separation of Concerns) وبناء تطبيق واضح وقابل للصيانة والاختبار.

---

## Table of Contents

- [Introduction](#introduction)
- [Why Use Repository & Service Pattern](#why-use-interface-repository-and-service)
- [Installation](#installation)
- [Create Model](#step-1--create-product-model)
- [Repository Interface](#step-2--repository-interface-with-typing)
- [Repository Implementation](#step-3--implement-repository)
- [Service Layer](#step-5--service-layer-business-logic)
- [Controller + Validation + Resource](#step-6--controller--form-request--resource)
- [Routes](#step-7--routes)
- [Architecture Diagram](#architecture-diagram-)
- [Folder Structure](#folder-structure-)
- [Future Improvements](#future-improvements-)

---

## Introduction

تطبيق Laravel منظم بطبقات واضحة (Repository, Service, Form Request, API Resource) وقابل للتوسع والاختبار.  
*A clean, layered Laravel example ready for learning and reuse.*

---

## Why Use Interface, Repository, and Service?  
## ليه نستخدم Interface و Repository و Service؟

- **Controller:** Handles HTTP requests and responses. — *يستقبل الطلب ويرد بالنتيجة.*
- **Service:** Contains the business logic. — *يحتوي منطق العمل (Business Logic).*
- **Repository:** Handles database interactions. — *يتعامل مع قاعدة البيانات.*
- **Interface:** Defines contracts for flexible implementations. — *يعرّف التعاقد ويُسهّل التبديل والاختبار.*

---

## Installation  
## التثبيت والتشغيل

لتجربة المشروع محلياً / *To run the project locally:*

```bash
git clone https://github.com/your-username/laravel-repository-service-pattern.git
cd laravel-repository-service-pattern
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
php artisan serve
```

> غيّر `your-username` إلى اسم المستخدم أو الرابط الفعلي للريبو.  
> *Replace `your-username` with your GitHub username or the actual repo URL.*

---

## Step 1 — Create Product Model  
## الخطوة 1: إنشاء موديل Product

```bash
php artisan make:model Product -m
```

`-m` ينشئ ملف Migration مع الموديل. / *Creates a migration file.*

### Update Migration / تعديل الـ Migration

```php
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->decimal('price', 10, 2); // السعر بفاصلة عشرية
    $table->timestamps();
});
```

ثم: `php artisan migrate`

---

## Step 2 — Repository Interface (with Typing)  
## الخطوة 2: واجهة الـ Repository مع الـ Typing

```php
<?php

namespace App\Repositories;

use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

interface ProductRepositoryInterface
{
    public function getAll(): Collection;
    public function getById(int $id): Product;
    public function create(array $data): Product;
    public function update(int $id, array $data): Product;
    public function delete(int $id): int;
}
```

---

## Step 3 — Implement Repository  
## الخطوة 3: تنفيذ الـ Repository

```php
<?php

namespace App\Repositories;

use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

class ProductRepository implements ProductRepositoryInterface
{
    public function getAll(): Collection
    {
        return Product::all();
    }

    public function getById(int $id): Product
    {
        return Product::findOrFail($id);
    }

    public function create(array $data): Product
    {
        return Product::create($data);
    }

    public function update(int $id, array $data): Product
    {
        $product = Product::findOrFail($id);
        $product->update($data);

        return $product;
    }

    public function delete(int $id): int
    {
        return Product::destroy($id);
    }
}
```

---

## Step 4 — Bind Interface in Service Provider  
## الخطوة 4: ربط الـ Interface في AppServiceProvider

في `app/Providers/AppServiceProvider.php` داخل `register()`:

```php
use App\Repositories\ProductRepositoryInterface;
use App\Repositories\ProductRepository;

$this->app->bind(
    ProductRepositoryInterface::class,
    ProductRepository::class
);
```

---

## Step 5 — Service Layer (Business Logic)  
## الخطوة 5: طبقة الـ Service (منطق العمل)

الـ Service هنا مش مجرد pass-through؛ فيه منطق فعلي (مثلاً تقريب السعر).

```php
<?php

namespace App\Services;

use App\Repositories\ProductRepositoryInterface;
use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

class ProductService
{
    public function __construct(
        protected ProductRepositoryInterface $productRepository
    ) {}

    public function getAll(): Collection
    {
        return $this->productRepository->getAll();
    }

    public function getById(int $id): Product
    {
        return $this->productRepository->getById($id);
    }

    public function create(array $data): Product
    {
        // Business logic مثال
        if (isset($data['price'])) {
            $data['price'] = round($data['price'], 2);
        }

        return $this->productRepository->create($data);
    }

    public function update(int $id, array $data): Product
    {
        return $this->productRepository->update($id, $data);
    }

    public function delete(int $id): int
    {
        return $this->productRepository->delete($id);
    }
}
```

---

## Step 6 — Controller + Form Request + Resource  
## الخطوة 6: الـ Controller مع Form Request و API Resource

- **Form Request** بدل `Request` + `$request->all()` — تحقق من البيانات (Validation).
- **API Resource** بدل `response()->json()` — تنظيم شكل الـ API.

```php
<?php

namespace App\Http\Controllers;

use App\Services\ProductService;
use App\Http\Requests\StoreProductRequest;
use App\Http\Resources\ProductResource;

class ProductController extends Controller
{
    public function __construct(
        protected ProductService $productService
    ) {}

    public function index()
    {
        return ProductResource::collection(
            $this->productService->getAll()
        );
    }

    public function store(StoreProductRequest $request)
    {
        $product = $this->productService->create($request->validated());

        return (new ProductResource($product))
            ->response()
            ->setStatusCode(201);
    }

    public function show(int $id)
    {
        return new ProductResource(
            $this->productService->getById($id)
        );
    }

    public function update(StoreProductRequest $request, int $id)
    {
        $product = $this->productService->update($id, $request->validated());

        return new ProductResource($product);
    }

    public function destroy(int $id)
    {
        $this->productService->delete($id);

        return response()->noContent(); // 204 No Content
    }
}
```

### Form Request Example / مثال Form Request

```bash
php artisan make:request StoreProductRequest
```

```php
public function rules(): array
{
    return [
        'name'  => ['required', 'string', 'max:255'],
        'price' => ['required', 'numeric', 'min:0'],
    ];
}
```

### API Resource Example / مثال API Resource

```bash
php artisan make:resource ProductResource
```

```php
public function toArray(Request $request): array
{
    return [
        'id'         => $this->id,
        'name'       => $this->name,
        'price'      => $this->price,
        'created_at' => $this->created_at,
    ];
}
```

---

## Step 7 — Routes  
## الخطوة 7: الـ Routes

في `routes/api.php`:

```php
use App\Http\Controllers\ProductController;

Route::resource('products', ProductController::class);
```

---

## Architecture Diagram  
## رسم تدفق المعمارية

```
Client
  ↓
Route
  ↓
Controller
  ↓
Service Layer
  ↓
Repository Layer
  ↓
Database
```

---

## Folder Structure / هيكل المجلدات

```text
app
 ├── Http
 │   ├── Controllers
 │   ├── Requests
 │   └── Resources
 │
 ├── Repositories
 │   ├── ProductRepositoryInterface.php
 │   └── ProductRepository.php
 │
 ├── Services
 │   └── ProductService.php
```

---

## Future Improvements / تحسينات مقترحة

- Add Unit Tests — إضافة اختبارات وحدة للـ Service و Repository.
- Add Feature Tests — اختبارات سيناريوهات الـ API.
- Implement DTO Layer — طبقة DTO لفصل بيانات الطلب عن النطاق.
- Add Caching in Repository — تخزين مؤقت في طبقة الـ Repository.

---

## Author / المؤلف

**Muhammed Salama**  
[devmuhammedsalama@gmail.com](mailto:devmuhammedsalama@gmail.com)
