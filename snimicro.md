Okay, here is a detailed step-by-step guide for implementing **Option 1: The "In-Process Proxy" using Spring Boot**. This involves creating a separate Spring Boot application (`sni-wrapper`) that handles SNI and proxies requests to your existing, unmodified WAR application (`original.war`) running in a separate process.

**Goal:** Access `https://abc.com` (gets cert A) and `https://xyz.com` (gets cert B), both served by `original.war`, using the `sni-wrapper` as the entry point.

**Prerequisites:**

1.  **Java Development Kit (JDK):** Make sure you have a compatible JDK installed (usually Java 11 or 17+ for modern Spring Boot).
2.  **Maven or Gradle:** Your build tool for the Spring Boot project.
3.  **Original WAR File:** (`original.war`) - Unmodified.
4.  **SSL Certificates & Keys:**
    *   Certificate and private key for `abc.com`.
    *   Certificate and private key for `xyz.com`.
    *   These should ideally be in a format Java can easily use, like PKCS12 (`.p12`, `.pfx`) keystores. If you have separate `.crt` and `.key` files, you can convert them using `openssl`.
5.  **Ability to Run Two Processes:** Your deployment environment must allow you to run two separate Java processes concurrently.

---

**Step 1: Create the `sni-wrapper` Spring Boot Project**

