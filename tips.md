## Routing
```php
Route::prefix('admin')
->as('admin.') // all names will be prefixed by admin.
->group(function() {
    Route::resource('users', UserController::class);
});
```
