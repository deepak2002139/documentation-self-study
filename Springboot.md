# Spring Boot Interview Notes

## 1. What is Dependency Injection (DI)?

Dependency Injection means:
- An object does not create its dependent object by itself.
- Spring creates the required object and gives it to the class.

### Simple definition
**DI is a way to provide required dependencies to a class from outside instead of creating them inside the class.**

### Example without DI
```java
class Car {
    Engine engine = new Engine();
}
```

Problem:
- `Car` is tightly coupled with `Engine`.
- Hard to test.
- Hard to change implementation.

### Example with DI
```java
class Car {
    private Engine engine;

    Car(Engine engine) {
        this.engine = engine;
    }
}
```

Here, `Engine` is given from outside.

### In Spring Boot
Spring container creates beans and injects them where needed.

Example:
```java
@Component
class Engine {
}

@Component
class Car {
    private final Engine engine;

    public Car(Engine engine) {
        this.engine = engine;
    }
}
```

Spring creates:
- `Engine` object
- `Car` object
- injects `Engine` into `Car`

---

## 2. Types of Dependency Injection in Spring

### 1. Constructor Injection
Best and most recommended way.

```java
@Service
class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### Why constructor injection is best
- Dependency is mandatory
- Easy to test
- Object becomes immutable
- Recommended by Spring

---

### 2. Setter Injection
Dependency is set using setter method.

```java
@Service
class UserService {
    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### Use case
- When dependency is optional
- When you may want to change dependency later

---

### 3. Field Injection
Dependency is injected directly into field.

```java
@Service
class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

### Drawbacks
- Hard to test
- Not recommended
- Hidden dependency

---

## 3. Interview Answer for Dependency Injection

**Short Answer:**
Dependency Injection is an IOC principle in which Spring gives the required object to a class instead of the class creating it itself. It reduces tight coupling, improves testability, and makes code easier to maintain.

**Best practice:**
Use **constructor injection**.

---

## 4. How many ways to create object in Spring Boot?

This question can be answered in **two meanings**:

## Meaning 1: Normal Java ways to create object
In Java, objects can be created in multiple ways:

### 1. Using `new` keyword
```java
Student s = new Student();
```

### 2. Using reflection
```java
Student s = Student.class.getDeclaredConstructor().newInstance();
```

### 3. Using `clone()`
```java
Student s2 = (Student) s1.clone();
```

### 4. Using deserialization
Object is recreated from byte stream.

### 5. Using factory method
```java
Student s = StudentFactory.createStudent();
```

---

## Meaning 2: Ways to create object in Spring Boot / Spring

In Spring Boot, object means **bean**.

### 1. Using `@Component`
```java
@Component
class MyService {
}
```

Also includes:
- `@Service`
- `@Repository`
- `@Controller`
- `@RestController`

These are stereotype annotations.

---

### 2. Using `@Bean` inside `@Configuration`
```java
@Configuration
class AppConfig {
    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

Used when:
- Class is from third-party library
- You want manual control over object creation

---

### 3. Using XML configuration
Old way.

```xml
<bean id="myService" class="com.example.MyService"/>
```

Not common in Spring Boot projects.

---

### 4. Using Java configuration + conditional creation
Example:
- `@Bean`
- `@Profile`
- `@Conditional`

Spring creates bean only when condition matches.

---

### 5. Using `ApplicationContext.getBean()`
Important point:
This does **not create bean definition manually**, but gets the object managed by Spring.

```java
ApplicationContext context = ...;
MyService service = context.getBean(MyService.class);
```

Interview tip:
Say this carefully:
**Spring creates the bean, and we fetch it using `getBean()`**.

---

## 5. Best Interview Answer for “Ways to create object in Spring Boot”

**Answer:**
In Spring Boot, objects are usually created as Spring beans. Main ways are:

1. Using `@Component` and its specialized annotations like `@Service`, `@Repository`, `@Controller`
2. Using `@Bean` inside a `@Configuration` class
3. Using XML bean configuration (older approach)
4. Using conditional/configuration-based bean creation such as `@Profile` or `@Conditional`

If interviewer asks general Java object creation ways, then mention:
- `new`
- reflection
- clone
- deserialization
- factory method

---

## 6. Difference between `@Component` and `@Bean`

### `@Component`
- Used on class
- Auto-detected by component scanning
- Good for application classes

### `@Bean`
- Used on method
- Written inside `@Configuration`
- Good for third-party or custom object creation

---

## 7. Important Keywords to Speak in Interview

For **Dependency Injection**:
- loose coupling
- IOC
- Spring container
- testability
- constructor injection
- maintainability

For **Object creation in Spring Boot**:
- bean
- component scanning
- `@Component`
- `@Bean`
- `@Configuration`
- application context

---

## 8. 30-Second Revision

### Dependency Injection
- Spring gives dependent object to class
- Class does not create dependency itself
- Reduces tight coupling
- Improves testability
- Best way: constructor injection

### Object creation in Spring Boot
- Spring creates objects as beans
- Main ways:
  - `@Component`
  - `@Bean`
  - XML
  - conditional bean creation
- In Java generally:
  - `new`
  - reflection
  - clone
  - deserialization
  - factory method

---

## 9. One-Line Interview Answers

### Q1. What is dependency injection?
Dependency Injection is a design pattern in which Spring provides required dependencies to a class instead of the class creating them itself.

### Q2. Which injection type is best?
Constructor injection is best.

### Q3. How many ways to create object in Spring Boot?
Mainly by using `@Component` and `@Bean`; older way is XML, and beans can also be created conditionally.

### Q4. Why is DI useful?
Because it reduces tight coupling and makes code easier to test and maintain.
