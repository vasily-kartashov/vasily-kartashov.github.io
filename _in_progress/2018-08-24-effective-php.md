// ---
// layout: post
// title: Effective PHP
// ---

This is a short collection of few things that I use to rectify some especially fun features of PHP. This post will be updated but original structure is inspired by Effective Java.

# Creating and Destroying Object

## Consider static factory methods instead of constructors.

Two things about PHP constructor that worth fixing with static factory methods. Firstly, `new` operator has lower precedence than `->` so we cannot write `new Builder()->withEntity(...)`, and it needs to be `(new Builder())->withEntity(...)`. Secondly, there's no method overloading in PHP, so if you have 2 different ways to initialize an object you either end up with some "union" signature of `__construct`, or pass obscure arrays of configuration options.

Take for example class `SpeechOutput` which can be either pure text, or SSML + the type of speech. Here are the two pretty standard ways to implement it:

```
class SpeechOutput1
{
    ...

    public function __construct(string $type, string $content)
    {
        if ($type == 'text') {
            $this->type = $type;
            $this->text = $content;
        } elseif ($type == 'ssml') {
            $this->type = $type;
            if (!$this->validSsml($ssm)) {
                throw new IllegalArgumentException(...);
            }
            $this->ssml = $ssml;
        } else {
            throw new IllegalArgumentException(...);
        }
    }
}

class SpeechOutput2
{
    ...

    public function __construct(array $options)
    {
        if ($options['type'] == 'text') {
            $this->type = $options['type'];
            $this->text = $options['text'];
        } elseif ($options['type'] == 'ssml') {
            $this->type = $options['type'];
            if (!$this->validSsml($options['ssml'])) {
                throw new IllegalArgumentException(...);
            }
            $this->ssml = $options['ssml'];
        } else {
            throw new IllegalArgumentException(...);
        }
    }
}
```

And don't pretend you havent seen code like this. Both are bad. We don't know what `type`s are avaialble, how the options gonna look like, and which types of content will be accepted. Static factory methods help a lot.

```
class SpeechOuptut3
{
    ...

    private function __construct()
    {
    }

    public static function plainText(string $text)
    {
        $speech = new self;
        $speech->type = 'text';
        $speech->text = $text;
        return $speech;
    }

    public static function ssml(string $ssml)
    {
        $speech = new self;
        $speech->type = 'ssml';
        if (!self::validSsml($ssml)) {
            throw new IllegalArgumentException(...);
        }
        $speech->ssml = $ssml;
        return $speech;
    }
}
```

## Consider a builder when faced with many constructor parameters

PHP makes it somewhat hard to create builders because it lacks inner classes and package visibility. One way around it is to use anonymous constructor replacement. Consider `Downloader` component that needs a `Task` object to tell it what to do. Typical content of a `Task` object would be cacheing policy, batch sizes, response validator, retry policy. Not all of it is needs all the time, and the number of parameters is quite long, so it would make perfect sense to have a `TaskBuilder`. So this is how it goes:

```
class Task
{
    private $cacheKeyPrefix;
    private $timeToLive;
    ...

    private function __construct()
    {
    }

    ...

    public static function builder(): TaskBuilder
    {
        $taks = new Task();
        $constructor = function ($cacheKeyPrefix, $timeToLive) use ($task) {
            $task->cacheKeyPrefix = $cacheKeyPrefix;
            $task->timeToLive = $timeToLive;
            ...

            return $task;
        }
        return new class($constructor) implements TaskBuilder
        {
            private $constructor;
            private $cacheKeyPrefix;
            private $timeToLive;

            public function __construct(callable $constructor)
            {
                $this->constructor = $constructor;
            }

            public function withCache(string $cacheKeyPrefix, int $timeToLive): TaskBuilder
            {
                if ($timeToLive < 0) {
                    throw new InvalidArgumentException('TTL must be non-negative');
                }
                $this->cacheKeyPrefix = $cacheKeyPrefix;
                $this->timeToLive = $timeToLive;
                return $this;
            }

            ...

            public function build(): Task
            {
                return ($this->constructor)(
                    $this->cacheKeyPrefix,
                    $this->timeToLive
                );
            }
        };
    }
}

interface TaskBuilder
{
    public function withCache(string $cacheKeyPrefix, int $timeToLive): TaskBuilder;
}
```

It's not as elegant as a standard Java implementation, but it's close to be airtight.

- The only way to create a task if though a builder,
- The builder can only be created by calling `Task::builder()`
- The resulting `Task` object is immutable

Unpleasantly redundant function `$constructor` is the necessary evil capturing constructed object and returning it.

## Enforce singleton with a static property

