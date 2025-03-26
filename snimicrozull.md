Okay, implementing SNI (Server Name Indication) alongside mTLS (Mutual TLS or two-way SSL) in your existing Spring Boot Zuul API Gateway requires programmatic configuration of the embedded web server, as Spring Boot's default `server.ssl.*` properties don't support SNI.

Since you mentioned Zuul and Eureka in the same service, this likely implies you're using **Spring Cloud Zuul 1**, which is now in maintenance mode. The principles are similar for newer gateways like Spring Cloud Gateway, but the implementation details (especially the `WebServerFactoryCustomizer` specifics if using Netty instead of Tomcat) would differ. This guide assumes you are using **Zuul 1 with the default embedded Tomcat**.

**Goal:** Your Zuul gateway should:
1.  Listen on HTTPS (e.g., port 443).
2.  Present the `abc.com` server certificate when accessed via `https://abc.com`.
3.  Present the `xyz.com` server certificate when accessed via `https://xyz.com`.
4.  In *both* cases, require connecting clients to present a valid client certificate signed by a CA listed in your truststore (mTLS).

**Prerequisites:**

1.  **Existing Zuul Project:** Your Spring Boot app with `spring-cloud-starter-netflix-zuul` dependency.
2.  **Java Development Kit (JDK)**.
3.  **Maven or Gradle**.
4.  **Certificates:**
    *   **Server Keystore for `abc.com`:** Contains the private key and certificate for `abc.com`. (e.g., `abc.com.p12`)
    *   **Server Keystore for `xyz.com`:** Contains the private key and certificate for `xyz.com`. (e.g., `xyz.com.p12`)
    *   **Client Truststore:** Contains the Certificate Authority (CA) certificate(s) that signed the *client certificates* you want to allow. This is used to *validate* incoming client certs. (e.g., `client-truststore.p12`)
    *   **(For Testing):** A valid client certificate and its private key, signed by one of the CAs in your `client-truststore.p12`. (e.g., `client.p12` or `client.crt` + `client.key`)

---

**Step 1: Place Certificates and Truststore**

Place your keystore (`abc.com.p12`, `xyz.com.p12`) and truststore (`client-truststore.p12`) files where the Zuul application can access them. Options:
*   **Classpath:** `src/main/resources/certs/` (bundled in JAR)
*   **Filesystem:** `/etc/ssl/zuul/` (external, more flexible)

---

**Step 2: Implement SNI + mTLS Configuration**

1.  Create a new `@Configuration` class (e.g., `SniMtlsTomcatConfiguration`).
2.  Implement `WebServerFactoryCustomizer<TomcatServletWebServerFactory>` to programmatically configure Tomcat.

