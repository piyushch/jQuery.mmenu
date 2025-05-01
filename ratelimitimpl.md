Great! Here’s a minimal working Spring Cloud Gateway example with in-memory rate limiting using Bucket4j, no Lombok, and application.properties format.

⸻

✅ Project Structure

spring-cloud-gateway-rate-limit/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── com/example/gateway/
        │       ├── GatewayApplication.java
        │       └── RateLimiterGatewayFilterFactory.java
        └── resources/
            └── application.properties



⸻

🧾 1. pom.xml

<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>spring-cloud-gateway-rate-limit</artifactId>
    <version>1.0.0</version>
    <name>Spring Cloud Gateway Rate Limiting Without Redis</name>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2022.0.4</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>

        <dependency>
            <groupId>com.github.vladimir-bukhtoyarov</groupId>
            <artifactId>bucket4j-core</artifactId>
            <version>7.6.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
    </dependencies>

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

</project>



⸻

🌐 2. GatewayApplication.java

package com.example.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}



⸻

🛡️ 3. RateLimiterGatewayFilterFactory.java

package com.example.gateway;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Refill;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimiterGatewayFilterFactory extends AbstractGatewayFilterFactory<RateLimiterGatewayFilterFactory.Config> {

    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();

    public RateLimiterGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String clientIp = getClientIp(exchange);
            Bucket bucket = cache.computeIfAbsent(clientIp, k -> Bucket4j.builder()
                    .addLimit(Bandwidth.classic(
                            config.getCapacity(),
                            Refill.greedy(config.getRefillTokens(), Duration.ofSeconds(config.getRefillDuration()))
                    ))
                    .build());

            if (bucket.tryConsume(1)) {
                return chain.filter(exchange);
            } else {
                exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                return exchange.getResponse().setComplete();
            }
        };
    }

    private String getClientIp(ServerWebExchange exchange) {
        return exchange.getRequest()
                .getHeaders()
                .getFirst("X-Forwarded-For") != null ?
                exchange.getRequest().getHeaders().getFirst("X-Forwarded-For") :
                exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
    }

    public static class Config {
        private int capacity;
        private int refillTokens;
        private int refillDuration;

        public int getCapacity() {
            return capacity;
        }

        public void setCapacity(int capacity) {
            this.capacity = capacity;
        }

        public int getRefillTokens() {
            return refillTokens;
        }

        public void setRefillTokens(int refillTokens) {
            this.refillTokens = refillTokens;
        }

        public int getRefillDuration() {
            return refillDuration;
        }

        public void setRefillDuration(int refillDuration) {
            this.refillDuration = refillDuration;
        }
    }
}



⸻

⚙️ 4. application.properties

server.port=8080

spring.cloud.gateway.routes[0].id=rate_limited_route
spring.cloud.gateway.routes[0].uri=http://httpbin.org
spring.cloud.gateway.routes[0].predicates[0]=Path=/get
spring.cloud.gateway.routes[0].filters[0].name=RateLimiter
spring.cloud.gateway.routes[0].filters[0].args.capacity=5
spring.cloud.gateway.routes[0].filters[0].args.refillTokens=5
spring.cloud.gateway.routes[0].filters[0].args.refillDuration=10



⸻

▶️ How It Works
	•	This config limits each IP address to 5 requests every 10 seconds.
	•	The rate limiting logic is in-memory, so no Redis is needed.
	•	It’s ideal for single-instance deployments or simple setups.

Would you like this as a downloadable .zip or GitHub repo link?
