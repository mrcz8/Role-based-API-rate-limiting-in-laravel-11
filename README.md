# Role-Based API Rate Limiting in Laravel 11
Implement granular API rate limiting based on user roles in Laravel 11 applications. Creating custom middleware to enforce specific rate limits for different user role.

### First create a new middleware
```
php artisan make:middleware RoleBasedRateLimiter
```

### Define the Middleware Logic
```
<?php

namespace App\Http\Middleware;

use Illuminate\Support\Facades\RateLimiter;
use Closure;
use Illuminate\Http\Request;

class RoleBasedRateLimiter
{
    public function handle(Request $request, Closure $next)
    {
        $user = $request->user();

        // Check if the user is logged in and has a role
        if ($user && $user->role) {
            // Different rate limits based on roles
            if ($user->role === 'admin') {
                $this->setRateLimit($request, 'admin', 100, 1); // 100 requests per minute for admins
            } else {
                $this->setRateLimit($request, 'user', 60, 1); // 60 requests per minute for regular users
            }
        }

        return $next($request);
    }

    protected function setRateLimit(Request $request, $role, $maxAttempts, $decayMinutes)
    {
        RateLimiter::for($role, function () use ($request, $maxAttempts, $decayMinutes) {
            return Limit::perMinute($maxAttempts)->by(optional($request->user())->id ?: $request->ip());
        });

        // Check if the user has exceeded the rate limit
        if (RateLimiter::tooManyAttempts($role, optional($request->user())->id ?: $request->ip())) {
            return response()->json([
                'message' => 'Too many requests, please slow down.',
            ], 429);
        }
    }
}
```

### Register Middleware
Register your custom middleware in the bootstrap/app.php file.
```
<?php
  
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
  
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
  
        $middleware->alias([
            'role.rate.limit' => \App\Http\Middleware\RoleBasedRateLimiter::class,
        ]);
          
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

###  Apply Middleware to Routes
```
Route::middleware(['auth:sanctum', 'role.rate.limit'])->group(function () {
    Route::get('/admin/dashboard', [AdminController::class, 'dashboard']);
    Route::get('/user/profile', [UserController::class, 'profile']);
});
```