There's one very simple way to create singletons in PHP.

```
class Environment
{
    pulic function database() {
        static $database;
        if (!isset($database)) {
            ...
        }
        return $database;
    }
}
```

Please note, this is going to be a real singleton, and will be shared across all instances of class `Envrionment`. If you don't want this, one can use the following simple pattern.

```
class Environment
{
    private $scope = [];

    public static function database(): Database
    {
        return $this->register(Database::class, function () {
            return new Database();
        });
    }

    private function register(string $key, callable $generator)
    {
        if (!isset($this->scope[$key])) {
            $this->scope[$key] = $generator();
        }
        return $this->scope[$key];
    }
}
```

## Enforce noninstantiability with a private constructor


The following class cannot be instantiated and can be used for a collection common function

```
class TestHelper
{
    private function __construct()
    {
    }

    public function randomName(): string
    {
        ...
    }
}
```

## Use namespaces and functions instead of static class methods

Big advantage of the general archaics of PHP design is that thing have survived that used to be OK, than less fashinable, then OK again. PHP never banned free standing functions, and with advent of namespaces we can now easily have proper global functions without clashing namespaces and without creating non-instantiatable classes.

```
namespace Project;

function distance(
    float $latitude1,
    float $longitude1,
    float $latitude2,
    float $longitude2
): float {
    ...
}


\Project\distance(112.0, 53.9, -23.4, 2.3);
```

The main disadvantage of this solution is that these functions cannot be use with autoloader and need to be loaded on every request. If this is not acceptable there's always a java like solution by using static methods in a non-instantiatable class.

```
class Geometry
{
    private function __construct() {}

    public function distance(
        float $latitude1,
        float $longitude1,
        float $latitude2,
        float $longitude2
    ): float {
        ...
    }
}
```



















# Methods common to all object

## Implement __toString

Less used, known feature is implementing magic `__toString` method, that creates a string value of specific object. To illustrate consider the following

```
> php -a
Interactive shell

php > class User { public $name; }
php > $user = new User;
php > $user->name = 'Vasily';
php > echo $user;
PHP Catchable fatal error:  Object of class User could not be converted to string in php shell code on line 1

Catchable fatal error: Object of class User could not be converted to string in php shell code on line 1
```

Whereas implementing `__toString()` produces proper string representation:
```
php > class User2 { public $name; public function __toString() { return $this->name; } }
php > $user2 = new User2;
php > $user2->name = 'Vasily';
php > echo $user2;
Vasily
```