```java
package com.example.zuulgateway.config; // Adjust package name

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
public class SniMtlsTomcatConfiguration {

    // --- General SSL Port ---
    @Value("${server.sni.port:443}")
    private int sniPort;

    // --- Server Certificate for abc.com ---
    @Value("${server.sni.abc.hostname:abc.com}")
    private String abcHostname;
    @Value("${server.sni.abc.keystore.path}") // e.g., classpath:certs/abc.com.p12
    private String abcKeystorePath;
    @Value("${server.sni.abc.keystore.password}")
    private String abcKeystorePassword;
    @Value("${server.sni.abc.keystore.type:PKCS12}")
    private String abcKeystoreType;
    // Optional: Key alias if keystore has multiple entries
    // @Value("${server.sni.abc.key.alias:abc_alias}")
    // private String abcKeyAlias;

    // --- Server Certificate for xyz.com ---
    @Value("${server.sni.xyz.hostname:xyz.com}")
    private String xyzHostname;
    @Value("${server.sni.xyz.keystore.path}") // e.g., file:/etc/ssl/zuul/xyz.com.p12
    private String xyzKeystorePath;
    @Value("${server.sni.xyz.keystore.password}")
    private String xyzKeystorePassword;
    @Value("${server.sni.xyz.keystore.type:PKCS12}")
    private String xyzKeystoreType;
    // Optional: Key alias
    // @Value("${server.sni.xyz.key.alias:xyz_alias}")
    // private String xyzKeyAlias;

    // --- Mutual TLS (Client Authentication) ---
    @Value("${server.sni.mtls.truststore.path}") // e.g., classpath:certs/client-truststore.p12
    private String clientTruststorePath;
    @Value("${server.sni.mtls.truststore.password}")
    private String clientTruststorePassword;
    @Value("${server.sni.mtls.truststore.type:PKCS12}")
    private String clientTruststoreType;
    // Controls mTLS requirement: "required" (like NEED) or "optional" (like WANT)
    @Value("${server.sni.mtls.client.auth:required}")
    private String clientAuth;

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> sniMtlsCustomizer() {
        return factory -> {
            // CRITICAL: Disable Spring Boot's default SSL config via properties
            // Ensure server.ssl.enabled=false in application.properties/yml

            Connector httpsConnector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
            httpsConnector.setScheme("https");
            httpsConnector.setSecure(true);
            httpsConnector.setPort(sniPort);

            Http11NioProtocol protocol = (Http11NioProtocol) httpsConnector.getProtocolHandler();
            protocol.setSSLEnabled(true);

            try {
                String truststoreFilePath = ResourceUtils.getURL(clientTruststorePath).getPath();

                // --- Configure abc.com with mTLS ---
                SSLHostConfig sslHostConfigAbc = new SSLHostConfig();
                sslHostConfigAbc.setHostName(abcHostname);
                // Server cert config
                sslHostConfigAbc.setCertificateKeystoreFile(ResourceUtils.getURL(abcKeystorePath).getPath());
                sslHostConfigAbc.setCertificateKeystorePassword(abcKeystorePassword);
                sslHostConfigAbc.setCertificateKeystoreType(abcKeystoreType);
                // sslHostConfigAbc.setCertificateKeyAlias(abcKeyAlias); // If using alias

                // mTLS config (same for all hosts on this connector)
                sslHostConfigAbc.setTruststoreFile(truststoreFilePath);
                sslHostConfigAbc.setTruststorePassword(clientTruststorePassword);
                sslHostConfigAbc.setTruststoreType(clientTruststoreType);
                sslHostConfigAbc.setCertificateVerification(clientAuth); // "required" or "optional"

                // Add other settings (protocols, ciphers) if needed, apply consistently
                // sslHostConfigAbc.setSslProtocol("TLS");
                // sslHostConfigAbc.setProtocols("TLSv1.2,TLSv1.3");
                // sslHostConfigAbc.setCiphers("...");

                protocol.addSslHostConfig(sslHostConfigAbc);


                // --- Configure xyz.com with mTLS ---
                SSLHostConfig sslHostConfigXyz = new SSLHostConfig();
                sslHostConfigXyz.setHostName(xyzHostname);
                // Server cert config
                sslHostConfigXyz.setCertificateKeystoreFile(ResourceUtils.getURL(xyzKeystorePath).getPath());
                sslHostConfigXyz.setCertificateKeystorePassword(xyzKeystorePassword);
                sslHostConfigXyz.setCertificateKeystoreType(xyzKeystoreType);
                // sslHostConfigXyz.setCertificateKeyAlias(xyzKeyAlias); // If using alias

                // mTLS config (points to the same truststore and settings)
                sslHostConfigXyz.setTruststoreFile(truststoreFilePath);
                sslHostConfigXyz.setTruststorePassword(clientTruststorePassword);
                sslHostConfigXyz.setTruststoreType(clientTruststoreType);
                sslHostConfigXyz.setCertificateVerification(clientAuth);

                // Add other settings (protocols, ciphers) consistently
                // sslHostConfigXyz.setSslProtocol("TLS");
                // sslHostConfigXyz.setProtocols("TLSv1.2,TLSv1.3");
                // sslHostConfigXyz.setCiphers("...");

                protocol.addSslHostConfig(sslHostConfigXyz);


                // --- Configure Default Host with mTLS (Optional but Recommended) ---
                SSLHostConfig defaultSslHostConfig = new SSLHostConfig();
                defaultSslHostConfig.setHostName("_default_");
                // Use one of the server certs (e.g., abc.com) as default
                defaultSslHostConfig.setCertificateKeystoreFile(ResourceUtils.getURL(abcKeystorePath).getPath());
                defaultSslHostConfig.setCertificateKeystorePassword(abcKeystorePassword);
                defaultSslHostConfig.setCertificateKeystoreType(abcKeystoreType);
                // defaultSslHostConfig.setCertificateKeyAlias(abcKeyAlias);

                // mTLS config (same truststore and settings)
                defaultSslHostConfig.setTruststoreFile(truststoreFilePath);
                defaultSslHostConfig.setTruststorePassword(clientTruststorePassword);
                defaultSslHostConfig.setTruststoreType(clientTruststoreType);
                defaultSslHostConfig.setCertificateVerification(clientAuth);

                // Other settings (protocols, ciphers) consistently
                // defaultSslHostConfig.setSslProtocol("TLS");
                // defaultSslHostConfig.setProtocols("TLSv1.2,TLSv1.3");
                // defaultSslHostConfig.setCiphers("...");

                protocol.addSslHostConfig(defaultSslHostConfig, true); // Mark as default
                protocol.setDefaultSSLHostConfigName("_default_");

                // Add the fully configured connector
                factory.addAdditionalTomcatConnectors(httpsConnector);

            } catch (FileNotFoundException e) {
                throw new RuntimeException("Failed to find keystore or truststore file specified in properties", e);
            }
        };
    }
}
```

