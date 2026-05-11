# Simple Guide to Logging, SLF4J, and Log4j2

## What is Logging?

Logging means saving messages from your application while it runs.

These messages help you:
- understand what the application is doing
- debug errors
- track important events
- save history in console or files

Example:
- application started
- user logged in
- database connection failed

---

## What is SLF4J?

**SLF4J** stands for **Simple Logging Facade for Java**.

It is **not a real logging system**.

It is just an **API/interface** that your code uses for logging.

Why use SLF4J?
- your code stays independent from the logging framework
- you can switch logging framework later easily
- it is a common standard in Java projects

Example:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Demo {
    private static final Logger logger = LoggerFactory.getLogger(Demo.class);

    public static void main(String[] args) {
        logger.info("Application started");
        logger.error("Something went wrong");
    }
}
```

---

## What is Log4j2?

**Log4j2** is a **real logging framework**.

It does the actual work:
- prints logs to console
- saves logs to file
- formats log messages
- controls log levels
- supports rolling files

So:

- **SLF4J** = logging API
- **Log4j2** = logging implementation

Usually:
- your code uses **SLF4J**
- **Log4j2** works behind it

---

## Common Log Levels

These are the most used log levels:

- **TRACE** → very detailed information
- **DEBUG** → useful for developers
- **INFO** → normal important messages
- **WARN** → something unexpected but application still works
- **ERROR** → serious problem

Example:
```java
logger.debug("Debug message");
logger.info("User created successfully");
logger.warn("Low disk space");
logger.error("Database connection failed");
```

---

## How to Use SLF4J with Log4j2

Your Java code will use the SLF4J logger.

Example:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class App {
    private static final Logger logger = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        logger.info("App started");
        logger.warn("This is a warning");
        logger.error("This is an error");
    }
}
```

---

## Maven Configuration (`pom.xml`)

Add these dependencies in your `pom.xml`:

```xml
<dependencies>
    <!-- SLF4J API -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>2.0.13</version>
    </dependency>

    <!-- Log4j2 Core -->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.23.1</version>
    </dependency>

    <!-- Log4j2 API -->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.23.1</version>
    </dependency>

    <!-- SLF4J to Log4j2 binding -->
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-slf4j2-impl</artifactId>
        <version>2.23.1</version>
    </dependency>
</dependencies>
```

---

## Gradle Configuration

Add this in `build.gradle`:

```gradle
dependencies {
    implementation 'org.slf4j:slf4j-api:2.0.13'
    implementation 'org.apache.logging.log4j:log4j-api:2.23.1'
    implementation 'org.apache.logging.log4j:log4j-core:2.23.1'
    implementation 'org.apache.logging.log4j:log4j-slf4j2-impl:2.23.1'
}
```

If you use **Gradle Kotlin DSL** (`build.gradle.kts`):

```kotlin
dependencies {
    implementation("org.slf4j:slf4j-api:2.0.13")
    implementation("org.apache.logging.log4j:log4j-api:2.23.1")
    implementation("org.apache.logging.log4j:log4j-core:2.23.1")
    implementation("org.apache.logging.log4j:log4j-slf4j2-impl:2.23.1")
}
```

---

## How to Configure Log4j2

Create a file named:

```text
src/main/resources/log4j2.xml
```

Example configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>

        <File name="FileLogger" fileName="logs/app.log">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n"/>
        </File>
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="FileLogger"/>
        </Root>
    </Loggers>
</Configuration>
```

---

## How to Save Logs to a File

In the above configuration:

```xml
<File name="FileLogger" fileName="logs/app.log">
```

This means:
- a folder named `logs` will be used
- logs will be saved in `app.log`

So your logs go to:

```text
logs/app.log
```

---

## Better Option: Rolling File Logs

A rolling file means:
- logs are saved to file
- when file becomes too large or date changes, a new file is created

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>

        <RollingFile name="RollingFile"
                     fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd}-%i.log.gz">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
            <DefaultRolloverStrategy max="7"/>
        </RollingFile>
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```

This will:
- save current logs in `logs/app.log`
- create backup log files daily or when size reaches `10 MB`
- keep up to 7 rolled files

---

## Simple Project Structure

```text
your-project/
 └── src/
     └── main/
         └── resources/
             └── log4j2.xml
```

---

## Simple Example Program

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LoggingExample {
    private static final Logger logger = LoggerFactory.getLogger(LoggingExample.class);

    public static void main(String[] args) {
        logger.info("Application started");
        logger.debug("Debugging application");
        logger.warn("Warning message");
        logger.error("Error message");
    }
}
```

---

## Summary

- **Logging** helps track application activity
- **SLF4J** is a logging API
- **Log4j2** is the logging framework implementation
- Use **SLF4J in code**
- Use **Log4j2 for configuration and file saving**
- Add dependencies in **Maven** or **Gradle**
- Create `log4j2.xml` in `src/main/resources`
- Use `File` or `RollingFile` appender to save logs

---

## Recommended Simple Setup

- Code uses `SLF4J`
- Backend uses `Log4j2`
- Config file: `src/main/resources/log4j2.xml`
- Save logs in `logs/app.log`
