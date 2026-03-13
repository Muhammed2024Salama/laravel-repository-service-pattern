# 🧠 ليه نستخدم (Interface - Repository - Service) في Laravel؟

## ✅ الهدف:

نفصل المسؤوليات (Separation of Concerns) وده مهم جدًا في المشاريع المتوسطة والكبيرة.

- **Controller:** يستقبل الطلب ويتعامل مع الرد (Response).
- **Service:** يحتوي على منطق العمل (Business Logic).
- **Repository:** يتعامل مع قاعدة البيانات مباشرة.
- **Interface:** يحدد التعاقد بين الـ Repository وباقي المشروع، وبيسهّل التبديل أو الاختبار.

---

## 🧱 الخطوة 1: إنشاء الـ Product Model مع Migration

```bash
php artisan make:model Product -m
```

شرح:  
`-m` تعني إنشاء ملف Migration تلقائي مع الموديل.

### ✏️ في ملف Migration:

```php
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->decimal('price', 10, 2); // السعر بفاصلة عشرية
    $table->timestamps();
});
```

ثم نشغّل الأمر:

```bash
php artisan migrate
```

---

## 🧱 الخطوة 2: إنشاء الـ Interface + Typing

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

## 🧱 الخطوة 3: إنشاء الـ Repository

```bash
touch app/Repositories/ProductRepository.php
```

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

## ⚙️ الخطوة 4: ربط الـ Interface بالـ Repository في `AppServiceProvider`

افتح `app/Providers/AppServiceProvider.php` وأضف:

```php
use App\Repositories\ProductRepositoryInterface;
use App\Repositories\ProductRepository;

public function register()
{
    $this->app->bind(ProductRepositoryInterface::class, ProductRepository::class);
}
```

---

## 🧠 الخطوة 5: إنشاء الـ Service

```bash
mkdir app/Services
touch app/Services/ProductService.php
```

```php
<?php

namespace App\Services;

use App\Repositories\ProductRepositoryInterface;
use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

class ProductService
{
    protected ProductRepositoryInterface $productRepository;

    public function __construct(ProductRepositoryInterface $productRepository)
    {
        $this->productRepository = $productRepository;
    }

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
        // مثال بسيط على Business Logic
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

## 🎮 الخطوة 6: إنشاء الـ Form Request + Controller مع Resource

```php
<?php

namespace App\Http\Controllers;

use App\Services\ProductService;
use App\Http\Requests\StoreProductRequest;
use App\Http\Resources\ProductResource;

class ProductController extends Controller
{
    protected ProductService $productService;

    public function __construct(ProductService $productService)
    {
        $this->productService = $productService;
    }

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

### مثال على Form Request:

```bash
php artisan make:request StoreProductRequest
```

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreProductRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'name'  => ['required', 'string', 'max:255'],
            'price' => ['required', 'numeric', 'min:0'],
        ];
    }
}
```

### مثال على Resource:

```bash
php artisan make:resource ProductResource
```

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class ProductResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     */
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'price'      => $this->price,
            'created_at' => $this->created_at,
        ];
    }
}
```
```

---

## 🛣️ الخطوة 7: إعداد الـ Routes

في `routes/api.php`:

```php
use App\Http\Controllers\ProductController;

Route::resource('products', ProductController::class);
```

---

## 📌 الترتيب النهائي:

1. **Interface:** تعريف قواعد التعامل.
2. **Repository:** تنفيذ العمليات مع قاعدة البيانات.
3. **Service:** يحتوي منطق العمل.
4. **Controller:** يستقبل الطلب ويرد بالنتيجة (باستخدام Form Request + Resource).
5. **Route:** يوصل بين الـ Endpoint و الـ Controller.

### Architecture Diagram (Simplified Flow)

Client  
↓  
Route  
↓  
Controller  
↓  
Service  
↓  
Repository  
↓  
Database

---

## 🗂 مقترح Folder Structure

```text
app
 ├── Http
 │   ├── Controllers
 │   │   └── ProductController.php
 │   ├── Requests
 │   │   └── StoreProductRequest.php
 │   └── Resources
 │       └── ProductResource.php
 │
 ├── Repositories
 │   ├── ProductRepositoryInterface.php
 │   └── ProductRepository.php
 │
 ├── Services
 │   └── ProductService.php
```

---

## 🚀 Future Improvements

- Add Unit Tests للـ Service و Repository.
- Add Feature Tests لسيناريوهات الـ API (index/store/show/update/destroy).
- إضافة طبقة DTO/Data Transformers لو حابب تفصل بين الـ Request Data والـ Domain Objects بشكل أكبر.

---

## 👨‍💻 Author

Muhammed Salama  
[devmuhammedsalama@gmail.com](mailto:devmuhammedsalama@gmail.com)