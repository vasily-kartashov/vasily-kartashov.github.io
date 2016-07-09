---
layout: post
title: A simple caching web framework
---

A while ago I was working at an agency and we had to develop a fairly simple set of web apps, that would scale very well and integrate tightly with a shared caching bus and content distribution network. Performance was the key. Especially for the markets around the globe, some of them with generally slow internet connections. This post is a short schematic description about the design I came up with.

So I started with two main considerations in my mind. The first was to integrate with HTTP protocol as transparently as possible. There's a set of RFCs out there, I am specifically talking about [RFC 2616](https://www.ietf.org/rfc/rfc2616.txt). The set of abstractions was modelled as closely as possibly to the this specification. By working with standards I hoped to easily integrate with numerous web caches and skip unnecessary refreshes. Additional but non-neglgible benefit was that a person knowing how HTTP works would be able to understand application logic easily.

The second, also important consideration was to not invent any DSLs, embedded or otherwise, used to replace constructs that the programming languages are already equipped with. To give an example of what I am talking about, I didn't want to write configurations / annotations / ant-matchers and similar nonsense in order to say that URLs starting with `/admin` require that the user is authenticated and belongs to the group admin. Instead the following code would do just fine:

```php
if ($request->startsWith('/admin')) {
  if (!$user->isAuthorized() or !$user->hasRole('admin')) {
    return new NonAuthorizedResponse();
  }
  ...
}
```

Generally I've noticed a tendency in programming community to write frameworks that would _force_ you to do the right thing. They're supposed to provide you with a rock solid skeleton for your next application, equip you with enough tools to tweak the behavior without giving you a chance to break things. As usual, in real life the tools given are never enough and you end up hacking and breaking things anyway.

Third consideration was to help with code reuse as much as possible. Most websites are quite simply built, there's a master template and numerous adjustments to it. There are generally two ways to reuse templates and resources serving them. First you can extract common parts into helpers where you end up with headers, footers, navigation panels and so on. Second way to use template inheritance to tweak just the specific areas of the page and not caring about how to assemble this page from parts.

After a couple of attempts I end up with the following set of abstractions. From HTTP specifications I needed

- Request
- Response
- Resource
- Entity

Additionally there was some non application-specific logic that needed the following set of abstractions

- Application
- Cache
- Template

Let's go through that list one by one before explaining how those parts fit together.

Request
-------

Very simple albeit wider interface that facades the HTTP request and adds a couple of path matching methods.

```php
interface Request {
    public function environmentNameEndsWith(string $suffix) : bool;
    public function getEnvironmentName() : string;
    public function getHeader(string $headerName) : string;
    public function getLanguageCodes() : array;
    public function getMethod() : string;
    public function getRemoteIpAddress() : string;
    public function getPath() : string;
    public function getBody() : string;
    public function hasParameter(string $name) : bool;
    public function getParameter(string $name, $defaultValue = null) : string;
    public function getParameters() : array;
    public function hasSessionParameter(string $name) : bool;
    public function getSessionParameter(string $name, $defaultValue = null) : string;
    public function getSessionParameters() : array;
    public function getCookie(string $name, $defaultValue = null) : string;
    public function pathMatches(string $path) : bool;
    public function pathMatchesPattern(string $pattern);
    public function pathStartsWith(string $prefix) : bool;
    public function pathStartsWithPattern(string $pattern);
    public function preconditionFulfilled(Entity $entity, Cache $cache) : bool;
}
```

The HTTP request is just an implementation of this interface and hides a lot of ugliness of handling super-globals. There are also additional implementations of this interface for unit tests and command line tools.

Response
--------

Response interface is trivial and tiny.

```php
interface Response {
    public function output(Request $request, Cache $cache) : void;
}
```

There is just one non trivial part about it. When response writes its output to stdout (btw. it may as well be easily abstracted by adding `writer` parameter to `output` method) we need to know what was the request, because there might be headers with preconditions that would modify the output. We also need to access the cache to see what was in the cache, returned cached value if possible without re-rendering response, and update the cached value. This is an interesting part, not so trivial. Let's se the implementation of `AbstractResponse` which is the superclass for all other responses.

```php
class AbstractResponse implements Response
{
    ...
    public function output(Request $request, Cache $cache) : void {
        ...
        if (!is_null($this->entity)) {
            $cacheEntry = $this->entity->loadOrGenerate($cache);
            $now = time();
            $maxAge = max(0, $cacheEntry['expires'] - $now);

            header('ETag: ' . $cacheEntry['tag']);
            header('Last-Modified: ' . $this->formatTimestamp($cacheEntry['modified']));
            header('Cache-Control: public, max-age=' . $maxAge);
            header('Expires: ' . $this->formatTimestamp($now + $maxAge));

            if ($this->embedEntity) {
                header('Content-Type: ' . $this->entity->getMediaType());
                header('Content-Length: ' . $cacheEntry['length']);
                header('Content-MD5: ' . $cacheEntry['digest']);
                $language = $this->entity->getContentLanguage();
                ...
                echo $cacheEntry['content'];
            }
        }
        exit;
    }
    ...
}
```

