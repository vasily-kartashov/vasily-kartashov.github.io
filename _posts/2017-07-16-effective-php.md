---
layout: post
title: Effective PHP
---

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
        if (!self::validSsml($ssm;)) {
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

It's not as elegant as a standard Java implementation, but it's close to be airtight.

- The only way to create a task if though a builder,
- The builder can only be created by calling `Task::builder()`
- The resulting `Task` object is immutable

Unpleasantly redundant function `$constructor` is the necessary evil capturing constructed object and returning it

```

## Enforce singleton with a static property

There's one very simple way to create singletons in PHP.

```
class Environment
{
    public static function database(): Database
    {
        static $database;
        if (!isset($database)) {
            $database = new Database();
        }
        return $database;
    }
}
```

Please note, this is going to be a real singleton, and will be shared across all instances of class `Envrionment`. If you don't want this, use instance property instead.

## Enforce noninstantiability with a private constructor



# Methods common to all object

## Implement __toString



## Implement JsonSerializable

## Learn to use clone

http://php.net/manual/en/language.oop5.cloning.php#object.clone

## Use __destruct when necessary



# Classes and Interfaces

## Minimize the accessibility of classes and memebers

## In public clsses, use accessor methods, not public fields

## Minimize mutability

## Favor composition over inheritance

## Design for inheritance otherwise prohibit it

## Prefer interfaces to abstract classes

## Inheritance, PHP style

## Prefer class hierarchies to tagged classes

## Use function objects to represent strategies

## Enums, PHP style




## Use anonymous classes to fine grained visibility

Unlike many other OOP languages PHP doesn't have package visibility, inner static classes, and many other means to hide implementation details. But there are ways to emulate those and achieve similar results. Let's say we want to return a complex object that is initialized in multiple stages and the stages and only the builder needs to have access to initialization methods. In java it whould be implemented as follows:

@todo think of a simple example
@todo add benchmark





# Methods

## Check parameters for validity

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



# Exceptions

## Use exceptions only for exceptional conditions



## Favor the use of standard exceptions



## Throw exceptions appropriate to the abstraction



## Include failure capture information



## Do not ignore exceptions


