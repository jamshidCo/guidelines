# Logging Guidelines

## 1. Use SLF4J API for Logging
* Don't directly use implementation-specific loggers like `ch.qos.logback.classic.Logger`.
  ```java
  // DON'T
  // import ch.qos.logback.classic.Logger;
  // import org.slf4j.LoggerFactory; // OK, but using wrong type below
  private final Logger log = (ch.qos.logback.classic.Logger) LoggerFactory.getLogger(MyService.class); // Ties to Logback
  ```
* Do use the `org.slf4j.Logger` interface provided by SLF4J.
  ```java
  // DO
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;

  @Service
  public class MyService {
      private static final Logger log = LoggerFactory.getLogger(MyService.class); // Standard SLF4J
      // Or using instance specific logger:
      // private final Logger log = LoggerFactory.getLogger(getClass());
  }
  ```
* Why? Decouples your code from the specific logging implementation (Logback, Log4j2, etc.), making it easier to switch later if needed. Spring Boot manages this abstraction well.
* Want to learn more? [SLF4J User Manual](http://www.slf4j.org/manual.html), [Spring Boot Logging Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)

## 2. Instantiate Loggers Correctly and Efficiently
* Don't create logger instances inside methods or repeatedly.
  ```java
  // DON'T
  public void processData(String data) {
      // Inefficient and non-standard
      Logger localLog = LoggerFactory.getLogger(MyProcessor.class);
      localLog.info("Processing data...");
  }
  ```
* Do declare a `static final` logger per class (most common) or a `final` instance logger. Use `YourClass.class` or `getClass()` as the argument.
  ```java
  // DO (Class-level logger - recommended)
  @Component
  public class DataProcessor {
      private static final Logger log = LoggerFactory.getLogger(DataProcessor.class);

      public void process(String input) {
          log.info("Starting processing for input: {}", input);
      }
  }

  // DO (Instance-level logger - less common, useful in hierarchies)
  public abstract class BaseHandler {
      protected final Logger log = LoggerFactory.getLogger(getClass()); // Gets subclass logger
  }
  ```
* Why? `static final` is the conventional and generally most performant approach. `LoggerFactory.getLogger` can be reasonably fast, but avoiding repeated calls is cleaner. Using `getClass()` correctly identifies the source class, even in inheritance.
* Want to learn more? [SLF4J FAQ: Performance](http://www.slf4j.org/faq.html#performance)

## 3. Use Parameterized Logging (`{}`)
* Don't use string concatenation for log messages with variables.
  ```java
  // DON'T
  String userId = "user123";
  int count = 42;
  // Inefficient if DEBUG is disabled - creates string anyway
  log.debug("Processing finished for user " + userId + " with count: " + count);
  ```
* Do use SLF4J's `{}` placeholders for variables.
  ```java
  // DO
  String userId = "user123";
  int count = 42;
  // Arguments evaluated only if DEBUG level is enabled
  log.debug("Processing finished for user {} with count: {}", userId, count);
  ```
* Why? Avoids unnecessary string creation and concatenation if the log level is disabled, improving performance. It's also generally more readable.
* Want to learn more? [SLF4J FAQ: Parameterized Logging](http://www.slf4j.org/faq.html#logging_performance)

## 4. Use Appropriate Log Levels Consistently
* Don't log everything at `INFO` or use levels inconsistently (e.g., `ERROR` for validation failures).
  ```java
  // DON'T
  log.info("Entering method processOrder with orderId: {}", orderId); // Too verbose for INFO
  try {
      validate(order);
  } catch (ValidationException e) {
      log.error("Order validation failed: {}", e.getMessage()); // ERROR usually for system issues
  }
  ```
* Do use standard log levels with clear intent:
    *   `ERROR`: Serious system issues preventing normal operation (uncaught exceptions, connection failures). Include stack traces.
    *   `WARN`: Potential problems or unexpected situations that don't stop execution but should be noted (e.g., fallback used, configuration issue).
    *   `INFO`: High-level application lifecycle events (startup, shutdown, major requests processed, significant state changes). Keep volume low in production.
    *   `DEBUG`: Fine-grained information useful for debugging specific flows (method entry/exit, important variable values).
    *   `TRACE`: Most detailed level, typically for tracing execution paths or very low-level diagnostics.
  ```java
  // DO
  log.info("Application started successfully on port {}", port); // INFO: Lifecycle
  log.debug("Processing order {} for user {}", orderId, userId); // DEBUG: Specific flow detail
  try {
    validate(order);
  } catch (ValidationException e) {
    log.warn("Order validation failed for order {}: {}", orderId, e.getMessage()); // WARN: Recoverable issue
  } catch (DatabaseAccessException e) {
    log.error("Failed to save order {} due to database issue", orderId, e); // ERROR: System issue with exception
  }
  ```
* Why? Allows effective filtering of logs based on severity in different environments (e.g., `INFO` in production, `DEBUG` in development). Makes logs meaningful and actionable.
* Want to learn more? [Logback Manual: Levels](https://logback.qos.ch/manual/architecture.html#effectiveLevel), [Spring Boot Logging: Log Levels](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging.log-levels)

## 5. Log Exceptions Correctly (Include Stack Trace)
* Don't log only the exception message (`e.getMessage()`) without the stack trace when catching exceptions, especially for `ERROR` or `WARN`.
  ```java
  // DON'T
  catch (IOException e) {
      // Missing stack trace - makes root cause analysis hard
      log.error("Failed to process file: " + e.getMessage());
  }
  ```
* Do pass the `Throwable` object as the *last* argument in the logging call. The placeholder `{}` is not needed for the exception itself.
  ```java
  // DO
  catch (IOException e) {
      // Stack trace will be included automatically
      log.error("Failed to process file {}", filePath, e);
  }
  catch (SpecificBusinessException e) {
      // Maybe only WARN if it's expected, but still include trace if useful for debugging
       log.warn("Business rule validation failed for request {}: {}", requestId, e.getMessage(), e);
  }
  ```
* Why? The stack trace is crucial for debugging and understanding the origin and context of an error. Logging only the message often hides vital information.
* Want to learn more? [SLF4J FAQ: Exception Logging](http://www.slf4j.org/faq.html#exception_logging)

## 6. Avoid Logging Sensitive Information
* Don't log passwords, credentials, API keys, personal identifiable information (PII), financial details, or full request/response bodies containing sensitive data.
  ```java
  // DON'T
  log.debug("Authenticating user {} with password {}", username, password); // Leaks password
  log.info("Processing payment for user {} with details: {}", userId, paymentDetailsObject); // Leaks payment info
  ```
* Do log events related to sensitive operations but exclude the sensitive data itself. Use identifiers where possible. Sanitize or mask data if partial logging is essential (use carefully).
  ```java
  // DO
  log.info("User {} authentication attempt.", username); // Log attempt, not password
  log.debug("User {} successfully authenticated.", username);
  log.info("Processing payment initiated for user {} with transactionId {}", userId, transactionId); // Log event with ID
  // Consider dedicated Audit logging for sensitive operations if required
  ```
* Why? Security and privacy are paramount. Logged sensitive data can be exposed through compromised log files or monitoring systems, leading to severe security breaches and compliance violations (e.g., GDPR).
* Want to learn more? [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

## 7. Configure Logging via External Configuration
* Don't configure logging programmatically (e.g., setting levels, adding appenders in code) unless there's a very specific, dynamic need.
  ```java
  // DON'T (Generally)
  // Logger root = (ch.qos.logback.classic.Logger)LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME);
  // root.setLevel(ch.qos.logback.classic.Level.DEBUG); // Hardcoded level in code
  ```
* Do use Spring Boot's standard configuration files (`application.properties` or `application.yml`) for common logging settings (levels, file output). Use `logback-spring.xml` (or `log4j2-spring.xml`) for more advanced configurations (custom appenders, filters, profiles).
  ```yaml
  # DO (in application.yml)
  logging:
    level:
      root: INFO
      com.yourcompany.service: DEBUG
      org.springframework.web: WARN
    file:
      name: logs/my-app.log
    pattern:
      console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
      file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  ```
* Why? Externalized configuration allows changing logging behavior without recompiling code. It's standard practice in Spring Boot and easier to manage across environments.
* Want to learn more? [Spring Boot Logging Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging.configuration)

## 8. Leverage MDC for Contextual Logging
* Don't manually prepend contextual information (like request ID, user ID) to every log message.
  ```java
  // DON'T
  String requestId = getCurrentRequestId();
  log.debug("[" + requestId + "] Fetching data for user: " + userId);
  log.info("[" + requestId + "] Data fetched successfully.");
  ```
* Do use MDC (Mapped Diagnostic Context) to store contextual information (like `traceId`, `spanId`, `userId`). Configure your log pattern to include MDC values. Frameworks like Spring Cloud Sleuth/Micrometer Tracing often populate MDC automatically.
  ```java
  // DO (Manual MDC example - often automated)
  try (MDC.MDCCloseable closable = MDC.putCloseable("requestId", req.getRequestId())) {
      log.debug("Fetching data for user: {}", userId); // requestId automatically included via pattern
      // ... processing ...
      log.info("Data fetched successfully.");
  } // MDC automatically cleared

  // Example logback-spring.xml pattern including MDC key "requestId"
  // <pattern>%d{...} [%thread] %-5level %logger{36} [ReqId: %X{requestId}] - %msg%n</pattern>
  ```
* Why? Provides consistent contextual information across all log messages related to a specific request or process without cluttering the code. Essential for tracing requests in distributed systems.
* Want to learn more? [Logback Manual: MDC](https://logback.qos.ch/manual/mdc.html), [Spring Cloud Sleuth Documentation (or Micrometer Tracing)](https://spring.io/projects/spring-cloud-sleuth)

## 9. Be Mindful of Logging Performance
* Don't log excessively in tight loops or performance-critical sections, especially at `DEBUG` or `TRACE` levels if they might be enabled in production.
  ```java
  // DON'T (Potentially slow if DEBUG is enabled)
  for (int i = 0; i < largeList.size(); i++) {
      log.debug("Processing item {} with value {}", i, largeList.get(i));
      // ... process item ...
  }
  ```
* Do log entry/exit points of critical sections or loops at appropriate levels (`INFO` or `DEBUG`). Log summaries instead of individual items in loops if necessary. Use conditional logging (`if (log.isDebugEnabled())`) only if constructing log arguments is *itself* expensive (rare with parameterized logging).
  ```java
  // DO
  log.debug("Starting processing of {} items.", largeList.size());
  for (int i = 0; i < largeList.size(); i++) {
       // ... process item ...
       // Log only on error or significant event inside loop, or log summary after
  }
  log.debug("Finished processing {} items.", largeList.size());
  ```
* Why? Excessive logging, especially synchronous logging to disk or network, can significantly impact application performance. Parameterized logging avoids most performance issues, but high volume still matters.
* Want to learn more? [Logback Manual: Performance](https://logback.qos.ch/manual/configuration.html#shutdownHook), General Application Performance Tuning articles.

## 10. Write Clear and Actionable Log Messages
* Don't write vague or uninformative log messages.
  ```java
  // DON'T
  log.error("Error occurred"); // What error? Where?
  log.info("Done."); // Done with what?
  log.debug("Value: " + value); // Value of what?
  ```
* Do write messages that clearly state what happened, include relevant identifiers (IDs, names), and provide context. For errors, suggest potential causes or next steps if known.
  ```java
  // DO
  log.error("Failed to connect to database server '{}' after {} retries.", dbHost, retryCount, exception);
  log.info("Order {} processed successfully for customer {}.", orderId, customerId);
  log.debug("Calculated discount {} for product {} based on ruleset {}.", discount, productId, rulesetId);
  log.warn("Configuration value '{}' not found. Using default value '{}'.", configKey, defaultValue);
  ```
* Why? Clear logs are essential for effective monitoring, debugging, and operational support. They allow developers and operators to quickly understand what the application is doing and diagnose problems.