The code is pretty much self explanatory but I am gonna walk through it anyway. If there's an entity (payload) that we want to send, let's load the latest cache entry for this entity. If the cache doesn't have an entry it gets generated. Now we're updating the expiration, and version control headers and output the results, unless someone explicitly told us not to embed the entity. And who might that be? Let's take a look at `OkOrNotModifiedResponse` to see the usage:

```php
class OkOrNotModifiedResponse extends AbstractResponse {

    public function __construct(Entity $entity, Request $request) {
        parent::__construct();
        $this->setEntity($entity);
    }

    public function output(Request $request, Cache $cache) : void {
        if ($request->preconditionFulfilled($this->entity, $cache)) {
            $this->setStatus('200 OK');
            $this->setEmbedEntity(true);
        } else {
            $this->setStatus('304 Not Modified');
            $this->setEmbedEntity(false);
        }
        parent::output($request, $cache);
    }
}
```

We need an entity to see if it's expired or to see if its ETag matches. It doesn't necessarily mean that we want to render the entity into response as is the case with 304 response code.

Entity
------

The next abstraction is the above mentioned Entity, which is a pretty much the payload of the response plus additional headers like `Content-Type`, `Content-Length` and similar.

```php
interface Entity {
    public function getCachingTime() : int;
    public function getContent() : string;
    public function getContentLanguage() : string;
    public function getKey() : string;
    public function getMediaType() : string;
    public function loadOrGenerate(Cache $cache) : CacheEntry;
}
```

Important to note that every entity need to be able efficiently generate a key which is used to locate cache entry.

The main implementation of the `Entity` interface is `AbstractEntity`

```php
abstract class AbstractEntity implements Entity {
    ...
    public function loadOrGenerate(Cache $cache) : CacheEntry {
        if (!is_null($this->cacheEntry)) {
            return $this->cacheEntry;
        }

        $key = $this->getKey();
        list($this->cacheEntry, $found) = $cache->get($key);
        $now = time();
        $expires = $this->cacheEntry->expires ?? 0;

        if (!$found or $now >= $expires) {
            $content = $this->getContent();
            $tag = md5($content);
            if (is_array($this->cacheEntry) and $tag == $this->cacheEntry['tag']) {
                $this->cacheEntry->expires = $now + $this->getCachingTime();
            } else {
                $this->cacheEntry = new CacheEntry([
                    'content'  => $content,
                    'tag'      => $tag,
                    'digest'   => base64_encode(pack('H*', md5($content))),
                    'length'   => strlen($content),
                    'modified' => $now,
                    'expires'  => $now + $this->getCachingTime(),
                ]);
            }
            $cache->set($key, $this->cacheEntry);
        }

        return $this->cacheEntry;
    }
}
```

Again the code is fairly trivial. We're looking at the cache, and if there's already a value under this key, we check if it's expired or not. If it's fresh, then this is what we return without re-rendering the entity. Otherwise we render the entity and see if the entity's content has changed since last time. If not, we only update the `expires`, otherwise we add new cache entry.

`Entity` as such is not an abstraction for the payload. It's an abstraction for _payload generator that has memory_. We can ask an `Entity` object to give us the latest value, and this object is smart enough to:

- skip generation if the cache value if fresh enough,
- only update the expiration header if the newly generate value is the same as the old one,
- generate new value otherwise

One example of an entity would be a `JsonEntity` that only asks the extending classes to implement `getData` logic. The rest of cacheing and expiration logic is inherited from `AbstractEntity`. The complete class looks like following:

```php
abstract class AbstractJsonEntity extends AbstractEntity {

    abstract protected function getData();

    public function getContent() : string {
        return json_encode($this->getData());
    }

    public function getMediaType() : string {
        return 'application/json;charset=UTF-8';
    }
}
```

Resource
--------

The last abstraction on standard list is the `Resource` which is also a tiny interface

```php
interface Resource {    
    public function getResponse(Request $request) : Response;
}
```

The job of `Resources` is to translate requests into responses, in other words to interpret the request, see if it's appropriate, validate input, call business methods, generate response. Some of the resources are trivial and really just return an entity. For this they can extend `EntityResource`

```php
class EntityResource implements Resource {

    protected $entity;
    protected $methods;

    public function __construct(Entity $entity, array $methods = ['GET']) {
        $this->entity = $entity;
        $this->methods = $methods;
    }

    public function getResponse(Request $request) : Response {
        if (in_array($request->getMethod(), $this->methods)) {
            $response = new OKOrNotModifiedResponse($this->entity, $request);
            return $response;
        }
        return new MethodNotAllowedResponse($this->methods);
    }
}
```

Application
-----------

