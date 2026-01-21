# laravel-allow-active-user-to-login-with-laravel-breeze
In this will look on the code on how to make active users to be able to login in the laravel project using laravel breeze
- will first  create a new laravel project in the machine and make it more conviniet to be used any where

# Why Laravel?

There are a variety of tools and frameworks available to you when building a web application. However, we believe Laravel is the best choice for building modern, full-stack web applications.
run the commad
### $ composer create-project laravel/laravel laravel-active-users
then cd to the $ cd laravel-active-users

#### Install laravel breeze using the command below with is the built in authenification system. that will allow user to login and register also be able to have access to the auth routes and if not auth user will not be able to access those pages. via middleware! thus the breeze

# Laravel Breeze

Laravel Breeze is a minimal, simple implementation of all of Laravel's authentication features, including login, registration, password reset, email verification, and password confirmation. In addition, Breeze includes a simple "profile" page where the user may update their name, email address, and password.

## composer require laravel/breeze --dev
## php artisan breeze:install


```markdown
No security vulnerability advisories found.
Using version ^2.3 for laravel/breeze
@premy-punchu ➜ /workspaces/laravel-allow-active-user-to-login-with-laravel-breeze/laravel-active-users (main) $ php artisan breeze:install
Xdebug: [Step Debug] Could not connect to debugging client. Tried: localhost:9003 (through xdebug.client_host/xdebug.client_port).

 ┌ Which Breeze stack would you like to install? ───────────────┐
 │ Blade with Alpine                                            │
 └──────────────────────────────────────────────────────────────┘

 ┌ Would you like dark mode support? ───────────────────────────┐
 │ No                                                           │
 └──────────────────────────────────────────────────────────────┘

 ┌ Which testing framework do you prefer? ──────────────────────┐
 │ PHPUnit                                                      │
 └──────────────────────────────────────────────────────────────┘

   INFO  Installing and building Node dependencies.
```



   # .env 
   ### go to the .env file and perfrom database configations by defaults comes with sqqlite

`DB_CONNECTION=sqlite
#DB_HOST=127.0.0.1
#DB_PORT=3306
#DB_DATABASE=laravel
#DB_USERNAME=root
#DB_PASSWORD=`

# Database table 
  ## laravel-active-users/database/migrations/0001_01_01_000000_create_users_table.php
  add the following line of code and then run migrations 
   
```markdown
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps();
        });
```
##### $table->boolean('is_active')->default(true);
 look like 

```markdown
         Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->boolean('is_active')->default(true);
            $table->rememberToken();
            $table->timestamps();
        });
```
Then run migrations of the database. but after migration user by defaults his active will be as true 
meaning 
```php
$user->is_active === 1;
where 
$user = Auth::user();
```

# lets see the magic under LoginRequest

## original code 
```php
<?php

namespace App\Http\Requests\Auth;

use Illuminate\Auth\Events\Lockout;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

class LoginRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'email' => ['required', 'string', 'email'],
            'password' => ['required', 'string'],
        ];
    }

    /**
     * Attempt to authenticate the request's credentials.
     *
     * @throws \Illuminate\Validation\ValidationException
     */
    public function authenticate(): void
    {
        $this->ensureIsNotRateLimited();

        if (! Auth::attempt($this->only('email', 'password'), $this->boolean('remember'))) {
            RateLimiter::hit($this->throttleKey());

            throw ValidationException::withMessages([
                'email' => trans('auth.failed'),
            ]);
        }

        RateLimiter::clear($this->throttleKey());
    }

    /**
     * Ensure the login request is not rate limited.
     *
     * @throws \Illuminate\Validation\ValidationException
     */
    public function ensureIsNotRateLimited(): void
    {
        if (! RateLimiter::tooManyAttempts($this->throttleKey(), 5)) {
            return;
        }

        event(new Lockout($this));

        $seconds = RateLimiter::availableIn($this->throttleKey());

        throw ValidationException::withMessages([
            'email' => trans('auth.throttle', [
                'seconds' => $seconds,
                'minutes' => ceil($seconds / 60),
            ]),
        ]);
    }

    /**
     * Get the rate limiting throttle key for the request.
     */
    public function throttleKey(): string
    {
        return Str::transliterate(Str::lower($this->string('email')).'|'.$this->ip());
    }
}

```

### change the authenticate() function to the following code and see the magic

```php
public function authenticate(): void
{
    $this->ensureIsNotRateLimited();

    if (! Auth::attempt(
        $this->only('email', 'password'),
        $this->boolean('remember')
    )) {
        RateLimiter::hit($this->throttleKey());

        throw ValidationException::withMessages([
            'email' => trans('auth.failed'),
        ]);
    }

    $user = Auth::user(); 

    if ($user->is_active === 0) {
        Auth::logout(); 

        throw ValidationException::withMessages([
            'email' => 'Your account is inactive. Please contact the administrator.',
        ]);
    }

    RateLimiter::clear($this->throttleKey());
}
```


