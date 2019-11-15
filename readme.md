## Verifying users phone numbers in a Laravel application with Twilio verify

In this tutorial, we will look at how to verify users phone number using Twilio Verify by building  a simple authentication system in Laravel.

[Twilio Verify](https://www.twilio.com/verify) makes it easier to verify user’s phone number to ensure it is a valid phone number by sending SMS short code to the number during registration. This can help reduce fake accounts and failure rate when sending SMS notifications to users.

## Prerequisite

In order to follow this tutorial, you will need:

- Basic knowledge of Laravel
- [Laravel](https://laravel.com/docs/master) Installed on your local machine
- [Composer](https://getcomposer.org/) globally installed
- [Twilio Account](https://www.twilio.com/referral/5PFGwv)

## Getting started

We will start off by creating a new Laravel project using the [Laravel Installer](https://laravel.com/docs/5.8#installation). If you don’t have it installed or prefer to use [Composer](https://laravel.com/docs/6.x/installation), you can check how to do so from the [Laravel documentation](https://laravel.com/docs/master). To generate a fresh Laravel project, run this command in your terminal:

    $ laravel new twilio-phone-verify  

Now change your working directory to `twilio-phone-verify` and install the [Twilio PHP SDK](https://www.twilio.com/docs/libraries/php) via composer:

    $ cd twilio-phone-verify
    $ composer require twilio/sdk 

If you don’t have Composer installed on your composer you can do so by following the instructions [here](https://getcomposer.org/doc/00-intro.md).
Next, we need to get our Twilio credentials from the Twilio dashboard. Head over to your [dashboard](https://www.twilio.com/console) and grab your `account_sid` and `auth_token`.

![](https://paper-attachments.dropbox.com/s_F2A8B2F68E4E7251C0E01BC69920BEB8CE8E4B362D8D4BA952FACA12F8136664_1573635422612_Group+8.png)


Now navigate to the [Verify](https://www.twilio.com/console/verify) section to create a new [Twilio Verify Service](https://www.twilio.com/console/verify/services). Take note of the `sid`  generated for you after creating the Verify service as this will be used for authenticating the instance of the verify sdk. 

![](https://paper-attachments.dropbox.com/s_F2A8B2F68E4E7251C0E01BC69920BEB8CE8E4B362D8D4BA952FACA12F8136664_1573635718713_Group+10.png)


Next we need to update the `.env` file with our Twilio credentials.  Open up `.env` located at the root of the project directory and add these values:

    TWILIO_SID="INSERT YOUR TWILIO SID HERE"
    TWILIO_AUTH_TOKEN="INSERT YOUR TWILIO TOKEN HERE"
    TWILIO_VERIFY_SID="INSERT YOUR TWILIO SYNC SERVICE SID"

### Setting up Database
Next let’s set up our database for our application. We will make use of [MySQL](https://www.mysql.com/) database for our application.  If you use a MySQL client like [phpMyAdmin](https://www.phpmyadmin.net/) to manage your database then go ahead and create a database named `phone-verify` and skip this section if not then install MySQL from the [official site](https://www.mysql.com/downloads/) for your platform of choice. After successful installation, fire up your terminal and run this command to login to MySQL:

    $ mysql -u {your_user_name}

**NOTE:** *Add the `-p` flag if you have a password for your mysql instance.*

Once you are logged in, run the following command to create a new database

    mysql> create database phone-verify;
    mysql> exit;

Next, let’s update our environmental variables with our database credentials. Open up `.env`  and make the following adjustments:

    DB_DATABASE=phone-verify
    DB_USERNAME={your_user_name}
    DB_PASSWORD={password if any}

### Updating User Migration and Model
We have successfully create our database, now  let’s update our `user` [migrations](https://laravel.com/docs/6.x/migrations).  By default Laravel creates a `user` migration and [Model](https://laravel.com/docs/6.x/eloquent) for us when we generate a new project. So we will have to make adjustments to these files to fit our needs.

Now, open up the project folder in your favourite IDE/text editor so we can make changes as needed. Let’s start from updating the needed fields in our User table. Open up the users migration file ( `database/migrations/2014_10_12_000000_create_users_table.php`) and make the following adjustments to the `up()` method:

       public function up()
        {
            Schema::create('users', function (Blueprint $table) {
                $table->bigIncrements('id');
                $table->string('name');
                $table->string('phone_number')->unique();
                $table->boolean('isVerified')->default(false);
                $table->string('password');
                $table->rememberToken();
                $table->timestamps();
            });
        }

We added the `phone_number` and `isVerified` fields for storing user’s phone number and checking is the phone number has been verified respectively.
Now we have added the needed fields for our application, let’s run our migration. Run the following command in the project directory root to add this table to our database:

    $ php artisan migrate

If the file get migrated successfully, we will see the file name (`{time_stamp}_create_users_table`) printed out in the terminal.
Next, let’s update the `[fillable](https://laravel.com/docs/6.x/eloquent#mass-assignment)` properties of the User model to include the `phone_number` and `isVerified` fields. Open up `app/User.php` and make the following changes to the `$fillable` array:

     /**
         * The attributes that are mass assignable.
         *
         * @var array
         */
        protected $fillable = [
            'name', 'email', 'password', 'phone_number', 'isVerified'
        ];


## Implementing Authentication Logic

At this point, we have successfully set up our Laravel project with Twilio SDK and also created our database. Now let’s write out our logic for authenticating a user. We will start off by generating a `AuthController`  which will house all needed logic for each authentication step. Open up a terminal in the project root directory and run the following command to generate a [Controller](https://laravel.com/docs/6.x/controllers):

    $ php artisan make:controller AuthController

The above command will generate a controller class file in `app/Http/Controllers/AuthController.php`.

### Registering Users
Now let’s get down to implementing our first authentication logic. We will start off by implementing the registration logic. 
Let’s for a moment assume we are going to send out SMS notifications to registered users later on from our application, we will need to ensure their phone numbers stored in our database is correct. And there’s no better place to enforce this validation than at the point of registration. To accomplish this, we will make use of [Twilio Verify](https://www.twilio.com/verify) to check if the phone number entered by our user is a valid phone number. 

Let’s get started, now open up `app/Http/Controllers/AuthController.php` and add the following method:

     /**
         * Create a new user instance after a valid registration.
         *
         * @param  array  $data
         * @return \App\User
         */
        protected function create(Request $request)
        {
            $data = $request->validate([
                'name' => ['required', 'string', 'max:255'],
                'phone_number' => ['required', 'numeric', 'unique:users'],
                'password' => ['required', 'string', 'min:8', 'confirmed'],
            ]);
            /* Get credentials from .env */
            $token = getenv("TWILIO_AUTH_TOKEN");
            $twilio_sid = getenv("TWILIO_SID");
            $twilio_verify_sid = getenv("TWILIO_VERIFY_SID");
            $twilio = new Client($twilio_sid, $token);
            $twilio->verify->v2->services($twilio_verify_sid)
                ->verifications
                ->create($data['phone_number'], "sms");
            User::create([
                'name' => $data['name'],
                'phone_number' => $data['phone_number'],
                'password' => Hash::make($data['password']),
            ]);
            return redirect()->route('verify')->with(['phone_number' => $data['phone_number']]);
        }
    

Let’s take a closer look at the code above. After validating the data coming into our function via the `$request` property, we retrieve our Twilio credentials stored in the `.env` file using the built-in PHP [getenv()](http://php.net/manual/en/function.getenv.php) function and pass it into the Twilio Client to create a new instance. After which, we access the `verify` service from the instance of the Twilio client using:

    $twilio->verify->v2->services($twilio_verify_sid)
                ->verifications
                ->create($data['phone_number'], "sms");

We also pass in our Twilio Verify service `sid` to the `service` which allows us access the Twilio Verify service we created earlier in this tutorial.  Next we call the `->verifications->create()` method passing in the phone number to be verified and a channel for delivery the OTP which can be either  `mail`, `sms` or `call`. We are currently making use of the `sms` channel which means we want our OTP code sent to the user via SMS. Next we store our user’s data in the database using the [Eloquent](https://laravel.com/docs/6.x/eloquent) [`create`](https://laravel.com/docs/6.x/eloquent#mass-assignment) method:

    User::create([
                'name' => $data['name'],
                'phone_number' => $data['phone_number'],
                'password' => Hash::make($data['password']),
            ]);

After that we redirect the user to a `verify` page sending their `phone_number`  as data for the view.

### Verifying Phone number OTP
After successful registration of our user, we need to create a way for verifying the OTP sent to them via our `channel` of choice. Let’s create our `verify` method which will be used to verify the users phone number against OTP code entered in your form. Open `app/Http/Controllers/AuthController.php`  and add the following method:

      protected function verify(Request $request)
        {
            $data = $request->validate([
                'verification_code' => ['required', 'numeric'],
                'phone_number' => ['required', 'string'],
            ]);
            /* Get credentials from .env */
            $token = getenv("TWILIO_AUTH_TOKEN");
            $twilio_sid = getenv("TWILIO_SID");
            $twilio_verify_sid = getenv("TWILIO_VERIFY_SID");
            $twilio = new Client($twilio_sid, $token);
            $verification = $twilio->verify->v2->services($twilio_verify_sid)
                ->verificationChecks
                ->create($data['verification_code'], array('to' => $data['phone_number']));
            if ($verification->valid) {
                $user = tap(User::where('phone_number', $data['phone_number']))->update(['isVerified' => true]);
                /* Authenticate user */
                Auth::login($user->first());
                return redirect()->route('home')->with(['message' => 'Phone number verified']);
            }
            return back()->with(['phone_number' => $data['phone_number'], 'error' => 'Invalid verification code entered!']);
        }

Just like in the `register()` method, we first validate the data gotten from the request and also instantiate the Twilio SDK with our credentials before accessing the `verify` service. Let’s take a look at the important aspect here:

    $verification = $twilio->verify->v2->services($twilio_verify_sid)
                ->verificationChecks
                ->create($data['verification_code'], array('to' => $data['phone_number']));

From the above you can tell we are accessing the Twilio verify service as we did earlier but this time we are making use of another method available to us via the service:

    ->verificationChecks->create($data['verification_code'], array('to' => $data['phone_number']));

The `create()` function takes in two parameters, a `string` of the `OTP` code sent to the user and an `array` with a `to` property whose value is the user’s phone number which the OTP was sent to.
The `verificationChecks->create()` returns an object which contains several properties including a boolean property `valid` which is either `true` or `false` depending if the OTP entered is valid or not:

    if ($verification->valid) {
                $user = tap(User::where('phone_number', $data['phone_number']))->update(['isVerified' => true]);
                /* Authenticate user */
                Auth::login($user->first());
                return redirect()->route('home')->with(['message' => 'Phone number verified']);
            }
            return back()->with(['phone_number' => $data['phone_number'], 'error' => 'Invalid verification code entered!']);

 Next we check if the `valid` property is true and then proceed to update the `isVerified` field of the user to `true`.  We then proceed  to [manually authenticate](https://laravel.com/docs/6.x/authentication#other-authentication-methods) the user using Laravel’s `Auth::login`  method which will login and remember the given `User` model instance.
 
 **Note:** *The `User` model must implement the `Authenticatable` interface before it can be used with the Laravel `Auth::login` method.*

After successful verification of the user, we then redirect them to the application dashboard.


## Building The Views

At this point, we have successfully written out our logic for registering and verifying a user. Now let’s build our view which the user will use to interact with our application. First let’s create a [layout](https://laravel.com/docs/6.x/blade#defining-a-layout) which will serve as the main layout of our application. Create a folder named `layouts`  in `resources/views/`, next create a file named `app.blade.php`  in the `layouts` folder. Now open up the just created file (`resources/views/layouts/app.blade.php`) and add the following:

    <!doctype html>
    <html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <!-- CSRF Token -->
        <meta name="csrf-token" content="{{ csrf_token() }}">
        <title>{{ config('app.name', 'Laravel') }}</title>
        <!-- Styles -->
        <link href="https://stackpath.bootstrapcdn.com/bootstrap/4.2.1/css/bootstrap.min.css" rel="stylesheet"
            integrity="sha384-GJzZqFGwb1QTTN6wy59ffF1BuGJpLSa9DkKMp0DgiMDm4iYMj70gZWKYbI706tWS" crossorigin="anonymous">
    </head>
    <body>
        <div id="app">
            <nav class="navbar navbar-expand-md navbar-light bg-white shadow-sm">
                <div class="container">
                    <a class="navbar-brand" href="{{ url('/') }}">
                        {{ config('app.name', 'Laravel') }}
                    </a>
                    <button class="navbar-toggler" type="button" data-toggle="collapse"
                        data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false"
                        aria-label="{{ __('Toggle navigation') }}">
                        <span class="navbar-toggler-icon"></span>
                    </button>
                    <div class="collapse navbar-collapse" id="navbarSupportedContent">
                        <!-- Left Side Of Navbar -->
                        <ul class="navbar-nav mr-auto">
                        </ul>
                        <!-- Right Side Of Navbar -->
                        <ul class="navbar-nav ml-auto">
                            <!-- Authentication Links -->
                            @guest
                            <li class="nav-item">
                                <a class="nav-link" href="{{ route('register') }}">{{ __('Register') }}</a>
                            </li>
                            @else
                            <li class="nav-item dropdown">
                                <a id="navbarDropdown" class="nav-link dropdown-toggle" href="#" role="button"
                                    data-toggle="dropdown" aria-haspopup="true" aria-expanded="false" v-pre>
                                    {{ Auth::user()->name }} <span class="caret"></span>
                                </a>
    
                            </li>
                            @endguest
                        </ul>
                    </div>
                </div>
            </nav>
            <main class="py-4">
                @yield('content')
            </main>
        </div>
    </body>
    </html>
    

***Note:** We are making use of bootstrap for styling our application and forms.*

Next create a folder called `auth` in `resources/views/` now create the following files and paste in their respective content also. In `resources/views/auth/register.blade.php`: 

    @extends('layouts.app')
    @section('content')
    <div class="container">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card">
                    <div class="card-header">{{ __('Register') }}</div>
                    <div class="card-body">
                        <form method="POST" action="{{ route('register') }}">
                            @csrf
                            <div class="form-group row">
                                <label for="name" class="col-md-4 col-form-label text-md-right">{{ __('Name') }}</label>
                                <div class="col-md-6">
                                    <input id="name" type="text" class="form-control @error('name') is-invalid @enderror" name="name" value="{{ old('name') }}" required autocomplete="name" autofocus>
                                    @error('name')
                                        <span class="invalid-feedback" role="alert">
                                            <strong>{{ $message }}</strong>
                                        </span>
                                    @enderror
                                </div>
                            </div>
                            <div class="form-group row">
                                <label for="phone_number" class="col-md-4 col-form-label text-md-right">{{ __('Phone Number') }}</label>
                                <div class="col-md-6">
                                    <input id="phone_number" type="tel" class="form-control @error('phone_number') is-invalid @enderror" name="phone_number" value="{{ old('phone_number') }}" required>
                                    @error('phone_number')
                                        <span class="invalid-feedback" role="alert">
                                            <strong>{{ $message }}</strong>
                                        </span>
                                    @enderror
                                </div>
                            </div>
                            <div class="form-group row">
                                <label for="password" class="col-md-4 col-form-label text-md-right">{{ __('Password') }}</label>
                                <div class="col-md-6">
                                    <input id="password" type="password" class="form-control @error('password') is-invalid @enderror" name="password" required autocomplete="new-password">
                                    @error('password')
                                        <span class="invalid-feedback" role="alert">
                                            <strong>{{ $message }}</strong>
                                        </span>
                                    @enderror
                                </div>
                            </div>
                            <div class="form-group row">
                                <label for="password-confirm" class="col-md-4 col-form-label text-md-right">{{ __('Confirm Password') }}</label>
                                <div class="col-md-6">
                                    <input id="password-confirm" type="password" class="form-control" name="password_confirmation" required autocomplete="new-password">
                                </div>
                            </div>
                            <div class="form-group row mb-0">
                                <div class="col-md-6 offset-md-4">
                                    <button type="submit" class="btn btn-primary">
                                        {{ __('Register') }}
                                    </button>
                                </div>
                            </div>
                        </form>
                    </div>
                </div>
            </div>
        </div>
    </div>
    @endsection
    

and now in `resources/views/auth/verify.blade.php`:

    @extends('layouts.app')
    @section('content')
    <div class="container">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card">
                    <div class="card-header">{{ __('Verify Your Phone Number') }}</div>
                    <div class="card-body">
                        @if (session('error'))
                        <div class="alert alert-danger" role="alert">
                            {{session('error')}}
                        </div>
                        @endif
                        Please enter the OTP sent to your number: {{session('phone_number')}}
                        <form action="{{route('verify')}}" method="post">
                            @csrf
                            <div class="form-group row">
                                <label for="verification_code"
                                    class="col-md-4 col-form-label text-md-right">{{ __('Phone Number') }}</label>
                                <div class="col-md-6">
                                    <input type="hidden" name="phone_number" value="{{session('phone_number')}}">
                                    <input id="verification_code" type="tel"
                                        class="form-control @error('verification_code') is-invalid @enderror"
                                        name="verification_code" value="{{ old('verification_code') }}" required>
                                    @error('verification_code')
                                    <span class="invalid-feedback" role="alert">
                                        <strong>{{ $message }}</strong>
                                    </span>
                                    @enderror
                                </div>
                            </div>
                            <div class="form-group row mb-0">
                                <div class="col-md-6 offset-md-4">
                                    <button type="submit" class="btn btn-primary">
                                        {{ __('Verify Phone Number') }}
                                    </button>
                                </div>
                            </div>
                        </form>
                    </div>
                </div>
            </div>
        </div>
    </div>
    @endsection
    

Lastly, let’s also create a page where verified users will be taken to. Create a file called `home.blade.php` in `resources/views/` and add in the following content (`resources/views/home.blade.php`):

    @extends('layouts.app')
    @section('content')
    <div class="container">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card">
                    <div class="card-header">Dashboard</div>
                    <div class="card-body">
                        @if (session('message'))
                            <div class="alert alert-success" role="alert">
                                {{ session('message') }}
                            </div>
                        @endif
                        You are logged in!
                    </div>
                </div>
            </div>
        </div>
    </div>
    @endsection
    


## Updating Our Routes

Awesome now we are done with our view let’s update our `routes/web.php` file with the needed routes for our application. Open up `routes/web.php` and make the following changes:

    <?php
    /*
    |--------------------------------------------------------------------------
    | Web Routes
    |--------------------------------------------------------------------------
    |
    | Here is where you can register web routes for your application. These
    | routes are loaded by the RouteServiceProvider within a group which
    | contains the "web" middleware group. Now create something great!
    |
     */
    Route::get('/', function () {
        return view('auth.register');
    })->name('register');
    
    Route::get('/verify', function () {
        return view('auth.verify');
    })->name('verify');
    
    Route::get('/home', function () {
        return view('home');
    })->name('home');
    
    Route::post('/', 'AuthController@create')->name('register');
    Route::post('/verify', 'AuthController@verify')->name('verify');
    


## Testing Our Application

Awesome! we are done with building our application, now let’s actually test it out. Open up your terminal and navigate to the project directory and run the following command

    $ php artisan serve

This will serve your Laravel application on a localhost port, normally `8000`. Open up the localhost link printed out after running the command on your browser and you should be greeted with registration page similar to this:

![](https://paper-attachments.dropbox.com/s_F2A8B2F68E4E7251C0E01BC69920BEB8CE8E4B362D8D4BA952FACA12F8136664_1573634680020_Screenshot+from+2019-11-13+09-44-21.png)


Now, go ahead and fill out the form after submitting the form you will be sent a OTP code which you are to use in filling the form in the page you were redirected to. 

![](https://paper-attachments.dropbox.com/s_F2A8B2F68E4E7251C0E01BC69920BEB8CE8E4B362D8D4BA952FACA12F8136664_1573634963714_Peek+2019-11-13+09-48.gif)

## Conclusion

Great! Now you have completed this tutorial, you have learnt how to make use of Twilio Verify Service for validating phone number(s) in a Laravel application. Also, we learnt how to manually authenticate a user in a Laravel application.
If you would like to take a look at the complete source code for this tutorial, you can find both the complete source code  on [Github](https://github.com/thecodearcher/twilio-verify-phone-number-verification).

I’d love to answer any question(s) you might have concerning this tutorial. You can reach me via

- Email: [brian.iyoha@gmail.com](mailto:brian.iyoha@gmail.com)
- Twitter: [thecodearcher](https://twitter.com/thecodearcher)
- GitHub: [thecodearcher](https://github.com/thecodearcher)
