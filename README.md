# Configure Spatie Package With UUID 🚀🚀

Spatie Package is used to assign Roles and Permissions to the Users

## Require Package
```
composer require spatie/laravel-permission

```
## Vendor Publish
```
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
```
## Migrate Database
```
php artisan migrate
```

## Update Permision Migration
```

<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        $teams = config('permission.teams');
        $tableNames = config('permission.table_names');
        $columnNames = config('permission.column_names');
        $pivotRole = $columnNames['role_pivot_key'] ?? 'role_id';
        $pivotPermission = $columnNames['permission_pivot_key'] ?? 'permission_id';

        if (empty($tableNames)) {
            throw new \Exception('Error: config/permission.php not loaded. Run [php artisan config:clear] and try again.');
        }
        if ($teams && empty($columnNames['team_foreign_key'] ?? null)) {
            throw new \Exception('Error: team_foreign_key on config/permission.php not loaded. Run [php artisan config:clear] and try again.');
        }

        Schema::create($tableNames['permissions'], function (Blueprint $table) {
            $table->uuid('uuid')->primary()->unique(); // permission id
            $table->string('name');       // For MySQL 8.0 use string('name', 125);
            $table->string('guard_name'); // For MySQL 8.0 use string('guard_name', 125);
            $table->timestamps();

            $table->unique(['name', 'guard_name']);
        });

        Schema::create($tableNames['roles'], function (Blueprint $table) use ($teams, $columnNames) {
            $table->uuid('uuid')->primary()->unique(); // role id
            if ($teams || config('permission.testing')) { // permission.testing is a fix for sqlite testing
                $table->unsignedBigInteger($columnNames['team_foreign_key'])->nullable();
                $table->index($columnNames['team_foreign_key'], 'roles_team_foreign_key_index');
            }
            $table->string('name');       // For MySQL 8.0 use string('name', 125);
            $table->string('guard_name'); // For MySQL 8.0 use string('guard_name', 125);
            $table->timestamps();
            if ($teams || config('permission.testing')) {
                $table->unique([$columnNames['team_foreign_key'], 'name', 'guard_name']);
            } else {
                $table->unique(['name', 'guard_name']);
            }
        });

        Schema::create($tableNames['model_has_permissions'], function (Blueprint $table) use ($tableNames, $columnNames, $pivotPermission, $teams) {
            $table->uuid($pivotPermission);

            $table->string('model_type');
            $table->uuid($columnNames['model_morph_key']);
            $table->index([$columnNames['model_morph_key'], 'model_type'], 'model_has_permissions_model_id_model_type_index');

            $table->foreign($pivotPermission)
                ->references('uuid') // permission id
                ->on($tableNames['permissions'])
                ->onDelete('cascade');
            if ($teams) {
                $table->unsignedBigInteger($columnNames['team_foreign_key']);
                $table->index($columnNames['team_foreign_key'], 'model_has_permissions_team_foreign_key_index');

                $table->primary(
                    [$columnNames['team_foreign_key'], $pivotPermission, $columnNames['model_morph_key'], 'model_type'],
                    'model_has_permissions_permission_model_type_primary'
                );
            } else {
                $table->primary(
                    [$pivotPermission, $columnNames['model_morph_key'], 'model_type'],
                    'model_has_permissions_permission_model_type_primary'
                );
            }
        });

        Schema::create($tableNames['model_has_roles'], function (Blueprint $table) use ($tableNames, $columnNames, $pivotRole, $teams) {
            $table->uuid($pivotRole);

            $table->string('model_type');
            $table->uuid($columnNames['model_morph_key']);
            $table->index([$columnNames['model_morph_key'], 'model_type'], 'model_has_roles_model_id_model_type_index');

            $table->foreign($pivotRole)
                ->references('uuid') // role id
                ->on($tableNames['roles'])
                ->onDelete('cascade');
            if ($teams) {
                $table->unsignedBigInteger($columnNames['team_foreign_key']);
                $table->index($columnNames['team_foreign_key'], 'model_has_roles_team_foreign_key_index');

                $table->primary(
                    [$columnNames['team_foreign_key'], $pivotRole, $columnNames['model_morph_key'], 'model_type'],
                    'model_has_roles_role_model_type_primary'
                );
            } else {
                $table->primary(
                    [$pivotRole, $columnNames['model_morph_key'], 'model_type'],
                    'model_has_roles_role_model_type_primary'
                );
            }
        });

        Schema::create($tableNames['role_has_permissions'], function (Blueprint $table) use ($tableNames, $pivotRole, $pivotPermission) {
            $table->uuid($pivotPermission);
            $table->uuid($pivotRole);

            $table->foreign($pivotPermission)
                ->references('uuid') // permission id
                ->on($tableNames['permissions'])
                ->onDelete('cascade');

            $table->foreign($pivotRole)
                ->references('uuid') // role id
                ->on($tableNames['roles'])
                ->onDelete('cascade');

            $table->primary([$pivotPermission, $pivotRole], 'role_has_permissions_permission_id_role_id_primary');
        });

        app('cache')
            ->store(config('permission.cache.store') != 'default' ? config('permission.cache.store') : null)
            ->forget(config('permission.cache.key'));
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        $tableNames = config('permission.table_names');

        if (empty($tableNames)) {
            throw new \Exception('Error: config/permission.php not found and defaults could not be merged. Please publish the package configuration before proceeding, or drop the tables manually.');
        }

        Schema::drop($tableNames['role_has_permissions']);
        Schema::drop($tableNames['model_has_roles']);
        Schema::drop($tableNames['model_has_permissions']);
        Schema::drop($tableNames['roles']);
        Schema::drop($tableNames['permissions']);
    }
};

```

