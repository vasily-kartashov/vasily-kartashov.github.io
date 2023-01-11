---
layout: post
title: PHP Global Error Handler
tags: php
---

When it comes to handling uncaught `Throwables` in PHP 7, there are a few things to consider:

- It's best to have only one handler defined
- The handler should be stateful, meaning it should be able to capture something like a `LoggerInterface`
- The additional handler shouldn't hide the code location, meaning that in logs, we should see the file and line of the previous hop in the stack trace, not the handler's
- Registering handlers should be simple and straightforward

Here's an example of an implementation that meets these criteria:

```
class ErrorHandler implements GenericHandler
{
    private $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function __invoke($code, $message, $file, $line)
    {
        switch ($code) {
            case E_USER_ERROR:
                if (extension_loaded('newrelic')) {
                    newrelic_notice_error($message);
                }
                $this->logger->critical($message);
                break;
            default:
                $this->logger->warning($message);
                break;
        }
        return true;
    }
}
```

And here's an example of an exception handler:

```
class ExceptionHandler implements GenericHandler
{
    private $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function __invoke(Throwable $exception)
    {
        if (extension_loaded('newrelic')) {
            newrelic_notice_error($exception->getMessage(), $exception);
        }
        $this->logger->warning($exception->getMessage(), ['exception' => $exception]);
        http_response_code(500);
    }
}
```

And for the regisration:

```
set_error_handler(new ErrorHandler($logger), E_ALL);
set_exception_handler(new ExceptionHandler($logger));
```
In addition, to make this work, you will need to use [a fork of Log4Php](https://packagist.org/packages/vasily-kartashov/log4php), version 4.*, that:

- includes namespaces,
- implements the `psr/log` interface,
- accounts for the new `Throwable` hierarchy change in PHP 7,
- adds `GenericHandler` and `GenericLogger` marker interfaces that are used to skip additional logging layers in stack traces.
