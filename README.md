![Laravel best practices](/resources/images/logo-english.png?raw=true)

It's not a Laravel adaptation of SOLID principles, patterns etc. Here you'll find the best practices which are usually ignored in real life Laravel projects.

## Contents

[Single responsibility principle](#single-responsibility-principle)

[Fat models, skinny controllers](#fat-models-skinny-controllers)

[Validation](#validation)

[Business logic should be in service class](#business-logic-should-be-in-service-class)

[Don't repeat yourself (DRY)](#dont-repeat-yourself-dry)

[Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections to arrays](#prefer-to-use-eloquent-over-using-query-builder-and-raw-sql-queries-prefer-collections-to-arrays)

[Mass assignment](#mass-assignment)

[Do not execute queries in Blade templates and use eager loading (N + 1 problem)](#do-not-execute-queries-in-blade-templates-and-use-eager-loading-n--1-problem)

[Chunk data for data-heavy tasks](#chunk-data-for-data-heavy-tasks)

[Prefer descriptive method and variable names over comments](#prefer-descriptive-method-and-variable-names-over-comments)

[Do not put JS and CSS in Blade templates and do not put any HTML in PHP classes](#do-not-put-js-and-css-in-blade-templates-and-do-not-put-any-html-in-php-classes)

[Use config and language files, constants instead of text in the code](#use-config-and-language-files-constants-instead-of-text-in-the-code)

[Use standard Laravel tools accepted by community](#use-standard-laravel-tools-accepted-by-community)

[Follow Laravel naming conventions](#follow-laravel-naming-conventions)

[Use shorter and more readable syntax where possible](#use-shorter-and-more-readable-syntax-where-possible)

[Use IoC container or facades instead of new Class](#use-ioc-container-or-facades-instead-of-new-class)

[Do not get data from the `.env` file directly](#do-not-get-data-from-the-env-file-directly)

[Store dates in the standard format. Use accessors and mutators to modify date format](#store-dates-in-the-standard-format-use-accessors-and-mutators-to-modify-date-format)

[Use Invokable Controller](#use-invokable-controller)

[Use Resource Controller](#use-resource-controller)

[Debugging / Performance](#debugging--performance)

[Consider using View Models](#consider-using-view-models)

[Consider using Laravel Query Builder package](#consider-using-laravel-query-builder-package)

[Other good practices](#other-good-practices)

### **Single responsibility principle**

A class and a method should have only one responsibility.

Bad:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Good:

```php
public function getFullNameAttribute(): bool
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient(): bool
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong(): string
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort(): string
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[???? Back to contents](#contents)

### **Fat models, skinny controllers**

Put all DB related logic into Eloquent models.

Bad:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Good:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders(): Collection
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[???? Back to contents](#contents)

### **Validation**

Move validation from controllers to Request classes.

Bad:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Good:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules(): array
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[???? Back to contents](#contents)

### **Business logic should be in service class**

A controller must have only one responsibility : get the parameters, call methods from external classes and returns the result or redirect
. So better to move the business logic from controllers to service classes.

Bad:

```php
public function store(Request $request)
{
    Article::create($request->validated());
    
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

Good:

```php
public function store(Request $request)
{
    $this->articleService->store($request->validated(), $request->file('image'));

    ....
}

class ArticleService
{

    public function store(array $data, $image)
    {
        Article::create($data);

        $this->handleUploadedImage($image);
    }
    
    private function handleUploadedImage($image): void
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

Can also consider using Use Cases : one Use Case by feature

```php
class StoreArticleUseCase
{
    public function handle()
    {
        // business logic
    }
}
```


[???? Back to contents](#contents)

### **Don't repeat yourself (DRY)**

Reuse code when you can. SRP is helping you to avoid duplication. Also, reuse Blade templates, use Eloquent scopes etc.

Bad:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Good:

```php
public function scopeActive($q)
{
    return $q->where('verified', true)->whereNotNull('deleted_at');
}

public function getActive(): Collection
{
    return $this->active()->get();
}

public function getArticles(): Collection
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[???? Back to contents](#contents)

### **Prefer to use Eloquent over using Query Builder and raw SQL queries. Prefer collections to arrays**

Eloquent allows you to write readable and maintainable code. Also, Eloquent has great built-in tools like soft deletes, events, scopes etc.

Bad:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Good:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[???? Back to contents](#contents)

### **Mass assignment**

Bad:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

Good:

```php
$category->article()->create($request->validated());
```

[???? Back to contents](#contents)

### **Do not execute queries in Blade templates and use eager loading (N + 1 problem)**

Bad (for 100 users, 101 DB queries will be executed):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Good (for 100 users, 2 DB queries will be executed):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[???? Back to contents](#contents)

### **Chunk data for data-heavy tasks**

Bad ():

```php
$users = $this->get();

foreach ($users as $user) {
    ...
}
```

Good:

```php
$this->chunk(500, function ($users) {
    foreach ($users as $user) {
        ...
    }
});
```

[???? Back to contents](#contents)

### **Prefer descriptive method and variable names over comments**

Bad:

```php
// Determine if there are any joins
if (count((array) $builder->getQuery()->joins) > 0)
```

Good:

```php
if ($this->hasJoins())
```

[???? Back to contents](#contents)

### **Do not put JS and CSS in Blade templates and do not put any HTML in PHP classes**

Bad:

```php
let article = `{{ json_encode($article) }}`;
```

Better:

```php
<input id="article" type="hidden" value='@json($article)'>

Or

<button class="js-fav-article" data-article='@json($article)'>{{ $article->name }}<button>
```

In a Javascript file:

```javascript
let article = $('#article').val();
```

The best way is to use specialized PHP to JS package to transfer the data.

[???? Back to contents](#contents)

### **Use config and language files, constants instead of text in the code**

Bad:

```php
public function isNormal(): bool
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Good:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[???? Back to contents](#contents)

### **Use standard Laravel tools accepted by community**

Prefer to use built-in Laravel functionality and community packages instead of using 3rd party packages and tools. Any developer who will work with your app in the future will need to learn new tools. Also, chances to get help from the Laravel community are significantly lower when you're using a 3rd party package or tool. Do not make your client pay for that.

Task | Standard tools | 3rd party tools
------------ | ------------- | -------------
Authorization | Policies | Entrust, Sentinel and other packages
Compiling assets | Laravel Mix | Grunt, Gulp, 3rd party packages
Development Environment | Laravel Sail, Homestead | Docker
Deployment | Laravel Forge | Deployer and other solutions
Unit testing | PHPUnit, Mockery | Phpspec
Browser testing | Laravel Dusk | Codeception
DB | Eloquent | SQL, Doctrine
Templates | Blade | Twig
Working with data | Laravel collections | Arrays
Form validation | Request classes | 3rd party packages, validation in controller
Authentication | Built-in | 3rd party packages, your own solution
API authentication | Laravel Passport, Laravel Sanctum | 3rd party JWT and OAuth packages
Creating API | Built-in | Dingo API and similar packages
Working with DB structure | Migrations | Working with DB structure directly
Localization | Built-in | 3rd party packages
Realtime user interfaces | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly
Generating testing data | Seeder classes, Model Factories, Faker | Creating testing data manually
Task scheduling | Laravel Task Scheduler | Scripts and 3rd party packages
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[???? Back to contents](#contents)

### **Follow Laravel naming conventions**

Follow [PSR standards](http://www.php-fig.org/psr/psr-2/).

Also, follow naming conventions accepted by Laravel community:

What | How | Good | Bad
------------ | ------------- | ------------- | -------------
Controller | singular | ArticleController | ~~ArticlesController~~
Route | plural | articles/1 | ~~article/1~~
Named route | snake_case with dot notation | users.show_active | ~~users.show-active, show-active-users~~
Model | singular | User | ~~Users~~
hasOne or belongsTo relationship | singular | articleComment | ~~articleComments, article_comment~~
All other relationships | plural | articleComments | ~~articleComment, article_comments~~
Table | plural | article_comments | ~~article_comment, articleComments~~
Pivot table | singular model names in alphabetical order | article_user | ~~user_article, articles_users~~
Table column | snake_case without model name | meta_title | ~~MetaTitle; article_meta_title~~
Model property | snake_case | $model->created_at | ~~$model->createdAt~~
Foreign key | singular model name with _id suffix | article_id | ~~ArticleId, id_article, articles_id~~
Primary key | - | id | ~~custom_id~~
Migration | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Method | camelCase | getAll | ~~get_all~~
Method in resource controller | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Method in test class | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variable | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collection | descriptive, plural | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | descriptive, singular | $activeUser = User::active()->first() | ~~$users, $obj~~
Config and language files index | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | kebab-case | show-filtered.blade.php | ~~showFiltered.blade.php, show_filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | adjective or noun | AuthenticationInterface | ~~Authenticatable, IAuthentication~~
Trait | adjective | Notifiable | ~~NotificationTrait~~

[???? Back to contents](#contents)

### **Use shorter and more readable syntax where possible**

Bad:

```php
$request->session()->get('cart');
$request->input('name');
```

Good:

```php
session('cart');
$request->name;
```

More examples:

Common syntax | Shorter and more readable syntax
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id` (in PHP 8: `$object->relation?->id`)
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[???? Back to contents](#contents)

### **Use IoC container or facades instead of new Class**

new Class syntax creates tight coupling between classes and complicates testing. Use IoC container or facades instead.

Bad:

```php
$user = new User;
$user->create($request->validated());
```

Good:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->validated());
```

[???? Back to contents](#contents)

### **Do not get data from the `.env` file directly**

Pass the data to config files instead and then use the `config()` helper function to use the data in an application.

Bad:

```php
$apiKey = env('API_KEY');
```

Good:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[???? Back to contents](#contents)

### **Store dates in the standard format. Use accessors and mutators to modify date format**

Bad:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Good:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at'];
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[???? Back to contents](#contents)

### **Use Invokable Controller**

Don't put logic in routes file.

Bad:

```php
// Controller
public function home()
{
    $lastUsers = User::getLastUsers();
    return Inertia::render('home', ['lastUsers' => $lastUsers])
}
// route
Route::get('/home', [HomeController::class, 'home'])->name('home')
```

Good:

```php
// Controller
public function __invoke()
{
    $lastUsers = User::getLastUsers();
    return Inertia::render('home', ['lastUsers' => $users])
}
// route
Route::get('/home', HomeController::class)->name('home')
```


[???? Back to contents](#contents)

### **Use resource controller**

Bad:

```php
Route::prefix('/forecasts')->group(function () {
    Route::get('/', [ForecastsController::class, 'index'])->name('forecasts.index');
    Route::post('/', [ForecastsController::class, 'store'])->name('forecasts.store');
    Route::put('/{forecast}', [ForecastsController::class, 'update'])->name('forecasts.update');
    Route::delete('/{forecast}', [ForecastsController::class, 'delete'])->name('forecasts.delete');
});
```

Better:

```php
Route::resource('forecasts', ForecastsController::class)->except(['create', 'show', 'edit']);

```

[Documentation](https://laravel.com/docs/8.x/controllers#actions-handled-by-resource-controller)

[???? Back to contents](#contents)

### Debugging / Performance

Use [telescope](https://laravel.com/docs/8.x/telescope) or [laravel debugbar](https://github.com/barryvdh/laravel-debugbar)

[???? Back to contents](#contents)

### **Consider using View Models**

If you need to prepare the data for the view : [Spatie View Models](https://github.com/spatie/laravel-view-models)

[???? Back to contents](#contents)

### **Consider using Laravel Query Builder package**

If you need to filter or sort your queries. [Laravel Query Builder](https://spatie.be/docs/laravel-query-builder/v3/introduction)

[???? Back to contents](#contents)

### **Other good practices**

Refer to documentation

Avoid using patterns and tools that are alien to Laravel and similar frameworks (i.e. RoR, Django). If you like Symfony (or Spring) approach for building apps, it's a good idea to use these frameworks instead.

Do not override standard framework features to avoid problems related to updating the framework version and many other issues.

Use modern PHP syntax where possible, but don't forget about readability.

Avoid using View Composers and similar tools unless you really know what you're doing. In most cases, there is a better way to solve the problem.

Use latest versions when possible : Laravel, packages and PHP.

[???? Back to contents](#contents)
