# RBAC System for Laravel 12

A complete Role-Based Access Control implementation for Laravel 12.

## Table of Contents
1. [Installation & Setup](#installation--setup)
2. [Database Migrations](#database-migrations)
3. [Models](#models)
4. [Middleware](#middleware)
5. [Service Provider](#service-provider)
6. [Usage Examples](#usage-examples)
7. [Blade Directives](#blade-directives)

---

## Installation & Setup

### Step 1: Create Laravel Project
```bash
composer create-project laravel/laravel rbac-app
cd rbac-app
```

### Step 2: Configure Database
Update your `.env` file with database credentials.

---

## Database Migrations

### Migration 1: Create Roles Table
```bash
php artisan make:migration create_roles_table
```

**File: `database/migrations/xxxx_xx_xx_create_roles_table.php`**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('roles', function (Blueprint $table) {
            $table->id();
            $table->string('name')->unique();
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('roles');
    }
};
```

### Migration 2: Create Permissions Table
```bash
php artisan make:migration create_permissions_table
```

**File: `database/migrations/xxxx_xx_xx_create_permissions_table.php`**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('permissions', function (Blueprint $table) {
            $table->id();
            $table->string('name')->unique();
            $table->string('slug')->unique();
            $table->text('description')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('permissions');
    }
};
```

### Migration 3: Create Role-Permission Pivot Table
```bash
php artisan make:migration create_permission_role_table
```

**File: `database/migrations/xxxx_xx_xx_create_permission_role_table.php`**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('permission_role', function (Blueprint $table) {
            $table->id();
            $table->foreignId('permission_id')->constrained()->onDelete('cascade');
            $table->foreignId('role_id')->constrained()->onDelete('cascade');
            $table->timestamps();
            
            $table->unique(['permission_id', 'role_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('permission_role');
    }
};
```

### Migration 4: Create User-Role Pivot Table
```bash
php artisan make:migration create_role_user_table
```

**File: `database/migrations/xxxx_xx_xx_create_role_user_table.php`**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('role_user', function (Blueprint $table) {
            $table->id();
            $table->foreignId('role_id')->constrained()->onDelete('cascade');
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->timestamps();
            
            $table->unique(['role_id', 'user_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('role_user');
    }
};
```

### Migration 5: Create User-Permission Pivot Table (for direct permissions)
```bash
php artisan make:migration create_permission_user_table
```

**File: `database/migrations/xxxx_xx_xx_create_permission_user_table.php`**
```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('permission_user', function (Blueprint $table) {
            $table->id();
            $table->foreignId('permission_id')->constrained()->onDelete('cascade');
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->timestamps();
            
            $table->unique(['permission_id', 'user_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('permission_user');
    }
};
```

### Run Migrations
```bash
php artisan migrate
```

---

## Models

### Model 1: Role Model
```bash
php artisan make:model Role
```

**File: `app/Models/Role.php`**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Role extends Model
{
    protected $fillable = ['name', 'slug', 'description'];

    /**
     * Roles can have many permissions
     */
    public function permissions(): BelongsToMany
    {
        return $this->belongsToMany(Permission::class);
    }

    /**
     * Roles can belong to many users
     */
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }

    /**
     * Check if role has a specific permission
     */
    public function hasPermission(string $permission): bool
    {
        return $this->permissions()->where('slug', $permission)->exists();
    }

    /**
     * Assign permission to role
     */
    public function givePermissionTo(Permission|string $permission): void
    {
        if (is_string($permission)) {
            $permission = Permission::where('slug', $permission)->firstOrFail();
        }
        
        $this->permissions()->syncWithoutDetaching($permission);
    }

    /**
     * Remove permission from role
     */
    public function revokePermissionTo(Permission|string $permission): void
    {
        if (is_string($permission)) {
            $permission = Permission::where('slug', $permission)->firstOrFail();
        }
        
        $this->permissions()->detach($permission);
    }
}
```

### Model 2: Permission Model
```bash
php artisan make:model Permission
```

**File: `app/Models/Permission.php`**
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Permission extends Model
{
    protected $fillable = ['name', 'slug', 'description'];

    /**
     * Permissions can belong to many roles
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }

    /**
     * Permissions can belong to many users (direct permissions)
     */
    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class);
    }
}
```

### Model 3: Update User Model

**File: `app/Models/User.php`**
Add the following trait and methods to your existing User model:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
        ];
    }

    /**
     * Users can have many roles
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class);
    }

    /**
     * Users can have direct permissions
     */
    public function permissions(): BelongsToMany
    {
        return $this->belongsToMany(Permission::class);
    }

    /**
     * Assign role to user
     */
    public function assignRole(Role|string $role): void
    {
        if (is_string($role)) {
            $role = Role::where('slug', $role)->firstOrFail();
        }
        
        $this->roles()->syncWithoutDetaching($role);
    }

    /**
     * Remove role from user
     */
    public function removeRole(Role|string $role): void
    {
        if (is_string($role)) {
            $role = Role::where('slug', $role)->firstOrFail();
        }
        
        $this->roles()->detach($role);
    }

    /**
     * Check if user has a specific role
     */
    public function hasRole(string|array $role): bool
    {
        if (is_array($role)) {
            return $this->roles()->whereIn('slug', $role)->exists();
        }
        
        return $this->roles()->where('slug', $role)->exists();
    }

    /**
     * Check if user has any of the given roles
     */
    public function hasAnyRole(array $roles): bool
    {
        return $this->roles()->whereIn('slug', $roles)->exists();
    }

    /**
     * Check if user has all of the given roles
     */
    public function hasAllRoles(array $roles): bool
    {
        return $this->roles()->whereIn('slug', $roles)->count() === count($roles);
    }

    /**
     * Give direct permission to user
     */
    public function givePermissionTo(Permission|string $permission): void
    {
        if (is_string($permission)) {
            $permission = Permission::where('slug', $permission)->firstOrFail();
        }
        
        $this->permissions()->syncWithoutDetaching($permission);
    }

    /**
     * Revoke direct permission from user
     */
    public function revokePermissionTo(Permission|string $permission): void
    {
        if (is_string($permission)) {
            $permission = Permission::where('slug', $permission)->firstOrFail();
        }
        
        $this->permissions()->detach($permission);
    }

    /**
     * Check if user has a specific permission (through role or direct)
     */
    public function hasPermission(string|array $permission): bool
    {
        if (is_array($permission)) {
            foreach ($permission as $perm) {
                if ($this->hasPermission($perm)) {
                    return true;
                }
            }
            return false;
        }

        // Check direct permissions
        if ($this->permissions()->where('slug', $permission)->exists()) {
            return true;
        }

        // Check role permissions
        return $this->roles()
            ->whereHas('permissions', function ($query) use ($permission) {
                $query->where('slug', $permission);
            })
            ->exists();
    }

    /**
     * Check if user has any of the given permissions
     */
    public function hasAnyPermission(array $permissions): bool
    {
        foreach ($permissions as $permission) {
            if ($this->hasPermission($permission)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Check if user has all of the given permissions
     */
    public function hasAllPermissions(array $permissions): bool
    {
        foreach ($permissions as $permission) {
            if (!$this->hasPermission($permission)) {
                return false;
            }
        }
        return true;
    }

    /**
     * Get all permissions for the user (from roles and direct)
     */
    public function getAllPermissions()
    {
        // Get direct permissions
        $directPermissions = $this->permissions;

        // Get permissions from roles
        $rolePermissions = Permission::whereHas('roles', function ($query) {
            $query->whereIn('role_id', $this->roles->pluck('id'));
        })->get();

        return $directPermissions->merge($rolePermissions)->unique('id');
    }
}
```

---

## Middleware

### Middleware 1: Role Middleware
```bash
php artisan make:middleware RoleMiddleware
```

**File: `app/Http/Middleware/RoleMiddleware.php`**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class RoleMiddleware
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (!$request->user()) {
            abort(401, 'Unauthorized');
        }

        if (!$request->user()->hasAnyRole($roles)) {
            abort(403, 'Forbidden - Insufficient role permissions');
        }

        return $next($request);
    }
}
```

### Middleware 2: Permission Middleware
```bash
php artisan make:middleware PermissionMiddleware
```

**File: `app/Http/Middleware/PermissionMiddleware.php`**
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class PermissionMiddleware
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next, string ...$permissions): Response
    {
        if (!$request->user()) {
            abort(401, 'Unauthorized');
        }

        if (!$request->user()->hasAnyPermission($permissions)) {
            abort(403, 'Forbidden - Insufficient permissions');
        }

        return $next($request);
    }
}
```

### Register Middleware

**File: `bootstrap/app.php`**
```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use App\Http\Middleware\RoleMiddleware;
use App\Http\Middleware\PermissionMiddleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'role' => RoleMiddleware::class,
            'permission' => PermissionMiddleware::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

---

## Service Provider

### Create RBAC Service Provider
```bash
php artisan make:provider RbacServiceProvider
```

**File: `app/Providers/RbacServiceProvider.php`**
```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
use Illuminate\Support\ServiceProvider;

class RbacServiceProvider extends ServiceProvider
{
    /**
     * Register services.
     */
    public function register(): void
    {
        //
    }

    /**
     * Bootstrap services.
     */
    public function boot(): void
    {
        // Blade directive for role checking
        Blade::if('role', function (string $role) {
            return auth()->check() && auth()->user()->hasRole($role);
        });

        // Blade directive for permission checking
        Blade::if('permission', function (string $permission) {
            return auth()->check() && auth()->user()->hasPermission($permission);
        });

        // Blade directive for any role checking
        Blade::if('anyrole', function (array $roles) {
            return auth()->check() && auth()->user()->hasAnyRole($roles);
        });

        // Blade directive for any permission checking
        Blade::if('anypermission', function (array $permissions) {
            return auth()->check() && auth()->user()->hasAnyPermission($permissions);
        });
    }
}
```

### Register the Service Provider

**File: `bootstrap/providers.php`**
```php
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\RbacServiceProvider::class,
];
```

---

## Database Seeder

### Create RBAC Seeder
```bash
php artisan make:seeder RbacSeeder
```

**File: `database/seeders/RbacSeeder.php`**
```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\Role;
use App\Models\Permission;
use App\Models\User;

class RbacSeeder extends Seeder
{
    public function run(): void
    {
        // Create Permissions
        $permissions = [
            ['name' => 'View Users', 'slug' => 'users.view'],
            ['name' => 'Create Users', 'slug' => 'users.create'],
            ['name' => 'Edit Users', 'slug' => 'users.edit'],
            ['name' => 'Delete Users', 'slug' => 'users.delete'],
            ['name' => 'View Posts', 'slug' => 'posts.view'],
            ['name' => 'Create Posts', 'slug' => 'posts.create'],
            ['name' => 'Edit Posts', 'slug' => 'posts.edit'],
            ['name' => 'Delete Posts', 'slug' => 'posts.delete'],
            ['name' => 'Manage Roles', 'slug' => 'roles.manage'],
            ['name' => 'Manage Permissions', 'slug' => 'permissions.manage'],
        ];

        foreach ($permissions as $permission) {
            Permission::create($permission);
        }

        // Create Roles
        $adminRole = Role::create([
            'name' => 'Administrator',
            'slug' => 'admin',
            'description' => 'Full system access'
        ]);

        $editorRole = Role::create([
            'name' => 'Editor',
            'slug' => 'editor',
            'description' => 'Can manage posts'
        ]);

        $userRole = Role::create([
            'name' => 'User',
            'slug' => 'user',
            'description' => 'Regular user with limited access'
        ]);

        // Assign all permissions to admin
        $adminRole->permissions()->attach(Permission::all());

        // Assign post permissions to editor
        $editorRole->permissions()->attach(
            Permission::whereIn('slug', [
                'posts.view',
                'posts.create',
                'posts.edit',
                'posts.delete'
            ])->get()
        );

        // Assign view permissions to user
        $userRole->permissions()->attach(
            Permission::whereIn('slug', [
                'posts.view',
                'users.view'
            ])->get()
        );

        // Create test users
        $admin = User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => bcrypt('password'),
        ]);
        $admin->assignRole('admin');

        $editor = User::create([
            'name' => 'Editor User',
            'email' => 'editor@example.com',
            'password' => bcrypt('password'),
        ]);
        $editor->assignRole('editor');

        $regularUser = User::create([
            'name' => 'Regular User',
            'email' => 'user@example.com',
            'password' => bcrypt('password'),
        ]);
        $regularUser->assignRole('user');
    }
}
```

### Run the Seeder
```bash
php artisan db:seed --class=RbacSeeder
```

---

## Usage Examples

### In Routes

**File: `routes/web.php`**
```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\UserController;
use App\Http\Controllers\PostController;

// Protected by role
Route::middleware(['auth', 'role:admin'])->group(function () {
    Route::get('/admin/dashboard', function () {
        return 'Admin Dashboard';
    });
});

// Protected by permission
Route::middleware(['auth', 'permission:posts.create'])->group(function () {
    Route::get('/posts/create', [PostController::class, 'create']);
    Route::post('/posts', [PostController::class, 'store']);
});

// Protected by multiple roles (any)
Route::middleware(['auth', 'role:admin,editor'])->group(function () {
    Route::get('/content/manage', function () {
        return 'Content Management';
    });
});

// Protected by multiple permissions (any)
Route::middleware(['auth', 'permission:users.edit,users.delete'])->group(function () {
    Route::resource('users', UserController::class)->except(['index', 'show']);
});
```

### In Controllers

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index()
    {
        // Check permission in controller
        if (!auth()->user()->hasPermission('posts.view')) {
            abort(403, 'Unauthorized');
        }

        // Your logic here
    }

    public function create()
    {
        // Using gate
        $this->authorize('create-post');

        // Your logic here
    }

    public function update(Request $request, $id)
    {
        $user = auth()->user();

        // Check if user has permission
        if (!$user->hasPermission('posts.edit')) {
            return response()->json(['error' => 'Unauthorized'], 403);
        }

        // Your logic here
    }
}
```

### Policy Example

```bash
php artisan make:policy PostPolicy
```

**File: `app/Policies/PostPolicy.php`**
```php
<?php

namespace App\Policies;

use App\Models\User;
use App\Models\Post;

class PostPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->hasPermission('posts.view');
    }

    public function view(User $user, Post $post): bool
    {
        return $user->hasPermission('posts.view');
    }

    public function create(User $user): bool
    {
        return $user->hasPermission('posts.create');
    }

    public function update(User $user, Post $post): bool
    {
        return $user->hasPermission('posts.edit');
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->hasPermission('posts.delete');
    }
}
```

---

## Blade Directives

### In Views

**File: `resources/views/dashboard.blade.php`**
```blade
@role('admin')
    <div class="admin-panel">
        <h2>Admin Panel</h2>
        <!-- Admin-only content -->
    </div>
@endrole

@permission('posts.create')
    <a href="{{ route('posts.create') }}" class="btn btn-primary">
        Create New Post
    </a>
@endpermission

@anyrole(['admin', 'editor'])
    <div class="content-management">
        <!-- Content for admins or editors -->
    </div>
@endanyrole

@anypermission(['users.edit', 'users.delete'])
    <div class="user-actions">
        <!-- User management actions -->
    </div>
@endanypermission

@role('admin')
    <p>You are an admin</p>
@else
    <p>You are not an admin</p>
@endrole
```

---

## API Usage Examples

### Assigning Roles and Permissions

```php
// Assign role to user
$user = User::find(1);
$user->assignRole('editor');

// Assign multiple roles
$user->assignRole('admin');
$user->assignRole('moderator');

// Remove role
$user->removeRole('editor');

// Give direct permission to user
$user->givePermissionTo('posts.publish');

// Revoke permission
$user->revokePermissionTo('posts.publish');

// Give permission to role
$role = Role::where('slug', 'editor')->first();
$role->givePermissionTo('posts.edit');

// Revoke permission from role
$role->revokePermissionTo('posts.edit');
```

### Checking Permissions

```php
$user = auth()->user();

// Check single role
if ($user->hasRole('admin')) {
    // User is admin
}

// Check multiple roles (any)
if ($user->hasAnyRole(['admin', 'editor'])) {
    // User is admin OR editor
}

// Check multiple roles (all)
if ($user->hasAllRoles(['admin', 'verified'])) {
    // User has both roles
}

// Check single permission
if ($user->hasPermission('posts.create')) {
    // User can create posts
}

// Check multiple permissions (any)
if ($user->hasAnyPermission(['posts.edit', 'posts.delete'])) {
    // User can edit OR delete posts
}

// Check multiple permissions (all)
if ($user->hasAllPermissions(['posts.create', 'posts.edit'])) {
    // User can both create AND edit posts
}

// Get all user permissions
$permissions = $user->getAllPermissions();
```

---

## Testing

### Create Feature Tests

```bash
php artisan make:test RbacTest
```

**File: `tests/Feature/RbacTest.php`**
```php
<?php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use App\Models\Role;
use App\Models\Permission;
use Illuminate\Foundation\Testing\RefreshDatabase;

class RbacTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_be_assigned_role(): void
    {
        $user = User::factory()->create();
        $role = Role::create(['name' => 'Admin', 'slug' => 'admin']);

        $user->assignRole('admin');

        $this->assertTrue($user->hasRole('admin'));
    }

    public function test_user_has_permission_through_role(): void
    {
        $user = User::factory()->create();
        $role = Role::create(['name' => 'Editor', 'slug' => 'editor']);
        $permission = Permission::create(['name' => 'Edit Posts', 'slug' => 'posts.edit']);

        $role->givePermissionTo($permission);
        $user->assignRole($role);

        $this->assertTrue($user->hasPermission('posts.edit'));
    }

    public function test_middleware_blocks_unauthorized_user(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->get('/admin/dashboard');

        $response->assertStatus(403);
    }

    public function test_middleware_allows_authorized_user(): void
    {
        $user = User::factory()->create();
        $role = Role::create(['name' => 'Admin', 'slug' => 'admin']);
        $user->assignRole($role);

        $response = $this->actingAs($user)
            ->get('/admin/dashboard');

        $response->assertStatus(200);
    }
}
```

---

## Advanced Features

### 1. Role Hierarchy

Add a `level` column to roles table and implement hierarchy checking:

```php
// In Role model
public function hasHigherLevelThan(Role $role): bool
{
    return $this->level > $role->level;
}
```

### 2. Temporary Permissions

Add `expires_at` to permission_user table for temporary access:

```php
// In User model
public function giveTemporaryPermissionTo(Permission $permission, $expiresAt): void
{
    $this->permissions()->attach($permission, ['expires_at' => $expiresAt]);
}
```

### 3. Cache Permissions

Improve performance by caching user permissions:

```php
// In User model
public function getCachedPermissions()
{
    return cache()->remember("user.{$this->id}.permissions", 3600, function () {
        return $this->getAllPermissions();
    });
}
```

---

## Summary

This RBAC system provides:
- ✅ Roles and Permissions management
- ✅ Many-to-Many relationships between Users, Roles, and Permissions
- ✅ Direct user permissions (bypass roles)
- ✅ Middleware protection for routes
- ✅ Blade directives for views
- ✅ Policy integration
- ✅ Flexible permission checking
- ✅ Database seeder for initial setup
- ✅ Comprehensive test coverage

You can extend this system with features like role hierarchy, temporary permissions, permission groups, and more based on your needs.
