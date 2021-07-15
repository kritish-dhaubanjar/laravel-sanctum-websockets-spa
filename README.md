#### Laravel Auth [SPA]

```
composer require laravel/ui

php artisan ui:auth
```
#### Laravel Sanctum
```
composer require laravel/sanctum

php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

php artisan migrate
```

Add Sanctum's middleware to your api middleware group within your `app/Http/Kernel.php` file:

```php

'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:60,1',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

.env
```
...
SESSION_DRIVER=cookie
...
SANCTUM_STATEFUL_DOMAINS=localhost,localhost:8000,localhost:8080,127.0.0.1,127.0.0.1:8000,127.0.0.1:8080,::1
```

##### Create a User
```
php artisan tinker

App\User::create(['name'=>'John Doe', 'email'=>'johndoe@example.org', 'password'=>Hash::make('secret')]);
```

##### Configurations

In `routes/api.php`

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

In `config/cors.php`

```php
return [
    'paths' => ['*'],
    'allowed_methods' => ['*'],
    'allowed_origins' => ['*'],
    'allowed_origins_patterns' => [],
    'allowed_headers' => ['*'],
    'exposed_headers' => [],
    'max_age' => 0,
    'supports_credentials' => true,
];
```



### Laravel Websockets

```
composer require beyondcode/laravel-websockets

php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="migrations"

php artisan migrate

php artisan vendor:publish --provider="BeyondCode\LaravelWebSockets\WebSocketsServiceProvider" --tag="config"
```

#### Pusher
```
composer require pusher/pusher-php-server "~3.0"
```

`.env`
```
BROADCAST_DRIVER=pusher
PUSHER_APP_KEY=s3cr3t
PUSHER_APP_SECRET=s3cr3t
PUSHER_APP_ID=local
PUSHER_APP_CLUSTER=mt1
```

`config/broadcasting.php`
```php
'pusher' => [
    'driver' => 'pusher',
    'key' => env('PUSHER_APP_KEY'),
    'secret' => env('PUSHER_APP_SECRET'),
    'app_id' => env('PUSHER_APP_ID'),
    'options' => [
        'cluster' => env('PUSHER_APP_CLUSTER'),
        'encrypted' => true,
        'host' => '127.0.0.1',
        'port' => 6001,
        'scheme' => 'http'
    ],
],
```

`config/app.php` uncomment
```php
App\Providers\BroadcastServiceProvider::class
```

#### Dashboard
`http://127.0.0.1:8000/laravel-websockets`

#### Event & Channel
```
php artisan make:event Chat
```

```php
class Chat implements ShouldBroadcast{

public $payload = 'Hello World!';
...
public function broadcastOn()
    {
        return new PrivateChannel('App.User.1');
    }
    
public function broadcastAs()
    {
        return 'new-message-event';
    }
...
}

```

In `routes/channels.php`
```php
Broadcast::channel('App.User.{id}', function ($user, $id) {
    //Check User's Authorization to listen on the channel.
    return true;
}
```


### Authorizing Private Broadcast Channels

You should place the `Broadcast::routes` method call within your `routes/api.php` file:

`Broadcast::routes(['middleware' => ['auth:sanctum']]);`

### Running Laravel & Websocket Server
```
php artisan serve
php artisan websockets:serve
```

### UI
```
npm i laravel-echo pusher-js axios
```

```js
<script>
import axios from "axios";
import Pusher from "pusher-js";
import Echo from "laravel-echo";

console.log(Pusher);

export default {
  name: "App",

  data() {
    return {
      data: {
        email: "johndoe@example.org",
        password: "secret",
      },
    };
  },

  async mounted() {
    axios.defaults.withCredentials = true;

    await axios.get("http://localhost:8000/sanctum/csrf-cookie");

    await axios.post("http://localhost:8000/login", this.data);

    const { data } = await axios.get("http://localhost:8000/api/user");

    let echo = new Echo({
      broadcaster: "pusher",
      key: "s3cr3t",
      wsHost: "localhost",
      wsPort: 6001,
      forceTLS: false,
      cluster: "mt1",
      disableStats: true,
      authorizer: (channel, options) => {
        console.log(options);
        return {
          authorize: (socketId, callback) => {
            axios({
              method: "POST",
              url: "http://localhost:8000/api/broadcasting/auth",
              data: {
                socket_id: socketId,
                channel_name: channel.name,
              },
            })
              .then((response) => {
                callback(false, response.data);
              })
              .catch((error) => {
                callback(true, error);
              });
          },
        };
      },
    });

    echo
      .private(`App.Models.User.${data.id}`)
      .listen(".new-message-event", (message) => {
        console.log(message);
      });
  },
};
</script>
```

### Fire an Event
```
php artisan tinker
event(new App\Events\Chat())
```
