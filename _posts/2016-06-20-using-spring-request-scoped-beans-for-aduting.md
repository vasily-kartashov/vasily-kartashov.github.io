---
layout: post
title: Using Spring request scoped beans for auditing
tags: java spring aop
---

Every multi-layerd application is built on idea of separation of concerns. The presentational layer knows about raw JSON payload and HTTP headers, but the service layer would only know about the domain objects, whereas the
underlying data access layer wouldn't even know about which business methods the current transaction is spanning.

So for this post let's assume that we want a tracing context during the whole execution. We want to timer all the methods in our controllers and see who sent the request. The approach is easily extensible, so we'll keep the example to bare minimum.

Let's create a timer aspect that will count execution time

```java
@Aspect
@Component
public class Timer {

    @Autowired
    private Auditor auditor;

    @Pointcut("execution(* com.kartashov.auditing.controllers.*.*(..))")
    public void methods() {}

    @Around("methods()")
    public Object profile(ProceedingJoinPoint point) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = point.proceed();
        long time = System.currentTimeMillis() - start;
        auditor.register(point.getSignature().toShortString(), time);
        return result;
    }
}
```

Make sure the aspect is marked as `@Component` otherwise Spring will not pick it up. We also need to create an Auditor who would store the execution context for the duration of request.

```java
public class Auditor {

    private final static Logger logger = LoggerFactory.getLogger(Auditor.class);

    private List<Record> records = new ArrayList<>();
    private String remoteAddress;

    void register(String method, long time) {
        records.add(new Record(method, time));
    }

    public void setRemoteAddress(String remoteAddress) {
        this.remoteAddress = remoteAddress;
    }

    @PreDestroy
    public void trace() {
        logger.info("Audit record. Request from {}\n{}",
                remoteAddress,
                records.stream().map(Record::toString).collect(Collectors.joining("\n")));
    }

    static class Record {
        ...
    }
}
```

We also need to report remote address from our controllers. For the sake of exposition I will add remote address manually and not use AspectJ for this.

```java
@RestController
public class PingController {

    @Autowired
    private Auditor auditor;

    @RequestMapping("/")
    public String ping(HttpServletRequest request) {
        auditor.setRemoteAddress(request.getRemoteAddr());
        return Instant.now().toString();
    }
}
```

And the final thing we need to explain to our Spring boot application what we're up to:

- Enable AspectJ proxies
- Make Auditor request scoped
- Make sure that Sprign autowires not the object but a scoped proxy that would create separate instances for every web request

```java
@SpringBootApplication
@EnableAspectJAutoProxy
@Configuration
public class Application {

    public static void main(String... args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
    public Auditor auditor() {
        return new Auditor();
    }
}
```

We're all set. Running the application will give us the following trace

```
2016-06-17 13:23:49.440  INFO 6738 : Audit record. Request from 0:0:0:0:0:0:0:1
PingController.ping(..): 24ms
```
This approach can be further extended with factories that would produce different request scoped Auditors for different situations. Important point to take here is that by autowiring request scoped Auditors we are keeping cross-cutting auditing aspect mostly separate from our business logic.