More information can be found under [php.net/manual](http://php.net/manual/en/language.oop5.magic.php#object.tostring)

## Implement JsonSerializable

Oddly unpaired interface lets you define the shape of your object during JSON serialization. Paired with factory methods it creates a nice way to map an object into a structured string representation and back

```
class UpdateTask implements JsonSerializable
{
    private $stationId;
    private $time;

    private function __construct() {}

    public static function create(int $stationId, int $time): UpdateTask {...}

    public static function createFromJson(string $json): UpdateTask
    {
        $data = json_decode($json, true);
        $stationId = $json['station_id'] ?? null;
        $time = $json['time'] ?? null;
        if (!is_int($stationId) || $stationId <= 0) {
            throw new IllegalArgumentException();
        }
        $currentTime = time();
        if ($time + 3600 < $currentTime || $currentTime < $time) {
            throw new IllegalArgumentException();
        }
        $task = new UpdateTask;
        $task->stationId = $stationId;
        $task->time = $time;
    }

    public function __jsonSerialize()
    {
        return [
            'station_id' => $this->stationId,
            'time' => $this->time
        ];
    }

    public function stationId(): int
    {
        return $this->stationId;
    }

    public function time(): DateTime
    {
        return new DateTime('@' . $this->time);
    }
}
```

## Learn to use clone

Clone creates a shallow copy of a PHP object. It's very often used to create a immutable objects that fork off a new object every time there's a modification. Let's assume there's a task object that can be delayed by adding time to an internal timestamp. The most robust and type safe (but not the most beautiful) would be

```
class ForecastUpdateTaks
{
    private $stationId, $location, $timeHorizon, $time;

    public function delay(): ForecastUpdateTask
    {
        $copy = clone $this;
        $copy->time += 3600;
        return $copy;
    }
}
```

This is not changing the original object, rubust with regards to adding new properties to class `ForecastUpdateTask`, but it's quite easy to mess up. The `clone` only creates a very shallow copies, thus if your class contains a reference to an object, then the reference will be shared between your original object and the clone. For more information see [here](http://php.net/manual/en/language.oop5.cloning.php#object.clone)














# Classes and Interfaces

## Minimize the accessibility of classes and members

The visibility modifiers in PHP are `private`, `protected` and `public`, and ommitting one is equivivlent to `public`. The last point is just an artifact of backwards compatibility. We should always start with the lowest accessibility modifier. The smaller the interface of a method the lower is the coupling

## In public classes, use accessor methods, not public fields

Using accessor methods allows you to add exceptions, validators, formattings, assertions, etc. Moreover methods are part of the interface. Compare the following implementation of the function `delete1`

```
function delete1(Identifiable $object) {
    $this->storage->remove($object->id());
}

class User implements Indentifiable {
    private $id;

    public function id() {
        return $this->id;
    }
}
```

this is a type safe and extensible alternative to following two

```
function delete2(User $user) {
    $this->storage->remove($user->id);
}

function delete3($object) {
    $this->storage->remove($object->id);
}
```

## Minimize mutability

Mutating an object makes is "more risky" to pass an object into a function, which might decide to mutate it. See the following example

```
function twoDaysAhead(DateTime $date): int {
    $date->add(new DateInterval('PT48H'));
    return $date->getTimestamp();
}
```

This function modifies the argument passed into it, and there's no syntactic way to even hint the user of the function that this might happen. The general advise is to make your objects immutable per default, and make defensive copies when mutating arguments or global variables, as in following implementation

```
function twoDaysAheadDefensive(DateTime $date): int {
    $copy = new DateTime('@'. $date->getTimestamp());
    $copy->add(new DateInterval('PT48H'));
    return $copy->getTimestamp();
}
```

## Favor composition over inheritance

There are two ways to achieve polymorphic behaviour. One is extension, another is composition. Let's say we have `NotificationAgent` that sends a notification to a user, depending on user's preferences of course. There is a notification agent that sends emails, and the one that uses SMS. Inheritance based solution could look something like this


```
abstract class NotificationAgent {
    abstract protected function send(Message $message);
}

class EmailNotificationAgent extends NotificationAgent {
    ...
    protected function send(Message $message) {
        $this->sendGrid->send($message, $this->email);
    }
}

class SmsNotificationAgent extends NotificationAgent {
    ...
    protected function send(Message $message) {
        $this->sns->send($message, $this->telephone);
    }
}
```

The agent can be initialized by the `UserService` as following

```
class UserService {
    ...
    public function notificationAgent(User $user): NotificationAgent {
        if ($user->useSms()) {
            return new SmsNotificationAgent($this->sns, $user->telephone());
        } else if ($user->useEmail()) {
            return new EmailNotificationAgent($this->sendGrid, $user->email());
        } else {
            return new EmptyNotificationAgent();
        }
    }
}
```

How will a version with composition look like?

```
class NotificationAgent {
    ...
    public function __construct(array $dispatchers) {
        $this->dispatches = $dispatches;
    }
    public function send(Message $message) {
        foreach ($this->dispatchers as $dispatcher) {
            $dispatcher->dispatch($message);
        }
    }
}

interface Dispatcher {
    public function dispatch(Message $message);
}

class Email implements Dispatcher {
    ...
    protected function dispatch(Message $message) {
        $this->sendGrid->send($message, $this->email);
    }
}

class SmsDispatcher implements Dispatcher {
    ...
    protected function dispatch(Message $message) {
        $this->sns->send($message, $this->telephone);
    }
}
```

The user service looks a bit different. Instead of using different `NotificationUser` types, it uses the same type but injects different dispatches depending on user preferences:

```
class UserService {
    ...
    public function notificationAgent(User $user): NotificationAgent {
        $dispatchers = [];
        if ($user->useSms()) {
            $dispatchers[] = new SmsDispatcher(...);
        }
        if ($user->useEmail()) {
            dispatchers[] = new EmailDispatcher(...);
        }
        return new NotificationAgent($dispatchers);
    }
}
```

Note the subtle but very important difference. With composition we have split the notification agent's functionality from dispatching functionality. This provides us with many advantages. For example we now can have multiple dispatchers within the same object, or we can extend method `dispatch(Message $message): bool` to return status code and use this code to stop dispatching on first success:

```
class NotificationAgent {
    ...
    public function __construct(array $dispatchers) {
        $this->dispatches = $dispatches;
    }
    public function send(Message $message): bool {
        foreach ($this->dispatchers as $dispatcher) {
            if ($dispatcher->dispatch($message)) {
                return true;
            }
        }
        return false;
    }
}

interface Dispatcher {
    public function dispatch(Message $message): bool;
}
```

Try to implement the same functionality with original inheritance based solution as an exercise.

It's hard to extract the essense of this composition-vs-inheritance "trick" though. Good composition requires you to extract abstractions, discovering new boundaries between objects, whereas inheritance is more like keeping everything in the family, very convenient, but very rigid.


## Design for inheritance otherwise prohibit it

If the compositio is so awesome, why would I ever use inheritance, right? Not quite.

... need a nice example




## Prefer interfaces to abstract classes



## Prefer class hierarchies to tagged classes



## Use function objects to represent strategies



## Enums, PHP style






## Use anonymous classes to fine grained visibility

Unlike many other OOP languages PHP doesn't have package visibility, inner static classes, and many other means to hide implementation details. But there are ways to emulate those and achieve similar results. Let's say we want to return a complex object that is initialized in multiple stages and the stages and only the builder needs to have access to initialization methods. In java it whould be implemented as follows:

@todo think of a simple example
@todo add benchmark















# Methods

## Check parameters for validity

@add pslam, phpdoc, type hints, runtime checks




## Design method signatures carefully



## Make defensive copies when needed

@todo DateTime example

## Use immutable objects

## Do not overload with default values without necessity, i.e. don't merge signatures

PHP method signatures only consist of the method's name. Declaring two methods `read(string $path)` and `read(array $settings)` will result in a PHP error. This often leads to the following solution

```
/**
 * @param string|array $location
 */
function read($location) {
    if (is_array($location))
        ...
    } else {
        ...
    }
 }
```

This is quite ugly and not necessarily helps your IDE / static analyser / next guy with understanding your code. Much simpler is to declare two functions:


```
function readFromPath(string $path)
{

}

function readFromArray(array $settings)
{

}
```

The only reason I see people don't do that is that they are used to languages where method signatures include parameter types.

## Use varargs, few tricks

This one is actually completely different from Java advise.

## Return empty arrays, not nulls, i.e. consider what your base case is

PHP type system is very liberal. From the point of view of the `if` operator quire a few very different objects are equivalent. Moreover the lack of native generics (there are some surrogates in PHPDoc) makes it hard to catch many problems.

See for example the following method
```
/**
 * @param int $companyId
 * @return array|null
 */
public function findCustomers(int $companyId)
{
    ...
}
```

There are numerous improvements than can be made to this method signature, including adding return type `array` to signature, adding `Customer[]` annotation to PHPDoc, returning an empty array on empty result set.

```
/**
 * @param int $companyId
 * @return Customer[]
 */
public function findCustomers(int $companyId): array
{
    ...
}
```

The advantages of the following type are:

- Hard failure on non-array return
- Static analysers and IDEs are able to pick up on return type and use proper type in a loop over result set
- There's algorythmically no difference between handling companies with and without customers

```
foreach (findCustomers(12) as $customer) {
    $customerService->makeHappy($customer);
}
```

## Use PHPDoc when necessary

## Do not use too many default values




# General Programming

## Use types signatures

## Use assertions for additional checks

There's not much that PHP type system can check, besides it's going to be runtime check anyway. You can mitigate the issue to some degree by using tools like PHPStorm inspections and php-stan, and you should, they're great. But how about checking that the array that's been passed as an argument contains only strings? For critical parts of the application I usually add asertions that implement extended type checks, as in

```
public function buildIntergerList(array $list)
{
    assert(are_ints($list));
}
```

Assertions can be disabled per global switch, so in light of this the version above is somewhat better than the following

```
public function buildIntergerList(array $list)
{
    foreach ($list as $item) {
        assert(is_int($item));
    }
}
```

## Use global exception and error handlers

## Use logger (PSR) and pass additional context

## Use reflection

## Use closures to reduce interdependency (cohesion?)


# Exceptions

## Use exceptions only for exceptional conditions



## Favor the use of standard exceptions



## Throw exceptions appropriate to the abstraction



## Include failure capture information

One advantage of using exceptions is ability to split the source of exceptional situation from the code which knows what to do about it. In fact more often than not the two bits of code reside in components separated by layers of abstraction. Take for example a situation where you need to download a file on user's behalf and show an error message when a file cannot be downloaded. Downloading component knows nothing about how much information to show about exception, which language the current user prefers and whether there's additional exception handling after showing the user a warning.

In light of this is quite convenient to capture additional information inside of an exception, that provides the context in which an exception occured. For the example above one can add `$url` and `$responseCode` data to the exception and let the presentational code decide what to do about this

```
class FileDownloadException extends Exception
{
    private $url;

    private $responseCode;

    ...

    public function url(): string {
        return $this->url;
    }

    public function responseCode(): int {
        return $this->responseCode;
    }
}
```

## Do not ignore exceptions

Example: Application component unavailable, unexpected exception.

