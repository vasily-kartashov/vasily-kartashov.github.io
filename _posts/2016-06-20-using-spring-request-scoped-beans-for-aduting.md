---
layout: post
title: Using Spring Request-Scoped Beans for Better Auditing
tags: java spring aop
---

Multi-layered apps are all about keeping different parts separate. The front end deals with JSON and HTTP headers, the service layer works with business objects, and the data layer doesn't even know which business methods are running.

In this post, we'll set up a simple way to track what's happening during a request. We'll time our controller methods and see who's making the request. This approach is easy to expand, but we'll keep it simple for now.

First, let's make a timer that will count how long methods take:

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

Make sure to mark it as `@Component` so Spring picks it up. We also need an Auditor to keep track of what's happening during the request:

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
        // Details left out for brevity
    }
}
```

We'll add the remote address in our controllers. Here's a simple example:

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

Lastly, we need to set up our Spring Boot app:

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

This does three important things:

- Turns on AspectJ proxies
- Makes Auditor request-scoped
- Tells Spring to use a special proxy that creates a new Auditor for each web request

When you run the app, you'll see something like this:

```
2016-06-17 13:23:49.440  INFO 6738 : Audit record. Request from 0:0:0:0:0:0:0:1
PingController.ping(..): 24ms
```

You can build on this idea to do more complex auditing. The key point is that by using request-scoped Auditors, we keep our auditing separate from our main business logic.