---

**Step 3: Configure Application Properties**

1.  Open `src/main/resources/application.properties` (or `.yml`).
2.  **Disable default Spring Boot SSL** and add properties referenced in the configuration class.

```properties
# --- Spring Boot Basic Config ---
# server.port=8765 # Optional: Set a separate HTTP port if needed
spring.application.name=zuul-gateway

# --- Eureka Client Config (If used) ---
# eureka.client.serviceUrl.defaultZone=http://eureka-server:8761/eureka/
# eureka.instance.preferIpAddress=true

# --- Zuul Routes (Existing configuration remains the same) ---
# zuul.routes.service-a.path=/service-a/**
# zuul.routes.service-a.serviceId=service-a-app
# ... other routes ...

# --- CRITICAL: Disable Default Spring Boot SSL ---
# This prevents conflicts with our custom connector
server.ssl.enabled=false

# --- SNI + mTLS Connector Configuration ---
server.sni.port=443 # Public HTTPS port for Zuul

# Server Keystore for abc.com
server.sni.abc.hostname=abc.com
server.sni.abc.keystore.path=classpath:certs/abc.com.p12
server.sni.abc.keystore.password=your_abc_keystore_password
server.sni.abc.keystore.type=PKCS12
# server.sni.abc.key.alias=abc_alias # Optional

# Server Keystore for xyz.com
server.sni.xyz.hostname=xyz.com
server.sni.xyz.keystore.path=file:/etc/ssl/zuul/xyz.com.p12
server.sni.xyz.keystore.password=your_xyz_keystore_password
server.sni.xyz.keystore.type=PKCS12
# server.sni.xyz.key.alias=xyz_alias # Optional

# Client Truststore (for validating incoming client certs)
server.sni.mtls.truststore.path=classpath:certs/client-truststore.p12
server.sni.mtls.truststore.password=your_client_truststore_password
server.sni.mtls.truststore.type=PKCS12
server.sni.mtls.client.auth=required # Or "optional"

# --- Optional: Logging for SSL Debugging ---
# logging.level.org.apache.coyote.http11.Http11NioProtocol=DEBUG
# logging.level.org.apache.tomcat.util.net.NioEndpoint=DEBUG
# logging.level.org.apache.tomcat.util.net.jsse.JSSEUtil=DEBUG
# javax.net.debug=ssl,handshake # Enable JVM SSL debugging (very verbose!) - Pass as JVM arg: -Djavax.net.debug=ssl,handshake
```

---

