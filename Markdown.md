# 🏁 أولاً: ليه نستخدم (Interface, Repository, Service) من الأساس؟

## ✅ الهدف:
نفصل المسؤوليات، بحيث:

- **Controller:** يستقبل الطلب ويتحكم في "السيناريو".
- **Service:** ينفذ المنطق الرئيسي (business logic).
- **Repository:** يتعامل مع قاعدة البيانات.
- **Interface:** يعرّف قواعد التعامل، بحيث تقدر تبدل أو تغيّر مصدر البيانات في أي وقت بسهولة.

## 🧱 الخطوة 1: إنشاء الـ Product Model
```bash
php artisan make:model Product -m
شرح:
make:model: أمر بيقول للـ Laravel: "اعمللي موديل".

-m: ينشئ ملف Migration معاً لبناء جدول المنتجات.

🧱 الخطوة 2: نفتح ملف الـ Migration ونعدّل جدول المنتجات
php

Schema::create('products', function (Blueprint $table) {
    $table->id(); 
    $table->string('name');
    $table->decimal('price', 10, 2); // السعر بفاصلة عشرية
    $table->timestamps(); 
});
ليه بنكتب كده؟
Laravel بيوفر طريقة سهلة لتعريف الجداول من غير ما تكتب SQL يدوياً.

بعد كده:

php artisan migrate
ينشئ الجداول في قاعدة البيانات.

🧱 الخطوة 3: نبدأ بالـ Interface مع Typing
<?php

namespace App\Repositories;

use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

interface ProductRepositoryInterface
{
    public function getAll(): Collection; // هترجع كل المنتجات
    public function getById(int $id): Product; // منتج واحد بالـ ID
    public function create(array $data): Product; // إنشاء منتج
    public function update(int $id, array $data): Product; // تعديل منتج
    public function delete(int $id): int; // عدد الصفوف المحذوفة
}
ليه عملنا كده؟
الـ Interface يحدد الوظائف اللازمة لأي repository، من غير تفاصيل تنفيذ.

🧱 الخطوة 4: نعمل الـ Repository الحقيقي

touch app/Repositories/ProductRepository.php
ونكتب:

<?php

namespace App\Repositories;

use App\Models\Product;
use Illuminate\Database\Eloquent\Collection;

class ProductRepository implements ProductRepositoryInterface
{
    public function getAll(): Collection
    {
        return Product::all(); // يجيب كل المنتجات
    }

    public function getById(int $id): Product
    {
        return Product::findOrFail($id); // منتج واحد أو يرجّع 404
    }

    public function create(array $data): Product
    {
        return Product::create($data); // إنشاء جديد
    }

    public function update(int $id, array $data): Product
    {
        $product = Product::findOrFail($id);
        $product->update($data); // تعديل البيانات

        return $product;
    }

    public function delete(int $id): int
    {
        return Product::destroy($id); // حذف المنتج
    }
}
شرح:
ننفذ الوظائف المعرفة في الـ Interface، بالتعامل مع Model.

⚙️ الخطوة 5: نربط الـ Interface بالـ Repository في AppServiceProvider
ليه؟
ليعرف Laravel كيف يحقن ProductRepositoryInterface.

افتح:


app/Providers/AppServiceProvider.php

use App\Repositories\ProductRepositoryInterface;
use App\Repositories\ProductRepository;

public function register()
{
    $this->app->bind(ProductRepositoryInterface::class, ProductRepository::class);
}
🧠 الخطوة 6: نعمل Service لمنتجاتنا

mkdir app/Services
touch app/Services/ProductService.php
واكتب فيه:

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
شرح:
الـ Service ينظم منطق التعامل مع الـ Repository بشكل منظم.

🎮 الخطوة 7: نعمل Controller

php artisan make:controller ProductController

واكتب:

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

🧱 مثال على Form Request:

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

🧱 مثال على Resource:

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

🛣️ الخطوة 8: نضيف الـ Routes

افتح routes/web.php أو routes/api.php لو هتشتغل API:

use App\Http\Controllers\ProductController;

Route::resource('products', ProductController::class);

الترتيب:

Interface: تعريف التعامل مع البيانات.

Repository: الكود الفعلي للتعامل مع قاعدة البيانات.

Service: تنظيم المنطق الرئيسي للتعامل مع الـ Repository (مع Business Logic).

Controller: استقبال الطلبات وتوجيهها للـ Service (مع Form Request + Resource).

Route: ربط الـ Endpoint بالـ Controller.

Architecture Diagram (Flow مبسط):

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

🗂 مقترح Folder Structure:

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

🚀 Future Improvements:

- Add Unit Tests للـ Service و Repository.
- Add Feature Tests لسيناريوهات الـ API (index/store/show/update/destroy).
- إضافة طبقة DTO/Data Transformers لو حابب تفصل بين الـ Request Data والـ Domain Objects بشكل أكبر.

👨‍💻 Author
Muhammed Salama
devmuhammedsalama@gmail.com
