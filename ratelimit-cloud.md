If you’re running only a single instance of Spring Cloud Gateway and want to avoid running Redis, that’s perfectly fine — you can implement in-memory rate limiting instead.

Spring Cloud Gateway’s built-in RequestRateLimiter filter requires Redis, but you can create a custom filter using an in-memory rate limiter like Bucket4j or even a simple ConcurrentHashMap with timestamp logic.

⸻

Option 1: Use Bucket4j (recommended for flexibility)

Bucket4j is a Java library for token-bucket rate limiting that works well in-memory.

Steps:
	1.	Add the dependency in pom.xml:

<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>7.6.0</version>
</dependency>

	2.	Create a custom filter:

@Component
public class RateLimiterGatewayFilterFactory extends AbstractGatewayFilterFactory<RateLimiterGatewayFilterFactory.Config> {

    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    public RateLimiterGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String key = exchange.getRequest().getRemoteAddress().toString(); // or use API key/IP
            Bucket bucket = cache.computeIfAbsent(key, k -> Bucket4j.builder()
                    .addLimit(Bandwidth.classic(config.capacity, Refill.greedy(config.refillTokens, Duration.ofSeconds(config.refillDuration))))
                    .build());

            if (bucket.tryConsume(1)) {
                return chain.filter(exchange);
            } else {
                exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                return exchange.getResponse().setComplete();
            }
        };
    }

    @Data
    public static class Config {
        private int capacity;
        private int refillTokens;
        private int refillDuration; // in seconds
    }
}

	3.	Configure the filter in application.yml:

spring:
  cloud:
    gateway:
      routes:
        - id: rate_limited
          uri: http://localhost:8080
          predicates:
            - Path=/api/**
          filters:
            - name: RateLimiter
              args:
                capacity: 10
                refillTokens: 10
                refillDuration: 1

	4.	Register the filter in your configuration class (optional if using Spring component scanning).

⸻

Option 2: Simple ConcurrentHashMap with timestamps

Good for learning or very basic use cases, but not production-ready.

⸻

Would you like a working example project or a GitHub repo setup for this approach?