## Create Role Model and Permission Model
```
php artisan make:model Role
php artisan make:model Permission
```

## Role Model
```
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Spatie\Permission\Models\Role as SpatieRole;

class Role extends SpatieRole
{
    use HasFactory;
    use HasUuids;
    protected $primaryKey = 'uuid';
}
```

## Permission Model
```
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Spatie\Permission\Models\Permission as SpatiePermission;

class Permission extends SpatiePermission
{
    use HasFactory;
    use HasUuids;
    protected $primaryKey = 'uuid';
}
```

## Permission File in Config
```
<?php

return [

    'models' => [

        'permission' => App\Models\Permission::class,
      
        'role' => App\Models\Role::class,

    ],

    'table_names' => [

        'roles' => 'roles',

        'permissions' => 'permissions',

        'model_has_permissions' => 'model_has_permissions',

        'model_has_roles' => 'model_has_roles',

        'role_has_permissions' => 'role_has_permissions',
    ],

    'column_names' => [
    
        'role_pivot_key' => null, 
        'permission_pivot_key' => null, 

        'model_morph_key' => 'model_uuid',

        'team_foreign_key' => 'team_id',
    ],

    'register_permission_check_method' => true,

    'register_octane_reset_listener' => false,

    'teams' => false,

    'use_passport_client_credentials' => false,

    'display_permission_in_exception' => false,

    'display_role_in_exception' => false,

    'enable_wildcard_permission' => false,


    'cache' => [


        'expiration_time' => \DateInterval::createFromDateString('24 hours'),

        'key' => 'spatie.permission.cache',

        'store' => 'default',
    ],
];

```

## Config Reference Site
```
https://spatie.be/docs/laravel-permission/v6/advanced-usage/uuid
```

## Using Roles in Views
```
<!-- Check if the user has the 'Admin' role -->
@role('Admin')
    <p>This is visible to users with the Admin role.</p>
@endrole

<!-- Alias for @role('Admin') -->
@hasrole('Admin')
    <p>This is visible to users with the Admin role.</p>
@endhasrole

<!-- Check if the user has any of the given roles -->
@hasanyrole('Admin|Manager')
    <p>This is visible to users with either the Admin or Manager role.</p>
@endhasanyrole

<!-- Check if the user has all of the given roles -->
@hasallroles('Admin|Manager')
    <p>This is visible to users with both Admin and Manager roles.</p>
@endhasallroles

<!-- Check if the user has a specific permission -->
@can('manage users')
    <p>This is visible to users with the 'manage users' permission.</p>
@endcan

```
## Using Roles in Helper / Controller
```
use App\Models\User;
use Illuminate\Support\Facades\Auth;

class SomeController extends Controller
{
    public function someMethod()
    {
        $user = Auth::user();

        // Check if the user has the 'Admin' role
        if ($user->hasRole('Admin')) {
            // Do something for Admin users
        }

        // Check if the user has any of the given roles
        if ($user->hasAnyRole(['Admin', 'Manager'])) {
            // Do something for Admin or Manager users
        }

        // Check if the user has all of the given roles
        if ($user->hasAllRoles(['Admin', 'Manager'])) {
            // Do something for users with both Admin and Manager roles
        }

        // Check if the user has a specific permission
        if ($user->can('manage users')) {
            // Do something for users with the 'manage users' permission
        }
    }
}

```

## Using In Routes Middleware

```
protected $routeMiddleware = [
    // ...
    'role' => \Spatie\Permission\Middlewares\RoleMiddleware::class,
    'permission' => \Spatie\Permission\Middlewares\PermissionMiddleware::class,
    'role_or_permission' => \Spatie\Permission\Middlewares\RoleOrPermissionMiddleware::class,
];


Route::group(['middleware' => ['role:Admin']], function () {
    Route::get('/admin', [AdminController::class, 'index']);
});

Route::group(['middleware' => ['permission:manage users']], function () {
    Route::get('/manage-users', [UserController::class, 'index']);
});

Route::group(['middleware' => ['role_or_permission:Admin|manage users']], function () {
    Route::get('/admin-or-manage-users', [AdminUserController::class, 'index']);
});

```

## Example with Seeding

```
// Define roles and permissions for a CRM system
$roles = [
    'Admin' => ['manage users', 'manage customers', 'view customers', 'manage tickets', 'view reports', 'generate reports'],
    'Manager' => ['manage customers', 'view customers', 'view reports', 'generate reports'],
    'Sales Representative' => ['manage customers', 'view customers', 'view reports'],
    'Support Agent' => ['manage tickets', 'view customers'],
    'Viewer' => ['view customers', 'view reports'],
];

// Create permissions
$permissions = array_unique(call_user_func_array('array_merge', array_values($roles)));
foreach ($permissions as $permissionName) {
    Permission::firstOrCreate(['name' => $permissionName]);
}

// Create roles and assign permissions
foreach ($roles as $roleName => $rolePermissions) {
    $role = Role::firstOrCreate(['name' => $roleName]);
    $role->syncPermissions($rolePermissions);
}

// Seed users with random roles
User::factory(100)->create()->each(function ($user) {
    $role = Role::inRandomOrder()->first();
    $user->assignRole($role);
});

```
