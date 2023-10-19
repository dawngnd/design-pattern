---
title: Ambassador
category: Structural
language: en
tag:
  - Decoupling
  - Cloud distributed
---

## Intent

Mục đích gần như giải quyết vấn đề gọi sang tài nguyên bên thứ 3, và có cơ chế check timeout , retry ...
Giải quyết vấn đề về xác định độ trễ của bên cung cấp tài nguyên 


**Programmatic Example**


```java
interface RemoteServiceInterface {
    long doRemoteFunction(int value) throws Exception;
}
```

A remote services represented as a singleton.

```java
@Slf4j
public class RemoteService implements RemoteServiceInterface {
    private static RemoteService service = null;

    static synchronized RemoteService getRemoteService() {
        if (service == null) {
            service = new RemoteService();
        }
        return service;
    }

    private RemoteService() {}

    @Override
    public long doRemoteFunction(int value) {
        long waitTime = (long) Math.floor(Math.random() * 1000);

        try {
            sleep(waitTime);
        } catch (InterruptedException e) {
            LOGGER.error("Thread sleep interrupted", e);
        }

        return waitTime >= 200 ? value * 10 : -1;
    }
}
```

A service ambassador adding additional features such as logging, latency checks

```java
@Slf4j
public class ServiceAmbassador implements RemoteServiceInterface {
  private static final int RETRIES = 3;
  private static final int DELAY_MS = 3000;

  ServiceAmbassador() {
  }

  @Override
  public long doRemoteFunction(int value) {
    return safeCall(value);
  }

  private long checkLatency(int value) {
    var startTime = System.currentTimeMillis();
    var result = RemoteService.getRemoteService().doRemoteFunction(value);
    var timeTaken = System.currentTimeMillis() - startTime;

    LOGGER.info("Time taken (ms): " + timeTaken);
    return result;
  }

  private long safeCall(int value) {
    var retries = 0;
    var result = (long) FAILURE;

    for (int i = 0; i < RETRIES; i++) {
      if (retries >= RETRIES) {
        return FAILURE;
      }

      if ((result = checkLatency(value)) == FAILURE) {
        LOGGER.info("Failed to reach remote: (" + (i + 1) + ")");
        retries++;
        try {
          sleep(DELAY_MS);
        } catch (InterruptedException e) {
          LOGGER.error("Thread sleep state interrupted", e);
        }
      } else {
        break;
      }
    }
    return result;
  }
}
```

A client has a local service ambassador used to interact with the remote service:

```java
@Slf4j
public class Client {
  private final ServiceAmbassador serviceAmbassador = new ServiceAmbassador();

  long useService(int value) {
    var result = serviceAmbassador.doRemoteFunction(value);
    LOGGER.info("Service result: " + result);
    return result;
  }
}
```

Here are two clients using the service.

```java
public class App {
  public static void main(String[] args) {
    var host1 = new Client();
    var host2 = new Client();
    host1.useService(12);
    host2.useService(73);
  }
}
```

Here's the output for running the example:

```java
Time taken (ms): 111
Service result: 120
Time taken (ms): 931
Failed to reach remote: (1)
Time taken (ms): 665
Failed to reach remote: (2)
Time taken (ms): 538
Failed to reach remote: (3)
Service result: -1
```

## Typical Use Case

* Control access to another object
* Implement logging
* Implement circuit breaking
* Offload remote service tasks
* Facilitate network connection

## Known uses

* [Kubernetes-native API gateway for microservices](https://github.com/datawire/ambassador)

## Related patterns

* [Proxy](https://java-design-patterns.com/patterns/proxy/)

## Credits

* [Ambassador pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/ambassador)
* [Designing Distributed Systems: Patterns and Paradigms for Scalable, Reliable Services](https://www.amazon.com/s?k=designing+distributed+systems&sprefix=designing+distri%2Caps%2C156&linkCode=ll2&tag=javadesignpat-20&linkId=a12581e625462f9038557b01794e5341&language=en_US&ref_=as_li_ss_tl)