There's really no generic interface for applications. It can do whatever you want. I usually extend the following `AbstractApplication` which asks me two things

- How do I find resource that is responsible for this request, so that I can dispatch my request to it.
- How do we get to cache server.

```php
abstract class AbstractApplication {

    abstract protected function findResource(Request $request) : Resource;

    public function run(Request $request) : Response {
        $resource = $this->findResource($request);
        $response = $resource->getResponse($request);
        return $response;
    }

    abstract protected function getCache(Request $request) : Cache;

    public function output(Request $request, Response $response) : void {
        $response->output($request, $this->getCache($request));
    }
}
```

Cache
-----

The next abstraction is the most trivial one. The only reason it exists is to add a layer on top of Memcached, Redis, Elasticache and similar solutions.

```php
interface Cache {
    public function get(string $key, $defaultValue = null) : CacheEntry;
    public function set(string $key, $value, int $timeToLive = 0);
    public function delete(string ...$keys);
}

class CacheEntry {
    private $content;
    private $tag;
    private $digest;
    private $length;
    private $modified;
    private $expires;

    ...
}
```

Example application
-------------------

The most trivial application I can think of is an application that gives you current day, but doesn't recalculate it every time user visits the endpoint. We start with an entity that generates the day and sets proper expiration time for the result.

```php
class DayOfWeekEntity extends JsonEntity {

    // Only one cache entry shared across all entities of this type
    public function getKey() {
       return __CLASS__;
    }

    // Return current day of week
    public function getData() {
      return date('l');
    }

    // Cache till the end of the day today
    public function getCachingTime() {
       return 86400 - time() % 86400;
    }
}
```

Now let's create our own application

```php
class MyApplication extends AbstractApplication {

    // Check if the request is for a registered path
    // Return standard JSON Resource
    // Otherwise let NotFoundResource handle the request
    protected function findResource(Request $request) : Resource {
        if ($request->pathMatches('/dayOfWeek')) {
           return new JsonEntityResource(new DayOfWeekEntity(), ['GET']);
        }
        return new NotFoundResource();
    }

    // Return connection to the shared cache
    protected function getCache(Request $request) : Cache {
        return new Memcache('host001.elasticache.aws.', 11211);
    }
}
```

That's all you need to do. The first visit to the endpoint generates the entity. The value is stored in cache and sent back to the client. The response headers have ETag, validation and expiration headers explaining to the client and proxies that the value just generated won't change till the end of the day today. If the user hits refresh and the browser decides to send new request with ETag and validations headers, the response is going to be 304 and no new entity will be generated.  In fact the response might as well be from a CDN. The application might as well sit behind a load balancer, as the shared cache allows for easy parallelization.

Template inheritance
--------------------

Last thing that I want to mention here is the template handling. Some of the renderings can be

- quite expensive
- collect a lot of data from different sources for rendering
- have a lot in common

One thing is we're provided with is the abstract template entity that expects you to provide data, the path to the template, as well as the rendering engine for example Twig or Smarty.

```php
abstract class AbstractTemplateEntity extends AbstractEntity {
    abstract protected function getTemplatePath() : string;
    abstract protected function getTemplateData();
    abstract protected function getTemplateRenderer() : TemplateRenderer;
    public function getContent() : string {
        return $this->getTemplateRenderer()
            ->render($this->getTemplateData(), $this->getTemplatePath());
    }
}
```

The way it works in general, you will end up with an inheritance hierarchy of TemplateEntities that call `parent::getTemplateData()` to collect the data required to generate parent template, then adds it's own data fetching and renders the response. For example for the parent template `main.tpl`:

```
<html>
  <body>
    <h1>[[header]]</h1>
    [% block content %]
      Welcome!
    [/ block ]
  </body>
</html>
```

And the child template `details.tpl`:

```
[% extends "main.tpl" %]

[% block content %]
    [[message]]
[/ block ]
```

The same hierarchy is repeated with entities. `MainEntity` and `DetailsEntity` would contain the following code

```php
class MainEntity extends TwigTemplateEntity {

    protected function getTemplatePath() : string {
        return 'main.tpl';
    }

    protected function getTemplateData() {
        return [
            'header' => 'My Homepage'
        ];
    }
}

class DetailsEntity extends MainEntity {

    protected function getTemplatePath() : string {
        return 'details.tpl';
    }

    protected function getTemplateData() {
        return parent::getTemplateData() + [
            'message' => 'The page is under construction'
        ];
    }
}
```

The extending entities only contain the code generating the difference to the data set of the parent template, which helps with code reuse. As with any entities, you can specify the cacheing time, so if for example you want to regenerate templates depending on some timestamp from the database, this is what `getKey()` of the entity should use.

The source code of the base framework is [available on GitHub](https://github.com/vasily-kartashov/hamlet-core) under GPL 2 license. At one point I am going to publish a follow up note with a non-trivial example application.