**Step 4: Build and Run**

1.  Build the Zuul gateway application:
    ```bash
    mvn clean package # or ./gradlew clean build
    ```
2.  Run the application:
    ```bash
    java -jar target/your-zuul-gateway-app.jar
    # Or pass JVM args if needed, e.g., for SSL debugging:
    # java -Djavax.net.debug=ssl,handshake -jar target/your-zuul-gateway-app.jar
    ```

---

**Step 5: Testing (Crucial)**

You need to test both SNI (correct server cert) and mTLS (client cert required).

1.  **Local Host Mapping:** Edit your `/etc/hosts` file (or Windows equivalent) as before:
    ```
    127.0.0.1 abc.com
    127.0.0.1 xyz.com
    ```
2.  **Use `curl`:** `curl` is excellent for this. You'll need your client certificate (`client.crt` or `client.p12`) and private key (`client.key`).

    *   **Test `abc.com` with valid client cert:**
        ```bash
        # If using separate .crt and .key files:
        curl -v --resolve abc.com:443:127.0.0.1 \
             --cert /path/to/your/client.crt \
             --key /path/to/your/client.key \
             # Optional: If server cert isn't trusted by default, specify CA: --cacert /path/to/server_ca.crt
             https://abc.com/some/zuul/route

        # If using a PKCS12 file for client cert+key:
        curl -v --resolve abc.com:443:127.0.0.1 \
             --cert /path/to/your/client.p12:your_client_p12_password \
             # Optional: --cacert /path/to/server_ca.crt
             https://abc.com/some/zuul/route
        ```
        *   **Check:**
            *   Verbose output (`-v`) shows the server certificate details – verify it's the `abc.com` certificate.
            *   TLS handshake completes successfully.
            *   You get the expected response from the downstream service via Zuul.

    *   **Test `xyz.com` with valid client cert:**
        ```bash
        # Using separate .crt and .key files:
        curl -v --resolve xyz.com:443:127.0.0.1 \
             --cert /path/to/your/client.crt \
             --key /path/to/your/client.key \
             # Optional: --cacert /path/to/server_ca.crt
             https://xyz.com/some/other/zuul/route

        # Using PKCS12:
        curl -v --resolve xyz.com:443:127.0.0.1 \
             --cert /path/to/your/client.p12:your_client_p12_password \
             # Optional: --cacert /path/to/server_ca.crt
             https://xyz.com/some/other/zuul/route
        ```
        *   **Check:**
            *   Verbose output shows the server certificate details – verify it's the `xyz.com` certificate.
            *   TLS handshake completes successfully.
            *   You get the expected response.

    *   **Test `abc.com` WITHOUT client cert (mTLS check):**
        ```bash
        curl -v --resolve abc.com:443:127.0.0.1 https://abc.com/some/zuul/route
        ```
        *   **Check:** The TLS handshake should **fail**. `curl` might report an error like "OpenSSL SSL_connect: SSL_ERROR_SYSCALL" or "alert certificate required" depending on the exact phase of failure. The connection should *not* complete.

    *   **Test `xyz.com` WITHOUT client cert (mTLS check):**
        ```bash
        curl -v --resolve xyz.com:443:127.0.0.1 https://xyz.com/some/other/zuul/route
        ```
        *   **Check:** The TLS handshake should **fail**.

---

**Step 6: Deployment Considerations**

*   **Secrets Management:** Securely manage passwords for keystores and truststores (use Spring Cloud Config Server with Vault, environment variables, k8s secrets, etc., not hardcoding in properties files for production).
*   **Certificate Rotation:** Plan how you will rotate server certificates and update the client truststore when needed. Externalizing the files (Option B in Step 1) makes this easier than rebuilding the JAR.
*   **Process Management:** Use `systemd`, Docker, Kubernetes, etc., to manage the Zuul gateway process.

This detailed guide allows you to configure your Zuul 1 (Tomcat-based) gateway for both SNI and mTLS, ensuring the correct server certificate is presented while also enforcing client certificate authentication. Remember to test thoroughly, especially the mTLS failure cases.