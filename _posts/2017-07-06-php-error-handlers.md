---
layout: post
title: PHP Global Error Handler
---

What do we do with uncaught `Throwable`s in PHP 7? Following things come to mind:

- Ideally we would like only one handler defined
- The handler should be stateful, i.e. we should be able to capture something like `LoggerInterface` in it
- The additional handler should't hide code location, i.e. in logs we should not see the file and line of the handler, but the previous hop in the stack trace
- Registration of handlers should be trivial

One implementation that satisfies these criteria is the following. For an error handler we get

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

For an exception handler

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

And for the regisration

```
set_error_handler(new ErrorHandler($logger), E_ALL);
set_exception_handler(new ExceptionHandler($logger));
```

For this to work you'd also need [`Log4Php` fork](https://packagist.org/packages/vasily-kartashov/log4php), version `4.*`. The fork essentially:
- adds namespaces,
- implements `psr/log` interface,
- accounts for new `Throwable` hierarchy change in PHP 7,
- adds `GenericHandler` and `GenericLogger` marker interfaces used to skip additional logging layers in stack traces
