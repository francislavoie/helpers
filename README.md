# Miscellaneous Helpers for PHP and Laravel

[![Build Status](https://travis-ci.org/halaei/helpers.svg)](https://travis-ci.org/halaei/helpers)
[![Latest Stable Version](https://poser.pugx.org/halaei/helpers/v/stable)](https://packagist.org/packages/halaei/helpers)
[![Total Downloads](https://poser.pugx.org/halaei/helpers/downloads)](https://packagist.org/packages/halaei/helpers)
[![Latest Unstable Version](https://poser.pugx.org/halaei/helpers/v/unstable)](https://packagist.org/packages/halaei/helpers)
[![License](https://poser.pugx.org/halaei/helpers/license)](https://packagist.org/packages/halaei/helpers)

## About this Package
This is a personal package of small pieces of codes that I have needed in one specific project once, and they may be needed in other projects later on, too.
So this is a way for me to stop copy and paste.

### Supervisor
Supervisor helps you with running a command in an loop, enabling you to monitor and control it.
Supervisor is a generalization of the laravel daemon queue implementation.
- Supervisor prevents the command from consuming too much memory, getting frozen or taking too long.
- Supervisor gracefully stops the loop of executing the given command if:
    - `artisan queue:restart` command is issued,
    - or memory limit is reached,
    - or `SIGTERM` signal is received.
- Supervisor pauses the loop when:
    - `artisan down` command is issued,
    - or`SIGUSR2` signal is received.
- Supervisor is highly configuration via event listeners:
    - If a listener to `LoopBeginning` event returns false, the supervisor pauses the loop.
    - If a listener to `LoopCompleting` event returns false, the supervisor stops and terminates the process.
    - Extra management and monitoring power is achievable by listening to `RunSucceed`, `RunFailed`, and `SupervisorStopping` events.
```php
use Halaei\Helpers\Supervisor\Supervisor;

app(Supervisor::class)->supervise(GetTelegramUpdates::class);

class GetTelegramUpdates
{
    function handle(Api $api)
    {
        $updates = $api->getUpdates(...);
        $this->queueUpdates($updates);
    }
    function queueUpdates($updates)
    {
        ...
    }
}
```

To configure the behaviour of supervisor, you can pass an instance of `Halaei\Helpers\Supervisor\SupervisorOptions` as the second argument to the `supervise()` method.

### Data Object
An instance of `DataObject` is an object-oriented representation of a key-value array.
The constructor of a data object accepts a key-value array and convert its items into types that are defined in the class `relations()` method.
Magic methods are defined to make properties of a data-object accessible via

* `$object->some_property` properties,
* `$object->getSomeProperty()` getter methods,
* and `$object->setSomeProperty('new value')` setter methods.

Data objects are Arrayable, Jsonable, and Rawable, having `toArray()`, `toJson()` and `toRaw()` methods.

```php
use Halaei\Helpers\Objects\DataObject;

class Order extends DataObject
{
    public static function relations()
    {
        return [
            'items' => [Item::class], // items is a collection of objects of type Item.
            'destination' => Location::class, //delivered_to is an object of type Location
            'customer_mobile' => [Mobile::class, 'decode'], // customer mobile is an string that can be casted to a Mobile object via 'decode' static function.
        ];
    }
}

/**
 * @property string $item_code
 * @property int $quantity
 * @property float $unit_price
 * @property float $total_price
 */
class Item extends DataObject
{
}

/**
 * @property float $lat
 * @property float $lon
 */
class Location extends DataObject
{
}

class Mobile extends DataObject
{
    public static function decode($str)
    {
        if (preg_match('/^\+(\d+)-(\d+)$/', $str, $parts)) {
            return new self(['code' => $parts[1], 'number' => $parts[2]]);
        }
    }

    public function toRaw()
    {
        return '+'.$this->code.'-'.$this->number;
    }
}

$array = [
    'id' => 1234,
    'items' => [
        [
            'item_code' => '#100',
            'quantity' => 5,
            'unit_price' => 24,
            'total_price' => 120,
        ],
        [
            'item_code' => '#200',
            'quantity' => 1,
            'unit_price' => 80,
            'total_price' => 80,
        ],
    ],
    'final_price' => 200,
    'delivered_to' => [
        'lat' => 37.74123543,
        'lon' => 49.43254355,
    ],
    'customer_mobile' => '+98-9131231212',
];
$order = new Order($array);
echo get_class($order->delivered_to); // Location
echo $order->items[0]->quantity;// 5
var_dump($order->toArray()['mobile_number']); // ['code' => '+98', 'number' => '9131231212']
var_dump($order->toRaw()['mobile_number']); // +98-9131231212
```

### Eloquent Cache
The `EloquentCache` class is a key-value repository for your eloquent models with caching features.
If you want to cache your models using this repository, you may optionally let your model implement the `Cacheable` interface.
The implementation of `Cacheable` is also available via `CacheableTrait`.
Features include:
1. Caching find by id queries via `EloquentCache::find()` method.
2. Caching find by secondary key queries via `EloquentCache::findBySecondaryKey()` method.
3. Invalidating the cache for a specific model via `EloquentCache::invalidateCache`, so that other processes don't use cache for that model.
4. Cleaning the cache when updating and deleting the model via `EloquentCache::update()` and `EloquentCache::delete()` methods.


### Eloquent Batch Update & Insert Ignore
When performance does matter, it is important to update multiple rows of a table at once using a single `update` query.
To do so, use the `Collection::update` macro:

```php
$newSensorData = [
    12 => ['value' => 32, 'observed_at' => '2016-06-30 12:30:01'],
    13 => ['value' => 33, 'observed_at' => '2016-06-30 12:30:05'],
    16 => ['value' => 30, 'observed_at' => '2016-06-30 12:30:05'],
];
$sensors = Sensor::whereIn('id', array_keys($newSensorData))->get();
foreach($sensors as $sensor) {
    //some calculations then save the new values
    $sensor->value = $newSensorData[$sensor->id]['value'];
    $sensor->observed_at = $newSensorData[$sensor->id]['observed_at'];
}
$sensors->update();
```

The resulting SQL statement (assuming that the old value for sensor#12 is 32):

```sql
UPDATE `sensors` SET
`value` = CASE 
    WHEN `id` = 13 THEN 33
    WHEN `id` = 16 THEN 30
ELSE `value` END,
`observed_at` = CASE
    WHEN `id` = 12 THEN  '2016-06-30 12:30:01'
    WHEN `id` = 13 THEN '2016-06-30 12:30:05'
    WHEN `id` = 16 THEN '2016-06-30 12:30:05'
ELSE `observed_at` END
WHERE id in (12, 13, 16)
```

To enable batch update feature (+ insert ignore) register `\Halaei\Helpers\Eloquent\EloquentServiceProvider::class` in the list of providers in `config/app.php`.

### Redis-Based mutual exclusive lock
The `\Halaei\Helpers\Redis\Lock` class provides Redis-Based mutual exclusive locks with auto-release timer. The implementation is based on `rpoplpush` and `brpoplpush` Redis commands.

A process that requires a lock should call the `lock($name, $tr = 2)` method in a loop until it returns true;
where `$name` is the name of the lock and `$tr` is the auto-release timer in seconds (with milliseconds resolution).
If the lock is on-hold, the `lock` method waits for at most `$tr` seconds, until the lock is released.

When a process is done it must immediately release the lock by calling the `unlock($name)` method, with the same `$name` parameter used for calling the `lock` method.
If the process holds a lock for more that `$tr` seconds, it will be regarded as a failed process and the lock will be automatically released.

Note: Callers to the `lock` may not be aware of the release of the lock, if the lock was auto-released after the call to the `lock` method. So they should call `lock` once more.

```php
use \Halaei\Helpers\Redis\Lock;

$lock = app(Lock::class);

...
// 1. Wait for lock named 'critical_section'
while (! $lock->lock('critical_section', 0.1) {}
// 2. Do some critical job
//...
// 3. Release the lock
$lock->unlock('critical_section', 0.1);
```

### Clean-up DB transactions between handling queued jobs
In order to make sure there is nothing wrong with the default DB connection even after a messed-up handling of a queued job,
call `Halaei\Helpers\Listeners\RefreshDBConnections::boot()` in `AppServiceProvider::boot()` function.

### Safely terminate long-running workers
Long-running workers may cause unexpected issues if you are not 100% careful.
To safely terminate long-running workers call `Halaei\Helpers\Listeners\RandomWorkerTerminator::boot()` in `AppServiceProvider::boot()`.

### Disabling blade @parent
@parent feature of Laravel framework is implemented with a [minor security bug](https://github.com/laravel/framework/issues/10068).
So you may need to disable the feature. If in your `config/app.php` file relplace `\Illuminate\View\ViewServiceProvider::class` with `\Halaei\Helpers\View\ViewServiceProvider::class`.

### Number Encryption

If you need to obfuscate auto-increment numbers into random-looking strings, use the `Numcrypt` class:

```php
use Halaei\Helpers\Crypt\NumCrypt;

$crypt = new NumCrypt();
echo $crypt->encrypt(36); // 53k7hx
echo $crypt->decrypt('53k7hx'); // 36
```

The NumCrypt constructor accepts charset for the accepted characters in the output and a key for encryption (uxing XOR).
By default the charset is `[a-z0-9]`, and the key is 308312529.

**Note**: NumCrypt is not meant to be cryptographically secure.

```php
use Halaei\Helpers\Crypt\NumCrypt;

$crypt = new NumCrypt('9876543210abcdef', 0 /*dont XOR*/);
echo $crypt->encrypt(16, 0 /*no padding*/); // 89
echo $crypt->decrypt('999989'); // 36
```

## License
This package is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT)
