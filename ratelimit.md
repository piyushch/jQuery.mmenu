Okay, here's a complete implementation using Bucket4j for in-memory rate limiting in Spring Cloud Gateway, suitable for a single instance.

**Assumptions:**

*   Spring Boot Version: 3.2.x or compatible (Using 3.2.5 as an example, 3.4.4 isn't released as of late 2024, but this structure should work).
*   Java Version: 17+
*   Build Tool: Maven (Gradle setup would be similar)
*   Goal: Rate limit based on the client's IP address.

---

**1. `pom.xml` (Dependencies)**

Make sure you have the necessary dependencies.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <!-- Use a recent stable Spring Boot 3 version -->
        <version>3.2.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>gateway-bucket4j</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gateway-bucket4j</name>
    <description>Spring Cloud Gateway with Bucket4j Rate Limiting</description>

    <properties>
        <java.version>17</java.version>
        <!-- Use the Spring Cloud version compatible with your Spring Boot version -->
        <spring-cloud.version>2023.0.1</spring-cloud.version>
        <bucket4j.version>8.9.0</bucket4j.version> <!-- Check for latest Bucket4j version -->
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Spring Cloud Gateway -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <!-- Bucket4j Core -->
        <dependency>
            <groupId>com.bucket4j</groupId>
            <artifactId>bucket4j-core</artifactId>
            <version>${bucket4j.version}</version>
        </dependency>

        <!-- Caffeine Cache (for managing buckets efficiently) -->
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
            <!-- Version managed by Spring Boot -->
        </dependency>

        <!-- Optional: Lombok for less boilerplate -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Configuration Processor for application.properties binding -->
        <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-configuration-processor</artifactId>
             <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

---

**2. `src/main/resources/application.properties`**

```properties
# Server Port
server.port=8080

# Logging levels (optional, for debugging)
logging.level.org.springframework.cloud.gateway=TRACE
logging.level.com.example.gateway=DEBUG # Your package name

# Spring Cloud Gateway Configuration
spring.cloud.gateway.routes[0].id=limited_api_route
spring.cloud.gateway.routes[0].uri=https://httpbin.org # Target service (replace with yours)
spring.cloud.gateway.routes[0].predicates[0]=Path=/get/** # Path to protect
spring.cloud.gateway.routes[0].filters[0].name=RequestRateLimiter
# Reference the KeyResolver bean by its name (ipKeyResolver) using SpEL
spring.cloud.gateway.routes[0].filters[0].args.key-resolver=#{ipKeyResolver}
# Reference the custom RateLimiter bean by its name (bucket4jRateLimiter) using SpEL
spring.cloud.gateway.routes[0].filters[0].args.rate-limiter=#{bucket4jRateLimiter}
# Add specific headers (optional)
spring.cloud.gateway.routes[0].filters[1].name=SetResponseHeader
spring.cloud.gateway.routes[0].filters[1].args.name=X-RateLimit-Info
spring.cloud.gateway.routes[0].filters[1].args.value=Handled by Bucket4j In-Memory Limiter


# --- Configuration for our Custom Bucket4j Rate Limiter ---
# Matches the CONFIGURATION_PROPERTY_NAME in Bucket4jRateLimiter.java
# The key under 'configs' (e.g., 'limited_api_route') MUST match the route ID defined above.
spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.limited_api_route.replenish-rate=5
spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.limited_api_route.burst-capacity=10
spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.limited_api_route.requested-tokens=1

# Example of another route with different limits
# spring.cloud.gateway.routes[1].id=another_route
# spring.cloud.gateway.routes[1].uri=http://example.com
# spring.cloud.gateway.routes[1].predicates[0]=Path=/another/**
# spring.cloud.gateway.routes[1].filters[0].name=RequestRateLimiter
# spring.cloud.gateway.routes[1].filters[0].args.key-resolver=#{ipKeyResolver}
# spring.cloud.gateway.routes[1].filters[0].args.rate-limiter=#{bucket4jRateLimiter}
#
# spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.another_route.replenish-rate=20
# spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.another_route.burst-capacity=40
# spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.another_route.requested-tokens=2

# --- Caffeine Cache Configuration (Optional Tuning) ---
# Adjust cache size and expiry as needed based on expected unique IPs and memory
spring.cache.caffeine.spec=maximumSize=10000,expireAfterAccess=3600s
```

---

**3. `src/main/java/com/example/gateway/GatewayApplication.java`**

```java
package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}
```

---

**4. `src/main/java/com/example/gateway/ratelimiter/Bucket4jRateLimiter.java`**

This is the core custom `RateLimiter` implementation.

```java
package com.example.gateway.ratelimiter;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import io.github.bucket4j.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.gateway.filter.ratelimiter.AbstractRateLimiter;
import org.springframework.cloud.gateway.filter.ratelimiter.RateLimiter;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Map;
import java.util.Objects;

@Component("bucket4jRateLimiter") // Define bean name for SpEL reference
@Primary // Make this the primary RateLimiter if Redis one is on classpath
@Slf4j
public class Bucket4jRateLimiter extends AbstractRateLimiter<Bucket4jRateLimiter.Config> {

    // Matches the prefix in application.properties
    public static final String CONFIGURATION_PROPERTY_NAME = "bucket4j-inmemory";

    private final Cache<String, Bucket> cache;

    public Bucket4jRateLimiter(
        @Qualifier("bucket4jConfigCache") Cache<String, Bucket> cache // Inject the configured cache
    ) {
        // Pass the Config class, property name, and optional validator
        super(Config.class, CONFIGURATION_PROPERTY_NAME, null); // Using default validator for now
        this.cache = cache;
        log.info("Initialized Bucket4j In-Memory Rate Limiter");
    }

    @Override
    public Mono<Response> isAllowed(String routeId, String id) {
        // Retrieve the configuration specific to this routeId
        // The AbstractRateLimiter parent class handles loading this from properties
        Config routeConfig = getConfig().get(routeId);

        if (routeConfig == null) {
            log.warn("No RateLimiter configuration found for route: {}", routeId);
            // Deny request if no configuration is found for safety
            return Mono.just(new Response(false, Map.of()));
        }

        log.trace("Rate limiting request for route: {}, key: {}, config: {}", routeId, id, routeConfig);

        // Get or create the bucket for the unique key (e.g., IP address)
        Bucket bucket = cache.get(id, key -> createNewBucket(routeConfig));

        // Try to consume the configured number of tokens
        long tokensToConsume = routeConfig.getRequestedTokens();
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(tokensToConsume);

        Response response;
        if (probe.isConsumed()) {
            response = new Response(true, createHeaders(routeConfig, probe.getRemainingTokens()));
            log.trace("Request allowed for key '{}'. Remaining tokens: {}", id, probe.getRemainingTokens());
        } else {
            response = new Response(false, createHeaders(routeConfig, probe.getRemainingTokens()));
            log.warn("Rate limit exceeded for key '{}'. Tokens needed: {}, Available: {}",
                    id, tokensToConsume, probe.getRemainingTokens());
             // Optionally add 'Retry-After' header (requires calculating nanosToWaitForRefill)
             // response.getHeaders().put("Retry-After", String.valueOf(probe.getNanosToWaitForRefill() / 1_000_000_000L));
        }

        return Mono.just(response);
    }

    private Bucket createNewBucket(Config config) {
        log.debug("Creating new bucket with config: {}", config);
        Refill refill = Refill.intervally(config.getReplenishRate(), Duration.ofSeconds(1));
        Bandwidth limit = Bandwidth.classic(config.getBurstCapacity(), refill);
        return Bucket.builder()
                .addLimit(limit)
                .build();
    }

    private Map<String, String> createHeaders(Config config, long remainingTokens) {
        return Map.of(
                "X-RateLimit-Remaining", Long.toString(remainingTokens),
                "X-RateLimit-Burst-Capacity", Long.toString(config.getBurstCapacity()),
                "X-RateLimit-Replenish-Rate", Long.toString(config.getReplenishRate())
        );
    }

    // --- Configuration Class ---
    // Needs to match the properties defined under 'configs' in application.properties
    // Lombok annotations generate getters, setters, toString, etc.
    @Data // Includes @Getter, @Setter, @ToString, @EqualsAndHashCode, @RequiredArgsConstructor
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class Config {
        private int replenishRate; // Tokens per second
        private int burstCapacity; // Max tokens stored
        @Builder.Default // Provide a default value if not specified
        private int requestedTokens = 1; // Tokens consumed per request
    }
}
```

---

**5. `src/main/java/com/example/gateway/config/RateLimiterConfiguration.java`**

This class defines the `KeyResolver` bean and configures the Caffeine cache used by the rate limiter.

```java
package com.example.gateway.config;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import io.github.bucket4j.Bucket;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.cloud.gateway.filter.ratelimiter.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

import java.net.InetSocketAddress;
import java.time.Duration;
import java.util.Objects;

@Configuration
@Slf4j
public class RateLimiterConfiguration {

    /**
     * Defines how to extract the key for rate limiting from the request.
     * Here, we use the client's IP address.
     */
    @Bean(name = "ipKeyResolver") // Bean name used in application.properties SpEL
    public KeyResolver ipKeyResolver() {
        log.info("Creating IP KeyResolver bean");
        return exchange -> {
            InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();
            // Handle cases where remote address might be null (e.g., during tests or unusual setups)
            if (remoteAddress != null && remoteAddress.getAddress() != null) {
                 String ip = remoteAddress.getAddress().getHostAddress();
                 log.trace("Resolved key: {}", ip);
                 return Mono.just(ip);
            }
            log.warn("Could not resolve remote IP address for rate limiting. Using fallback key.");
            // Fallback key if IP is not available - limits all such requests together
            return Mono.just("fallback-key");
        };
    }

     /**
      * Creates and configures the Caffeine cache for storing Bucket4j buckets.
      * The cache specification can be tuned via application.properties
      * (e.g., spring.cache.caffeine.spec=maximumSize=5000,expireAfterAccess=1h)
      */
     @Bean
     @Qualifier("bucket4jConfigCache") // Qualifier to inject this specific cache
     public Cache<String, Bucket> bucket4jConfigCache(Caffeine<Object, Object> caffeine) {
         log.info("Creating Caffeine cache for Bucket4j buckets");
         // Use the Caffeine builder configured by Spring Boot based on properties
         return caffeine.build();
     }

     /**
      * Provides the default Caffeine configuration builder.
      * Spring Boot Boot will automatically configure this bean based on
      * properties under `spring.cache.caffeine.spec`.
      * If no spec is provided, it uses default Caffeine settings.
      */
     @Bean
     public Caffeine<Object, Object> caffeineConfig() {
         log.info("Configuring Caffeine builder");
         // Returns the default builder; Spring Boot auto-configuration applies properties
         return Caffeine.newBuilder()
                .recordStats(); // Optional: Enable stats for monitoring
     }
}
```

---

**How it Works:**

1.  **Request Arrives:** A request hits the Gateway matching `/get/**`.
2.  **`RequestRateLimiter` Filter:** This filter is triggered.
3.  **`KeyResolver`:** The `ipKeyResolver` bean is invoked (`#{ipKeyResolver}`). It extracts the client's IP address from the request.
4.  **`RateLimiter`:** The `bucket4jRateLimiter` bean is invoked (`#{bucket4jRateLimiter}`). Its `isAllowed` method is called with the `routeId` (`limited_api_route`) and the resolved key (the IP address).
5.  **Configuration Lookup:** `Bucket4jRateLimiter` uses the `routeId` to look up its configuration (replenish rate, burst capacity) from the properties bound under `spring.cloud.gateway.ratelimiter.bucket4j-inmemory.configs.limited_api_route`.
6.  **Cache Interaction:** It uses the IP address as the key to look up a `Bucket` object in the Caffeine cache (`bucket4jConfigCache`).
7.  **Bucket Creation/Retrieval:** If a `Bucket` for that IP doesn't exist or has expired from the cache, `createNewBucket` is called using the route-specific configuration. The new `Bucket` is stored in the cache.
8.  **Token Consumption:** `bucket.tryConsumeAndReturnRemaining()` is called on the retrieved/created `Bucket`. Bucket4j handles the token bucket logic internally (refilling over time, checking capacity).
9.  **Response:** Based on whether consumption was successful, a `Response` object is created (allowing or denying the request) and appropriate headers (`X-RateLimit-*`) are added.
10. **Filter Chain:** If allowed, the request continues down the filter chain and is proxied to `https://httpbin.org`. If denied, the Gateway immediately returns a `429 Too Many Requests` status.

**To Run:**

1.  Save the files in the correct directory structure.
2.  Run the `GatewayApplication` main method.
3.  Send requests to `http://localhost:8080/get` (e.g., using `curl` or Postman). You should see the `X-RateLimit-*` headers in the response. If you send requests faster than the `replenish-rate`, you'll eventually get `429 Too Many Requests`.

```bash
# Example test (run multiple times quickly)
curl -i http://localhost:8080/get
```

This setup provides effective in-memory rate limiting for your single Gateway instance without needing Redis. Remember the state will be lost on restarts.