1.  Go to [Spring Initializr](https://start.spring.io/).
2.  Choose **Maven** or **Gradle** Project.
3.  Select **Java**.
4.  Choose a recent stable Spring Boot version.
5.  Define your project metadata (Group, Artifact ID - e.g., `com.example`, `sni-wrapper`).
6.  Select **Jar** packaging.
7.  Add Dependencies:
    *   **Spring Web:** (Provides embedded Tomcat and base web functionality)
    *   **Spring WebFlux:** (Needed for `WebClient`, the reactive HTTP client used for proxying)
8.  Click **Generate** and download the project zip.
9.  Unzip the project into your workspace.

---

**Step 2: Place Certificates**

1.  Decide where to store your keystore files (e.g., `abc.com.p12`, `xyz.com.p12`).
    *   **Option A (Classpath):** Place them inside the `sni-wrapper` project's `src/main/resources` directory. This bundles them into the JAR. Simple but less flexible for updates.
    *   **Option B (Filesystem):** Place them on the server's filesystem where the `sni-wrapper` will run (e.g., `/etc/ssl/myapp/`). More flexible for certificate rotation.
2.  Make sure the `sni-wrapper` process will have read permissions for these files.

---

**Step 3: Implement SNI Configuration in `sni-wrapper`**

1.  Create a new configuration class in your `sni-wrapper` project (e.g., `com.example.sniwrapper.config.SniTomcatConfiguration`).
2.  Implement the `WebServerFactoryCustomizer` to programmatically configure Tomcat for SNI.

```java
package com.example.sniwrapper.config;

import org.apache.catalina.connector.Connector;
import org.apache.coyote.http11.Http11NioProtocol;
import org.apache.tomcat.util.net.SSLHostConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.ResourceUtils;

import java.io.FileNotFoundException;

@Configuration
public class SniTomcatConfiguration {

    // Inject properties from application.properties/yml
    @Value("${server.sni.port:443}") // Default to 443 if not specified
    private int sniPort;

    @Value("${server.sni.abc.hostname:abc.com}")
    private String abcHostname;
    @Value("${server.sni.abc.keystore.path}") // e.g., classpath:abc.com.p12 or file:/etc/ssl/myapp/abc.com.p12
    private String abcKeystorePath;
    @Value("${server.sni.abc.keystore.password}")
    private String abcKeystorePassword;
    @Value("${server.sni.abc.keystore.type:PKCS12}")
    private String abcKeystoreType;

    @Value("${server.sni.xyz.hostname:xyz.com}")
    private String xyzHostname;
    @Value("${server.sni.xyz.keystore.path}")
    private String xyzKeystorePath;
    @Value("${server.sni.xyz.keystore.password}")
    private String xyzKeystorePassword;
    @Value("${server.sni.xyz.keystore.type:PKCS12}")
    private String xyzKeystoreType;

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> sniCustomizer() {
        return factory -> {
            // IMPORTANT: Ensure Spring Boot's default SSL is disabled in application.properties
            // server.ssl.enabled=false

            Connector httpsConnector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
            httpsConnector.setScheme("https");
            httpsConnector.setSecure(true);
            httpsConnector.setPort(sniPort); // Use the configured port

            Http11NioProtocol protocol = (Http11NioProtocol) httpsConnector.getProtocolHandler();
            protocol.setSSLEnabled(true);

            try {
                // --- Configure abc.com ---
                SSLHostConfig sslHostConfigAbc = new SSLHostConfig();
                sslHostConfigAbc.setHostName(abcHostname);
                // Use ResourceUtils to handle classpath: and file: prefixes
                sslHostConfigAbc.setCertificateKeystoreFile(ResourceUtils.getURL(abcKeystorePath).getPath());
                sslHostConfigAbc.setCertificateKeystorePassword(abcKeystorePassword);
                sslHostConfigAbc.setCertificateKeystoreType(abcKeystoreType);
                // Add other settings if needed: protocols, cipherSuite, etc.
                // sslHostConfigAbc.setSslProtocol("TLS");
                // sslHostConfigAbc.setProtocols("TLSv1.2,TLSv1.3");
                protocol.addSslHostConfig(sslHostConfigAbc);

                // --- Configure xyz.com ---
                SSLHostConfig sslHostConfigXyz = new SSLHostConfig();
                sslHostConfigXyz.setHostName(xyzHostname);
                sslHostConfigXyz.setCertificateKeystoreFile(ResourceUtils.getURL(xyzKeystorePath).getPath());
                sslHostConfigXyz.setCertificateKeystorePassword(xyzKeystorePassword);
                sslHostConfigXyz.setCertificateKeystoreType(xyzKeystoreType);
                protocol.addSslHostConfig(sslHostConfigXyz);

                // --- Configure Default (Optional) ---
                // Useful if accessed by IP or non-matching hostname
                SSLHostConfig defaultSslHostConfig = new SSLHostConfig();
                defaultSslHostConfig.setHostName("_default_"); // Tomcat's default marker
                // Use one of the certs (e.g., abc.com) or a dedicated default cert
                defaultSslHostConfig.setCertificateKeystoreFile(ResourceUtils.getURL(abcKeystorePath).getPath());
                defaultSslHostConfig.setCertificateKeystorePassword(abcKeystorePassword);
                defaultSslHostConfig.setCertificateKeystoreType(abcKeystoreType);
                protocol.addSslHostConfig(defaultSslHostConfig, true); // Mark as default
                protocol.setDefaultSSLHostConfigName("_default_"); // Tell Tomcat which one is default

                // Add the connector
                factory.addAdditionalTomcatConnectors(httpsConnector);

            } catch (FileNotFoundException e) {
                // Handle error appropriately - fail startup if certs are missing
                throw new RuntimeException("Failed to find keystore file specified in properties", e);
            }
        };
    }
}
```

---

**Step 4: Implement Proxying Logic in `sni-wrapper`**

1.  Create a `@RestController` to handle all incoming requests and forward them.
2.  Use `WebClient` to make the onward request to `original.war`.

```java
package com.example.sniwrapper.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.reactive.function.BodyInserters;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import javax.annotation.PostConstruct;
import java.net.URI;
import java.net.URISyntaxException;

@RestController
public class ProxyController {

    private static final Logger log = LoggerFactory.getLogger(ProxyController.class);

    @Value("${original.app.url}") // e.g., http://localhost:8080
    private String originalAppUrl;

    private WebClient webClient;

    @PostConstruct
    public void init() {
        // Configure WebClient
        // If original.war runs on HTTPS, you might need to configure SSL context
        // here to trust its certificate, especially if it's self-signed.
        this.webClient = WebClient.builder()
                .baseUrl(originalAppUrl)
                // Add SSL context configuration if target is HTTPS w/ self-signed cert
                .build();
        log.info("Proxy configured to forward requests to: {}", originalAppUrl);
    }

    @RequestMapping("/**") // Catch all requests
    public Mono<Void> proxyRequest(ServerHttpRequest request, ServerHttpResponse response) {
        URI originalUri = request.getURI();
        HttpMethod method = request.getMethod();

        // Construct the target URL
        String path = originalUri.getPath();
        String query = originalUri.getQuery();
        String targetUriString = path + (query != null ? "?" + query : "");

        log.debug("Proxying {} request for: {}", method, targetUriString);

        // Prepare the downstream request using WebClient
        WebClient.RequestBodySpec requestBodySpec = webClient
                .method(method)
                .uri(targetUriString)
                .headers(headers -> {
                    copyHeaders(request.getHeaders(), headers);
                    // Add/Modify specific headers if needed
                    headers.set(HttpHeaders.HOST, getHostFromTargetUrl()); // Set Host to target app's host
                    headers.set("X-Forwarded-Proto", originalUri.getScheme());
                    headers.set("X-Forwarded-Host", originalUri.getHost());
                    if (originalUri.getPort() != -1) {
                         headers.set("X-Forwarded-Port", String.valueOf(originalUri.getPort()));
                    }
                    // Add X-Forwarded-For if behind multiple proxies - requires careful handling
                    // headers.set("X-Forwarded-For", request.getRemoteAddress().getAddress().getHostAddress());
                });

        // Handle request body
        WebClient.RequestHeadersSpec<?> requestHeadersSpec;
        if (requiresBody(method)) {
            requestHeadersSpec = requestBodySpec.body(BodyInserters.fromDataBuffers(request.getBody()));
        } else {
            requestHeadersSpec = requestBodySpec;
        }

        // Execute the request and relay the response
        return requestHeadersSpec.exchangeToMono(clientResponse -> {
            response.setStatusCode(clientResponse.statusCode());
            copyHeaders(clientResponse.headers().asHttpHeaders(), response.getHeaders());
            log.debug("Received response {} from target", clientResponse.statusCode());
            // Stream the response body back to the original client
            return response.writeWith(clientResponse.bodyToFlux(DataBuffer.class));
        }).doOnError(error -> log.error("Error during proxying", error));
    }

    private void copyHeaders(HttpHeaders source, HttpHeaders target) {
        source.forEach((key, values) -> {
            // Avoid copying hop-by-hop headers like Connection, Keep-Alive, etc.
            // You might need a more sophisticated filtering logic here.
            if (!HttpHeaders.CONNECTION.equalsIgnoreCase(key) &&
                !HttpHeaders.HOST.equalsIgnoreCase(key) && // Host is set separately
                !"Keep-Alive".equalsIgnoreCase(key) &&
                !"Transfer-Encoding".equalsIgnoreCase(key) &&
                !"TE".equalsIgnoreCase(key) &&
                !"Upgrade".equalsIgnoreCase(key)) {
                 target.addAll(key, values);
            }
        });
    }

     private boolean requiresBody(HttpMethod method) {
        return method == HttpMethod.POST || method == HttpMethod.PUT || method == HttpMethod.PATCH;
     }

     private String getHostFromTargetUrl() {
         try {
             return new URI(originalAppUrl).getHost();
         } catch (URISyntaxException e) {
             log.error("Invalid original.app.url: {}", originalAppUrl, e);
             return "localhost"; // Fallback
         }
     }
}

```
*   **Alternative:** Consider using **Spring Cloud Gateway** within the `sni-wrapper`. It's specifically designed for building API gateways and proxies, simplifying routing and header manipulation significantly compared to the manual `WebClient` approach above. You would replace `spring-boot-starter-web` and `spring-boot-starter-webflux` with `spring-cloud-starter-gateway` and configure routes in `application.yml`. This would likely require using Netty instead of Tomcat, so the SNI configuration would need adapting (`WebServerFactoryCustomizer<NettyReactiveWebServerFactory>`).

---

**Step 5: Configure `sni-wrapper` Application Properties**

1.  Open `src/main/resources/application.properties` (or `.yml`).
2.  Add the necessary configuration:

```properties
# Spring Boot server config for the wrapper itself
# Let Spring Boot manage HTTP port if needed for health checks, etc. (optional)
# server.port=9090

# --- IMPORTANT: Disable Spring Boot's default SSL handling ---
server.ssl.enabled=false

# --- SNI Connector Configuration (used by SniTomcatConfiguration) ---
server.sni.port=443 # The public HTTPS port the wrapper listens on

# Keystore paths can use classpath: or file: prefixes
server.sni.abc.hostname=abc.com
server.sni.abc.keystore.path=classpath:certs/abc.com.p12 # Example using classpath
server.sni.abc.keystore.password=your_abc_password
server.sni.abc.keystore.type=PKCS12

server.sni.xyz.hostname=xyz.com
server.sni.xyz.keystore.path=file:/etc/ssl/myapp/xyz.com.p12 # Example using filesystem
server.sni.xyz.keystore.password=your_xyz_password
server.sni.xyz.keystore.type=PKCS12

# --- Proxy Target Configuration ---
# URL where the original WAR application is running
# Use HTTP if original.war runs on HTTP, HTTPS if it runs on HTTPS
original.app.url=http://localhost:8080

# Optional: Logging levels
logging.level.com.example.sniwrapper=DEBUG # Set to INFO in production
logging.level.org.springframework.web=INFO
logging.level.reactor.netty=INFO
logging.level.org.apache.tomcat=INFO # See Tomcat SSL logs if needed
```

---

**Step 6: Prepare and Run `original.war`**

1.  **Goal:** Run `original.war` on a *different port* than the `sni-wrapper` (e.g., 8080) and ideally on **HTTP** to simplify proxying.
2.  Since you cannot modify the WAR, you need to configure its embedded server externally at runtime. Common ways for Spring Boot WARs:
    *   **Command Line Arguments:**
        ```bash
        java -jar /path/to/original.war --server.port=8080 --server.ssl.enabled=false
        ```
    *   **Environment Variables:**
        ```bash
        export SERVER_PORT=8080
        export SERVER_SSL_ENABLED=false
        java -jar /path/to/original.war
        ```
    *   **External `application.properties`:** Place an `application.properties` file in the *same directory* as `original.war` or in a `./config/` subdirectory:
        ```properties
        # config/application.properties or ./application.properties
        server.port=8080
        server.ssl.enabled=false
        ```
        Then run: `java -jar /path/to/original.war`
3.  **If running `original.war` on HTTP is impossible:** Run it on its usual HTTPS port (e.g., 8443) using its existing certificate. Update `original.app.url` in the wrapper's properties to `https://localhost:8443`. You might need to configure the `WebClient` in `ProxyController` to trust the certificate used by `original.war` (especially if self-signed), which adds complexity.

---

**Step 7: Build and Run Both Applications**

1.  **Build the Wrapper:**
    ```bash
    cd /path/to/sni-wrapper
    mvn clean package # or ./gradlew clean build
    ```
    This creates `target/sni-wrapper-0.0.1-SNAPSHOT.jar`.
2.  **Start `original.war`:** (Using one of the methods from Step 6)
    ```bash
    # Example using command line args
    java -jar /path/to/original.war --server.port=8080 --server.ssl.enabled=false &
    # The '&' runs it in the background (on Linux/macOS)
    ```
    Wait for it to start fully. Check its logs.
3.  **Start `sni-wrapper.jar`:**
    ```bash
    java -jar /path/to/sni-wrapper/target/sni-wrapper-0.0.1-SNAPSHOT.jar
    # Or pass properties via command line if needed:
    # java -jar ... --original.app.url=http://127.0.0.1:8080 --server.sni.abc.keystore.password=...
    ```
    Check its logs for SNI configuration and proxy readiness.

---

**Step 8: Testing**

1.  **Local Host Mapping:** You need to trick your local machine into resolving `abc.com` and `xyz.com` to `127.0.0.1` where the `sni-wrapper` is listening.
    *   Edit your `/etc/hosts` file (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows - requires admin). Add:
        ```
        127.0.0.1 abc.com
        127.0.0.1 xyz.com
        ```
2.  **Use `curl`:** (Disable system CA checks if using self-signed certs, `-k`)
    *   **Test abc.com:**
        ```bash
        curl -v --resolve abc.com:443:127.0.0.1 https://abc.com/some/path/in/original/app
        ```
        *   Check the output (`* Server certificate:` section) to verify the `abc.com` certificate is presented.
        *   Verify you get the expected response content from `original.war`.
    *   **Test xyz.com:**
        ```bash
        curl -v --resolve xyz.com:443:127.0.0.1 https://xyz.com/some/other/path
        ```
        *   Check the output to verify the `xyz.com` certificate is presented.
        *   Verify you get the expected response content from `original.war`.
3.  **Use a Browser:** Access `https://abc.com` and `https://xyz.com`. Check the certificate details shown by the browser for each domain. You might need to accept security warnings if using self-signed certificates.

---

**Step 9: Deployment Considerations**

1.  **Process Management:** Use a proper process manager (`systemd`, `supervisorctl`, `docker-compose`) to manage both the `original.war` and `sni-wrapper.jar` processes. Ensure they start in the correct order (original app first ideally) and are restarted if they fail.
2.  **Docker Compose Example:**
    ```yaml
    # docker-compose.yml
    version: '3.8'
    services:
      original-app:
        image: openjdk:17-jdk-slim # Or your required Java base image
        container_name: original-app
        volumes:
          - ./original.war:/app/original.war
          # Mount external config if needed: - ./config/original-app:/app/config
        command: >
          java
          -jar /app/original.war
          --server.port=8080
          --server.ssl.enabled=false
          # Add other JVM opts if necessary
        # No ports exposed externally, only to the sni-wrapper network

      sni-wrapper:
        image: openjdk:17-jdk-slim
        container_name: sni-wrapper
        depends_on:
          - original-app
        ports:
          - "443:443" # Expose the public HTTPS port
        volumes:
          - ./sni-wrapper/target/sni-wrapper-0.0.1-SNAPSHOT.jar:/app/sni-wrapper.jar
          # Mount certificates from host filesystem:
          - /etc/ssl/myapp:/etc/ssl/myapp
          # Or mount a config file: - ./config/sni-wrapper:/app/config
        environment:
          # Pass config via environment variables (or use mounted config file)
          - ORIGINAL_APP_URL=http://original-app:8080 # Use Docker's internal DNS
          - SERVER_SNI_PORT=443
          - SERVER_SNI_ABC_HOSTNAME=abc.com
          - SERVER_SNI_ABC_KEYSTORE_PATH=file:/etc/ssl/myapp/abc.com.p12
          - SERVER_SNI_ABC_KEYSTORE_PASSWORD=your_abc_password
          - SERVER_SNI_XYZ_HOSTNAME=xyz.com
          - SERVER_SNI_XYZ_KEYSTORE_PATH=file:/etc/ssl/myapp/xyz.com.p12
          - SERVER_SNI_XYZ_KEYSTORE_PASSWORD=your_xyz_password
        command: >
          java
          -jar /app/sni-wrapper.jar
          # Add other JVM opts if necessary
    ```
3.  **Certificates:** Ensure certificates are securely stored and accessible by the `sni-wrapper` process/container. Manage their rotation.
4.  **Firewall:** Ensure port 443 (or your chosen `server.sni.port`) is open on the host machine's firewall.
5.  **Resource Allocation:** Monitor CPU and memory usage for both JVM processes.

This detailed guide provides a viable path under your constraints, but remember it involves managing two processes and introduces a proxy layer. Test thoroughly.