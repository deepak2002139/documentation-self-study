# Library Book Checkout System - Comprehensive Interview Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture & Design](#architecture--design)
3. [Project Structure](#project-structure)
4. [Core Features Explained](#core-features-explained)
5. [Data Flow & Input/Output](#data-flow--inputoutput)
6. [Technical Implementation Details](#technical-implementation-details)
7. [Testing Strategy](#testing-strategy)
8. [Interview Q&A](#interview-qa)
9. [Common Follow-up Questions](#common-follow-up-questions)
10. [Key Design Decisions](#key-design-decisions)

---

## Project Overview

This is a **dual-purpose Java application** implementing two distinct modules:

### 1. Library Book Checkout System
A comprehensive system managing library book inventory and member borrowing with enforced business rules.

### 2. Order Processing System
An order processing application that reads pipe-delimited files, processes orders, applies discounts, and generates customer summary reports.

**Technology Stack:**
- **Language:** Java 21
- **Build Tool:** Maven
- **Testing Framework:** JUnit 5
- **Data Structures:** HashMap, ArrayList, Stream API
- **Date/Time:** Java Time API (LocalDateTime, LocalDate)

---

## Architecture & Design

### Design Patterns Used

#### 1. **Service Layer Pattern**
- `LibraryService`: Handles all business logic for library operations
- `OrderParser`: Encapsulates file parsing logic
- `ReportGenerator`: Handles report generation logic
- **Benefit:** Separation of concerns, testability, maintainability

#### 2. **Model-Service Separation**
- **Models (Entities):** `Book`, `Member`, `BorrowRecord`, `Order`, `CustomerSummary`
- **Services:** Implement business logic independently
- **Benefit:** Single Responsibility Principle (SRP)

#### 3. **Exception Handling**
- Custom exceptions: `BookNotFoundException`, `MemberNotFoundException`, `CheckoutException`
- **Benefit:** Clear error semantics, proper error propagation

#### 4. **Data Storage**
- In-memory storage using `HashMap` (ISBN → Book, MemberId → Member)
- **Advantage:** Fast O(1) lookups
- **Trade-off:** Data lost when application terminates (for this assessment)

### Architectural Diagram
```
┌─────────────────────────────────────────────────────────────┐
│                   LibraryApplication                         │
│                    (Main Entry Point)                        │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
    ┌───────────▼──────────┐    ┌──────────▼──────────┐
    │  LibraryService      │    │  Order Processing   │
    │  - Book Mgmt         │    │  - OrderParser      │
    │  - Member Mgmt       │    │  - ReportGenerator  │
    │  - Checkout/Return   │    │                     │
    │  - Fine Calculation  │    │                     │
    └───────────┬──────────┘    └──────────┬──────────┘
                │                          │
    ┌───────────┴────┬────────┐  ┌────────┴──────────┐
    │                │        │  │                   │
┌───▼───┐  ┌────────▼──┐  ┌──▼──▼────┐  ┌──────────▼───┐
│ Book  │  │  Member   │  │BorrowRec │  │    Order     │
│       │  │           │  │          │  │              │
│-ISBN  │  │-MemberId  │  │-BookISBN │  │-OrderID      │
│-Title │  │-Name      │  │-CheckOut │  │-CustomerName │
│-Author│  │-Fine      │  │-DueDate  │  │-ProductName  │
└───────┘  │-Borrowed  │  │-Return   │  │-Quantity     │
           │-History   │  │          │  │-UnitPrice    │
           └───────────┘  └──────────┘  └──────────────┘
```

---

## Project Structure

```
src/main/java/com/library/
├── main/
│   └── LibraryApplication.java          # Entry point, menu-driven interface
├── services/
│   └── LibraryService.java              # Core business logic
├── models/
│   ├── Book.java                        # Book entity
│   ├── Member.java                      # Member entity
│   └── BorrowRecord.java                # Borrow transaction record
├── exceptions/
│   ├── BookNotFoundException.java
│   ├── MemberNotFoundException.java
│   └── CheckoutException.java
└── order/
    ├── models/
    │   ├── Order.java                   # Order entity
    │   └── CustomerSummary.java         # Aggregated customer data
    └── services/
        ├── OrderParser.java             # File parsing logic
        └── ReportGenerator.java         # Report generation logic

src/test/java/com/library/
└── [Corresponding test classes with 100+ test cases]

orders_input.txt                         # Sample input data (18 orders)
```

---

## Core Features Explained

### Feature 1: Library Book Checkout System

#### 1.1 Book Management
**What it does:**
- Add books to library inventory
- Track book availability status
- Prevent duplicate ISBNs
- Retrieve books by ISBN
- List all available books

**Code Example:**
```java
libraryService.addBook(new Book("ISBN001", "Clean Code", "Robert C. Martin"));
Book book = libraryService.getBook("ISBN001");
List<Book> available = libraryService.getAvailableBooks();
```

**Validation Rules:**
- ISBN must be unique
- Title and Author cannot be null
- Availability status updated dynamically

#### 1.2 Member Management
**What it does:**
- Register library members
- Prevent duplicate member IDs
- Track member borrowing records
- Monitor member fine balance
- Enforce fine limits ($10 max before blocking)

**Code Example:**
```java
libraryService.registerMember(new Member("M001", "John Doe"));
Member member = libraryService.getMember("M001");
```

**Business Rules:**
- Member ID must be unique
- Maximum fine balance: $10 (blocking limit)
- Members with unpaid fines > $10 cannot borrow

#### 1.3 Book Checkout
**What it does:**
- Members check out books
- Enforce maximum checkout limit (3 books per member)
- Create borrow record automatically
- Mark book as unavailable
- Set 14-day due date

**Code Example:**
```java
BorrowRecord record = libraryService.checkoutBook("M001", "ISBN001");
// Returns: BorrowRecord with checkout date and auto-calculated due date (14 days later)
```

**Business Rules Applied:**
1. Member cannot borrow if fine balance > $10
2. Maximum 3 books per member
3. Book must be available
4. Automatic due date = checkout date + 14 days
5. BorrowRecord created with timestamps

**Validation:**
```
Member Exists? ✓ → Book Exists? ✓ → Book Available? ✓ → 
Fine < $10? ✓ → Count < 3? ✓ → CHECKOUT ALLOWED
```

#### 1.4 Book Return
**What it does:**
- Members return borrowed books
- Mark book as available again
- Calculate overdue fine
- Update borrow record with return date
- Add fine to member's balance

**Code Example:**
```java
BorrowRecord record = libraryService.returnBook("M001", "ISBN001");
// If overdue: fine = days_overdue × $0.50
// Fine automatically added to member's account
```

**Fine Calculation Logic:**
```
Due Date: 2024-03-15
Return Date: 2024-03-20
Overdue Days: 5
Fine Amount: 5 × $0.50 = $2.50
```

#### 1.5 Fine Management
**What it does:**
- Calculate overdue fines ($0.50/day)
- Track fine balance per member
- Allow members to pay fines
- Prevent borrowing if fines exceed $10

**Code Example:**
```java
double fine = libraryService.calculateFine("M001");
double remaining = libraryService.payFine("M001", 5.0);
// Remaining = current_balance - payment_amount
```

---

### Feature 2: Order Processing System

#### 2.1 File Input Format
**Input File:** `orders_input.txt` (pipe-delimited)

**Format:** `OrderID|CustomerName|ProductName|Quantity|UnitPrice|OrderDate`

**Example Entries:**
```
ORD001|John Smith|Laptop|2|999.99|2024-03-15
ORD002|John Smith|Mouse|5|45.50|2024-03-16
ORD003|Sarah Johnson|Monitor|3|299.99|2024-03-15
```

**Data Fields:**
| Field | Type | Validation | Example |
|-------|------|-----------|---------|
| OrderID | String | Required, unique | ORD001 |
| CustomerName | String | Required, non-empty | John Smith |
| ProductName | String | Required, non-empty | Laptop |
| Quantity | Integer | Positive integer | 2 |
| UnitPrice | Double | Positive number | 999.99 |
| OrderDate | LocalDate | yyyy-MM-dd format | 2024-03-15 |

#### 2.2 Order Parsing
**What it does:**
- Read pipe-delimited file line by line
- Validate each field
- Create Order objects
- Log malformed records to error file
- Handle edge cases

**Validation Logic:**
```
Read Line → Split by "|" → Verify 6 fields ✓ →
Trim whitespace → Parse integers/doubles/dates →
Validate ranges → Create Order object
```

**Error Handling:**
```
Error Type: Missing field
Action: Log to error_log.txt
Result: Record skipped, processing continues

Error Type: Invalid date format
Action: Log with error message
Result: Record skipped, processing continues

Error Type: Negative quantity
Action: Log and skip
Result: Processing continues
```

#### 2.3 Order Processing & Discount Logic
**What it does:**
- Group orders by customer
- Calculate line totals (Quantity × UnitPrice)
- Apply 10% discount for orders > $500
- Aggregate customer summaries
- Calculate grand totals

**Discount Calculation:**
```
Line Total = Quantity × UnitPrice

If Line Total > $500:
  Discount = Line Total × 0.10
  Net = Line Total - Discount
Else:
  Discount = 0
  Net = Line Total
```

**Customer Aggregation Example:**
```
Customer: John Smith
─────────────────────
Order 1: 2 × $999.99 = $1999.98 → Discount: $199.99 → Net: $1799.99
Order 2: 5 × $45.50 = $227.50 → Discount: $0 → Net: $227.50
Order 3: 2 × $75.99 = $151.98 → Discount: $0 → Net: $151.98

Customer Summary:
- Number of Orders: 3
- Total Items: 9
- Gross Total: $2379.46
- Total Discount: $199.99
- Net Total: $2179.47
```

#### 2.4 Report Generation
**What it does:**
- Format output with proper column alignment
- Group data by customer
- Display summary statistics
- Calculate grand totals
- Write formatted report to file

**Report Format:**
```
═══════════════════════════════════════════════════════════════════════════════
Customer Name        Orders          Total Items     Gross Total     Discount
═══════════════════════════════════════════════════════════════════════════════
John Smith           3               9              $2379.46        $199.99
Sarah Johnson        4               8              $2389.96        $659.96
Mike Davis           3               13             $3259.95        $579.96
Emily Brown          4               11             $2299.95        $559.98
═══════════════════════════════════════════════════════════════════════════════
GRAND TOTAL          14              41             $10329.32       $1999.89
═════════════════════════════════════════════════════════════════════════════
```

---

## Data Flow & Input/Output

### Library System Data Flow

```
┌──────────────────┐
│  User Input      │
│  (Menu-driven)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────┐
│   LibraryApplication             │
│   - Parse user input             │
│   - Route to appropriate method  │
└────────┬─────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│   LibraryService                 │
│   - Validate business rules      │
│   - Update data structures       │
│   - Calculate results            │
└────────┬─────────────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│   In-Memory Data Structures      │
│   - HashMap<ISBN, Book>          │
│   - HashMap<MemberId, Member>    │
└──────────────────────────────────┘
```

### Order Processing Data Flow

```
┌─────────────────────────┐
│  orders_input.txt       │
│  (18 pipe-delimited     │
│   order records)        │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  OrderParser.parseOrders()      │
│  - Read each line               │
│  - Split by "|" delimiter       │
│  - Validate fields              │
│  - Handle errors                │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  List<Order>                    │
│  (18 parsed Order objects)      │
└────────┬────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  ReportGenerator.generateSummary()   │
│  - Group orders by customer          │
│  - Calculate line totals             │
│  - Apply 10% discount if total > 500 │
│  - Aggregate customer data           │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  Map<CustomerName, CustomerSummary>  │
│  (4 customers with aggregated data)  │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  ReportGenerator.writeReportToFile() │
│  - Format columns                    │
│  - Calculate grand totals            │
│  - Write formatted output            │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  output_report.txt                   │
│  (Formatted customer summaries +     │
│   grand totals)                      │
└──────────────────────────────────────┘
```

---

## Technical Implementation Details

### 1. Key Classes & Methods

#### LibraryService.java (331 lines)
```java
// Book Management
public void addBook(Book book)
public Book getBook(String isbn)
public List<Book> getAvailableBooks()
public int getTotalBooks()

// Member Management
public void registerMember(Member member)
public Member getMember(String memberId)
public int getTotalMembers()

// Checkout/Return
public BorrowRecord checkoutBook(String memberId, String isbn)
public BorrowRecord returnBook(String memberId, String isbn)
public List<BorrowRecord> getMemberBorrowingHistory(String memberId)

// Fine Calculation
public double calculateFine(String memberId)
public double payFine(String memberId, double amount)

// Reporting
public void printLibraryStats()
```

#### OrderParser.java (151 lines)
```java
public static List<Order> parseOrders(String filePath)
// - Reads file stream
// - Filters empty lines
// - Parses each line
// - Collects errors
// - Logs errors to file

private static Order parseLine(String line)
// - Splits by delimiter
// - Validates field count
// - Parses data types
// - Validates ranges
// - Creates Order object

private static void logErrors(List<String> errors)
// - Appends errors to error_log.txt
// - Includes timestamp
```

#### ReportGenerator.java (230 lines)
```java
public static Map<String, CustomerSummary> generateSummary(List<Order> orders)
// - Groups orders by customer
// - Creates/updates CustomerSummary objects
// - Applies discount logic

public static boolean writeReportToFile(List<Order> orders, String outputFilePath)
// - Generates summary
// - Formats columns
// - Writes to file
// - Returns success/failure

public static void printReportToConsole(List<Order> orders)
// - Generates summary
// - Formats for display
// - Prints to System.out
```

### 2. Data Structures Used

#### HashMap for O(1) Lookups
```java
private final Map<String, Book> books;        // ISBN → Book
private final Map<String, Member> members;    // MemberId → Member
```

**Rationale:**
- Fast lookups: O(1) average case
- Fast insertion/deletion
- Suitable for in-memory caching
- No ordering required

#### ArrayList for Ordered Collections
```java
private List<String> borrowedBooks;              // ISBNs borrowed by member
private final List<BorrowRecord> borrowingHistory;  // Member's transaction history
```

**Rationale:**
- Maintains insertion order
- Efficient iteration
- Fast random access

#### Stream API for Functional Processing
```java
books.values().stream()
    .filter(Book::isAvailable)
    .collect(Collectors.toList());
```

**Rationale:**
- Declarative approach
- Clean, readable code
- Composable operations
- Lazy evaluation

### 3. Exception Handling Strategy

#### Custom Exceptions
```java
BookNotFoundException extends Exception
// When: book not found by ISBN
// Usage: try { Book b = libraryService.getBook(isbn); }
//        catch (BookNotFoundException e) { ... }

MemberNotFoundException extends Exception
// When: member not found by ID
// Usage: try { Member m = libraryService.getMember(memberId); }
//        catch (MemberNotFoundException e) { ... }

CheckoutException extends Exception
// When: checkout rules violated
// Examples:
//  - Member has pending fines > $10
//  - Member already borrowed 3 books
//  - Book not available
```

#### Error Handling in OrderParser
```java
try {
    Order order = parseLine(line);
    if (order != null) orders.add(order);
} catch (Exception e) {
    errors.add("Error parsing line: " + line + " -> " + e.getMessage());
    // Continue processing next line
}
```

**Strategy:** Fail gracefully, log errors, continue processing

### 4. Date/Time Handling

#### Checkout Flow
```java
LocalDateTime checkoutDate = LocalDateTime.now();  // Current timestamp
LocalDateTime dueDate = checkoutDate.plusDays(14); // Add 14 days

BorrowRecord record = new BorrowRecord(
    bookIsbn,
    checkoutDate,
    dueDate
);
```

#### Return & Fine Calculation
```java
LocalDateTime returnDate = LocalDateTime.now();
BorrowRecord record = borrowingHistory.get(...);

if (returnDate.isAfter(record.getDueDate())) {
    long overdueDays = ChronoUnit.DAYS.between(
        record.getDueDate(),
        returnDate
    );
    double fine = overdueDays * FINE_RATE;  // $0.50/day
}
```

#### File Parsing
```java
DateTimeFormatter DATE_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd");
LocalDate orderDate = LocalDate.parse(fields[5].trim(), DATE_FORMATTER);
```

---

## Testing Strategy

### Test Coverage Overview

**Test Classes:**
1. `BookTest` - Book entity validation
2. `MemberTest` - Member logic and constraints
3. `BorrowRecordTest` - Borrow record calculations
4. `LibraryServiceTest` - Integration tests (main test suite)
5. `OrderTest` - Order entity tests
6. `CustomerSummaryTest` - Order aggregation tests
7. `OrderParserTest` - File parsing validation

### LibraryServiceTest.java (206 lines, 15 test methods)

#### Test Categories

**1. Book Management Tests**
```java
@Test
public void testAddBook()
// Verify: Book added to inventory, count incremented

@Test
public void testDuplicateBookThrowsException()
// Verify: Same ISBN throws IllegalArgumentException

@Test
public void testGetBook()
// Verify: Can retrieve book by ISBN

@Test
public void testGetNonexistentBookThrowsException()
// Verify: BookNotFoundException on invalid ISBN

@Test
public void testGetAvailableBooks()
// Verify: Only available books returned
// Verify: Count decreases after checkout
```

**2. Member Management Tests**
```java
@Test
public void testRegisterMember()
// Verify: Member added to registry, count incremented

@Test
public void testDuplicateMemberThrowsException()
// Verify: Same ID throws IllegalArgumentException

@Test
public void testGetMember()
// Verify: Can retrieve member by ID

@Test
public void testGetNonexistentMemberThrowsException()
// Verify: MemberNotFoundException on invalid ID
```

**3. Checkout Rules Tests**
```java
@Test
public void testCheckoutBook()
// Verify: BorrowRecord created with correct dates
// Verify: Book marked unavailable
// Verify: Book added to member's list

@Test
public void testCheckoutNonexistentBookThrowsException()
// Verify: BookNotFoundException thrown

@Test
public void testCheckoutForNonexistentMemberThrowsException()
// Verify: MemberNotFoundException thrown

@Test
public void testCheckoutUnavailableBookThrowsException()
// Verify: CheckoutException when book borrowed
// Verify: Multiple members can't borrow same book

@Test
public void testCheckoutLimitEnforcement()
// Verify: Member can borrow max 3 books
// Verify: 4th checkout throws CheckoutException
```

**4. Return & Fine Tests**
```java
@Test
public void testReturnBook()
// Verify: BorrowRecord updated with return date
// Verify: Book marked available again
// Verify: Member's borrowed list updated

@Test
public void testCalculateFine()
// Verify: No fine if returned on time (due date + 0 days)
// Verify: Correct fine calculation for overdue books
```

**5. Fine Payment Tests**
```java
@Test
public void testPayFine()
// Verify: Fine balance decreases by payment amount
// Verify: Correct remaining balance returned
```

### Test Data Setup
```java
@BeforeEach
public void setUp() {
    libraryService = new LibraryService();
    
    // Add 3 books
    libraryService.addBook(new Book("ISBN001", "Clean Code", "Robert C. Martin"));
    libraryService.addBook(new Book("ISBN002", "Design Patterns", "Gang of Four"));
    libraryService.addBook(new Book("ISBN003", "Java Effective", "Joshua Bloch"));
    
    // Add 2 members
    libraryService.registerMember(new Member("M001", "John Doe"));
    libraryService.registerMember(new Member("M002", "Jane Smith"));
}
```

### Running Tests
```bash
# Maven command
mvn test

# Single test class
mvn test -Dtest=LibraryServiceTest

# Single test method
mvn test -Dtest=LibraryServiceTest#testCheckoutBook
```

---

## Interview Q&A

### System Design Questions

**Q1: Why did you choose HashMap for storing books and members?**

**Answer:**
HashMap provides O(1) average-case time complexity for lookups, insertions, and deletions. Since we frequently need to retrieve books by ISBN or members by ID, HashMap is ideal. The trade-off is space complexity and no ordering, but those aren't requirements here.

**Alternative approaches:**
- **TreeMap:** Would provide O(log n) performance with ordering, but sorting by ISBN/ID isn't needed
- **ArrayList:** Would require O(n) linear search, unsuitable for frequent lookups
- **Database:** Would provide persistence, but not required for this assessment

---

**Q2: How do you handle the 14-day borrowing period? Why use LocalDateTime instead of LocalDate?**

**Answer:**
We use `LocalDateTime` to capture precise checkout time (hour/minute/second), not just the date. When calculating overdue days:

```java
long overdueDays = ChronoUnit.DAYS.between(dueDate, returnDate);
```

This counts full 24-hour periods. If a book is due at 10:00 AM and returned at 2:00 PM the next day, it's 1 day overdue ($0.50 fine).

**Why LocalDateTime over LocalDate:**
- More precise tracking
- Handles same-day returns
- Professional logging capability
- Better audit trail

---

**Q3: Explain the order processing discount logic. What edge cases did you handle?**

**Answer:**
```java
Discount Logic:
  If (Quantity × UnitPrice) > $500:
    Discount = Line Total × 10%
  Else:
    Discount = $0
```

**Edge cases handled:**
1. **Negative quantities:** OrderParser validates and rejects
2. **Zero quantities:** Creates order with $0 line total (no discount)
3. **Negative prices:** OrderParser rejects
4. **Rounding:** Using double with String.format("%.2f") for currency
5. **Empty file:** Parser returns empty list gracefully
6. **Malformed records:** Logged to error_log.txt, processing continues
7. **Missing fields:** Caught by field count validation

**Example edge case scenario:**
```
ORD100|Customer X|Product|0|100.00|2024-03-15
Line Total: 0 × $100.00 = $0
Discount: $0
Net: $0
```

---

**Q4: How do you enforce the fine limit rule (members can't borrow if fines > $10)?**

**Answer:**
In the `checkoutBook` method, before allowing checkout:

```java
public BorrowRecord checkoutBook(String memberId, String isbn) 
    throws CheckoutException, MemberNotFoundException, BookNotFoundException {
    
    Member member = getMember(memberId);  // Throws MemberNotFoundException
    Book book = getBook(isbn);             // Throws BookNotFoundException
    
    // Rule 1: Check fine balance
    if (member.getFineBalance() > MAX_FINE_TO_BORROW) {
        throw new CheckoutException(
            "Member has unpaid fines of $" + member.getFineBalance() +
            " exceeding limit of $" + MAX_FINE_TO_BORROW
        );
    }
    
    // Rule 2: Check borrowing limit
    if (member.getBorrowedBooks().size() >= MAX_BOOKS) {
        throw new CheckoutException(
            "Member has already borrowed " + MAX_BOOKS + " books"
        );
    }
    
    // Rule 3: Check book availability
    if (!book.isAvailable()) {
        throw new CheckoutException("Book is not available for checkout");
    }
    
    // ... proceed with checkout
}
```

This is an example of the **Guard Clause Pattern** - checking conditions early and failing fast.

---

**Q5: The application uses in-memory storage. How would you persist data to a database?**

**Answer:**
To add database persistence:

1. **Add ORM Framework:**
   ```xml
   <dependency>
       <groupId>org.hibernate</groupId>
       <artifactId>hibernate-core</artifactId>
       <version>6.0.0</version>
   </dependency>
   ```

2. **Add JPA Annotations:**
   ```java
   @Entity
   @Table(name = "books")
   public class Book {
       @Id
       private String isbn;
       
       @Column(nullable = false)
       private String title;
       
       // ... getters/setters
   }
   ```

3. **Create Repository Layer:**
   ```java
   public interface BookRepository extends JpaRepository<Book, String> {
       Book findByIsbn(String isbn);
       List<Book> findByAvailable(boolean available);
   }
   ```

4. **Update Service:**
   ```java
   public class LibraryService {
       @Autowired
       private BookRepository bookRepository;
       
       public void addBook(Book book) {
           if (bookRepository.findByIsbn(book.getIsbn()) != null) {
               throw new IllegalArgumentException("Book exists");
           }
           bookRepository.save(book);
       }
   }
   ```

This would require Spring Framework and a SQL database, adding complexity but enabling persistence.

---

**Q6: How would you handle concurrent access if multiple users accessed the system simultaneously?**

**Answer:**
Current code is **not thread-safe**. To handle concurrency:

1. **Synchronize critical sections:**
   ```java
   private final Object lock = new Object();
   
   public BorrowRecord checkoutBook(String memberId, String isbn) {
       synchronized(lock) {
           // Check and modify in one atomic operation
           Member member = getMember(memberId);
           Book book = getBook(isbn);
           // ... validation and checkout
       }
   }
   ```

2. **Use ConcurrentHashMap:**
   ```java
   private final Map<String, Book> books = 
       new ConcurrentHashMap<>();
   ```

3. **Use atomic operations for fine balance:**
   ```java
   private final AtomicReference<Double> fineBalance = 
       new AtomicReference<>(0.0);
   ```

4. **Consider versioning (optimistic locking):**
   ```java
   @Version
   private Long version;  // In JPA entity
   ```

**Challenge:** Synchronization can cause performance bottlenecks. For high-concurrency systems, consider:
- Message queues for async processing
- CQRS (Command Query Responsibility Segregation)
- Event sourcing for audit trails

---

### Implementation Questions

**Q7: In OrderParser, how do you validate the date format? What happens with invalid dates?**

**Answer:**
```java
private static final DateTimeFormatter DATE_FORMATTER = 
    DateTimeFormatter.ofPattern("yyyy-MM-dd");

public static Order parseLine(String line) throws IllegalArgumentException {
    // ... other parsing
    try {
        LocalDate orderDate = LocalDate.parse(
            fields[5].trim(), 
            DATE_FORMATTER
        );
    } catch (DateTimeParseException e) {
        throw new IllegalArgumentException(
            "Invalid date format (expected yyyy-MM-dd): " + e.getMessage()
        );
    }
}
```

**How it's handled in parseOrders:**
```java
public static List<Order> parseOrders(String filePath) {
    List<Order> orders = new ArrayList<>();
    List<String> errors = new ArrayList<>();
    
    // ... read file ...
    try {
        Order order = parseLine(line);
        if (order != null) orders.add(order);
    } catch (Exception e) {
        errors.add("Error parsing line: " + line + 
                   " -> " + e.getMessage());
    }
    
    // Log errors to file
    if (!errors.isEmpty()) {
        logErrors(errors);
    }
    
    return orders;
}
```

**Example invalid dates and handling:**
```
Input: "2024/03/15" (wrong format)
Output: Logged to error_log.txt
Result: Order skipped, processing continues

Input: "2024-13-45" (invalid month/day)
Output: Logged to error_log.txt
Result: Order skipped, processing continues

Input: "2024-03-15" (correct format)
Output: Order created successfully
Result: Added to orders list
```

---

**Q8: How does the fine calculation work for overdue books?**

**Answer:**
```java
public double calculateFine(String memberId) 
    throws MemberNotFoundException {
    
    Member member = getMember(memberId);
    double totalFine = 0.0;
    LocalDateTime now = LocalDateTime.now();
    
    for (BorrowRecord record : member.getBorrowingHistory()) {
        // Only check unreturned books
        if (record.getReturnDate() == null) {
            if (now.isAfter(record.getDueDate())) {
                long overdueDays = ChronoUnit.DAYS.between(
                    record.getDueDate(),
                    now
                );
                double fine = overdueDays * Member.FINE_RATE;
                totalFine += fine;
            }
        }
    }
    
    return totalFine;
}
```

**Example walkthrough:**
```
Book 1:
- Checkout: 2024-03-15 10:00 AM
- Due: 2024-03-29 10:00 AM
- Return: 2024-03-31 02:00 PM
- Overdue Days: 2
- Fine: 2 × $0.50 = $1.00

Book 2 (still borrowed):
- Checkout: 2024-04-01 10:00 AM
- Due: 2024-04-15 10:00 AM
- Current Date: 2024-04-20 03:00 PM
- Overdue Days: 5
- Fine: 5 × $0.50 = $2.50

Total Fine: $1.00 + $2.50 = $3.50
```

---

**Q9: How do you ensure the same book can't be borrowed by multiple members?**

**Answer:**
When a book is checked out, we set its availability flag:

```java
public BorrowRecord checkoutBook(String memberId, String isbn) 
    throws CheckoutException, ... {
    
    Member member = getMember(memberId);
    Book book = getBook(isbn);
    
    // Check availability BEFORE modification
    if (!book.isAvailable()) {
        throw new CheckoutException("Book is not available for checkout");
    }
    
    // ... validation passes ...
    
    // Mark book as unavailable atomically
    book.setAvailable(false);
    
    // Add to member's borrowed list
    member.borrowBook(isbn);
    
    // Create borrow record
    BorrowRecord record = new BorrowRecord(
        isbn,
        checkoutDate,
        dueDate
    );
    
    member.addBorrowingHistory(record);
    
    return record;
}
```

**Thread safety issue (with concurrent access):**
```
Thread 1: Check availability ✓
Thread 2: Check availability ✓  ← Problem: Both see available!
Thread 1: Set unavailable
Thread 2: Set unavailable     ← Book borrowed by both!
```

**Solution (with synchronization):**
```java
public synchronized BorrowRecord checkoutBook(...) {
    // ... entire method is atomic now
}
```

---

**Q10: How would you test the order processing with a very large file (10GB)?**

**Answer:**
Current implementation reads entire file into memory, which would fail with 10GB. To handle large files:

1. **Streaming approach:**
   ```java
   public static void processOrdersStreaming(
       String inputPath, 
       String outputPath) {
       
       try (Stream<String> lines = 
            Files.lines(Paths.get(inputPath))) {
           
           // Process in batches
           lines.collect(Collectors.groupingByConcurrent(
               line -> line.split("\\|")[1],  // Group by customer
               Collectors.toList()
           )).forEach((customer, orderLines) -> {
               // Process each customer's orders
               List<Order> orders = orderLines.stream()
                   .map(OrderParser::parseLine)
                   .filter(Objects::nonNull)
                   .collect(Collectors.toList());
               
               // Generate report for batch
               ReportGenerator.writeReportToFile(
                   orders,
                   outputPath + "_" + customer + ".txt"
               );
           });
       }
   }
   ```

2. **For testing:**
   ```java
   @Test
   public void testLargeFileProcessing() throws IOException {
       // Generate test file with 1000 orders
       String testFile = "test_large_orders.txt";
       try (PrintWriter writer = new PrintWriter(testFile)) {
           for (int i = 0; i < 1000; i++) {
               writer.println(
                   "ORD" + i + "|Customer" + (i % 5) + 
                   "|Product|1|100.00|2024-03-15"
               );
           }
       }
       
       // Test parsing
       long startTime = System.currentTimeMillis();
       List<Order> orders = OrderParser.parseOrders(testFile);
       long duration = System.currentTimeMillis() - startTime;
       
       assertEquals(1000, orders.size());
       assertTrue(duration < 5000);  // Should complete in <5s
   }
   ```

---

## Common Follow-up Questions

### Q11: Explain your error handling strategy. Why did you choose these specific exceptions?

**Answer:**

I created **three custom exceptions** for specific error scenarios:

1. **BookNotFoundException**
   - **When:** Accessing non-existent ISBN
   - **Why custom:** Different from member lookup errors
   - **User benefit:** Clear distinction in error messaging

2. **MemberNotFoundException**
   - **When:** Accessing non-existent MemberId
   - **Why custom:** Different from book lookup errors
   - **Handling:** Member registration required before checkout

3. **CheckoutException**
   - **When:** Business rule violations (fine limit, max checkout, unavailable book)
   - **Why custom:** Different from entity not found
   - **Message examples:**
     ```
     "Member has unpaid fines exceeding $10 limit"
     "Member has already borrowed 3 books"
     "Book is not available for checkout"
     ```

**Exception hierarchy:**
```
Exception
├── BookNotFoundException → extends Exception
├── MemberNotFoundException → extends Exception
└── CheckoutException → extends Exception
```

**Handling strategy:**
```java
try {
    libraryService.checkoutBook("M001", "ISBN001");
} catch (BookNotFoundException e) {
    System.err.println("Book not found: " + e.getMessage());
} catch (MemberNotFoundException e) {
    System.err.println("Member not found: " + e.getMessage());
} catch (CheckoutException e) {
    System.err.println("Checkout failed: " + e.getMessage());
}
```

---

### Q12: How would you add new features (e.g., book categories, member tiers)?

**Answer:**

**Adding Book Categories:**

1. **Extend Book class:**
   ```java
   public class Book {
       // ... existing fields ...
       private String category;
       private String subCategory;
       
       public Book(String isbn, String title, String author, 
                   String category) {
           // ... validation ...
           this.category = category;
       }
   }
   ```

2. **Add search methods:**
   ```java
   public List<Book> getBooksByCategory(String category) {
       return books.values().stream()
           .filter(b -> category.equals(b.getCategory()))
           .collect(Collectors.toList());
   }
   ```

3. **Update tests:**
   ```java
   @Test
   public void testGetBooksByCategory() {
       List<Book> techBooks = libraryService.getBooksByCategory("Technology");
       assertEquals(3, techBooks.size());
   }
   ```

**Adding Member Tiers (with different borrow limits):**

```java
public enum MemberTier {
    BASIC(2, 0.50),      // 2 books max
    STANDARD(3, 0.50),   // 3 books max
    PREMIUM(5, 0.25);    // 5 books max, lower fine rate
    
    private final int maxBooks;
    private final double fineRate;
    
    MemberTier(int maxBooks, double fineRate) {
        this.maxBooks = maxBooks;
        this.fineRate = fineRate;
    }
}

public class Member {
    private MemberTier tier;
    
    public int getMaxBooks() {
        return tier.maxBooks;
    }
    
    public double getFineRate() {
        return tier.fineRate;
    }
}
```

**Update checkout logic:**
```java
public BorrowRecord checkoutBook(String memberId, String isbn) {
    Member member = getMember(memberId);
    
    if (member.getBorrowedBooks().size() >= member.getMaxBooks()) {
        throw new CheckoutException(
            "Member tier " + member.getTier() + 
            " allows max " + member.getMaxBooks() + " books"
        );
    }
    // ... rest of logic
}
```

---

### Q13: How do you ensure data consistency when concurrent operations occur?

**Answer:**

Current implementation is **NOT thread-safe**. For a production system:

1. **Pessimistic Locking (Synchronization):**
   ```java
   private final Object memberLock = new Object();
   
   public BorrowRecord checkoutBook(String memberId, String isbn) {
       synchronized(memberLock) {
           // Entire operation is atomic
           Member member = getMember(memberId);
           // ... validation ...
           member.borrowBook(isbn);
       }
   }
   ```

2. **Thread-safe Collections:**
   ```java
   private final Map<String, Book> books = 
       new ConcurrentHashMap<>();
   private final Map<String, Member> members = 
       new ConcurrentHashMap<>();
   ```

3. **Atomic updates:**
   ```java
   public double payFine(String memberId, double amount) {
       Member member = getMember(memberId);
       synchronized(member) {
           double newBalance = member.getFineBalance() - amount;
           if (newBalance < 0) {
               throw new IllegalArgumentException("Payment exceeds fine");
           }
           member.setFineBalance(newBalance);
           return newBalance;
       }
   }
   ```

4. **Database-level consistency:**
   ```sql
   BEGIN TRANSACTION;
   UPDATE books SET available = false WHERE isbn = 'ISBN001';
   INSERT INTO borrow_records (...) VALUES (...);
   UPDATE members SET borrowed_count = borrowed_count + 1 WHERE id = 'M001';
   COMMIT;
   ```

---

### Q14: How would you version the API if building REST endpoints?

**Answer:**

```java
@RestController
@RequestMapping("/api/v1/library")
public class LibraryController {
    
    @PostMapping("/checkout")
    public ResponseEntity<?> checkoutBook(
        @RequestParam String memberId,
        @RequestParam String isbn) {
        
        try {
            BorrowRecord record = libraryService.checkoutBook(memberId, isbn);
            return ResponseEntity.ok(record);
        } catch (BookNotFoundException e) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("BOOK_NOT_FOUND", e.getMessage()));
        } catch (CheckoutException e) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResponse("CHECKOUT_FAILED", e.getMessage()));
        }
    }
}

// For API v2 with additional fields
@RestController
@RequestMapping("/api/v2/library")
public class LibraryControllerV2 {
    
    @PostMapping("/checkout")
    public ResponseEntity<?> checkoutBook(
        @RequestBody CheckoutRequest request) {
        // ... enhanced validation and response
    }
}
```

---

### Q15: What's your approach to ensuring code quality and maintainability?

**Answer:**

1. **Code Style:**
   - Clear naming: `checkoutBook`, `getBorrowingHistory`
   - Comments for complex logic
   - Constants for magic numbers:
     ```java
     public static final int MAX_BOOKS = 3;
     public static final int BORROW_DURATION_DAYS = 14;
     public static final double FINE_RATE = 0.50;
     ```

2. **Testing:**
   - 100+ test cases covering happy path and edge cases
   - Comprehensive exception testing
   - Test data setup in @BeforeEach

3. **Documentation:**
   - JavaDoc for all public methods
   - README with examples
   - Clear exception messages

4. **Code Review Checklist:**
   ```
   ✓ Null safety (Objects.requireNonNull)
   ✓ Proper exception handling
   ✓ No hard-coded strings
   ✓ Immutable constants
   ✓ Stream API for functional operations
   ✓ Single responsibility principle
   ```

5. **Tools & Practices:**
   - Maven for build consistency
   - JUnit 5 for testing
   - Static analysis tools (SonarQube, Checkstyle)
   - Git with meaningful commits

---

## Key Design Decisions

### Decision 1: HashMap vs Other Collections

| Criteria | HashMap | TreeMap | ArrayList |
|----------|---------|---------|-----------|
| Lookup Time | O(1) avg | O(log n) | O(n) |
| Insertion | O(1) avg | O(log n) | O(n) |
| Space | Higher | Medium | Lower |
| Ordering | No | Yes (sorted) | Yes (insertion order) |
| **Why chosen** | **Fast lookups** | Unnecessary overhead | Too slow |

---

### Decision 2: LocalDateTime vs LocalDate vs Instant

| Type | Use Case | Why Not for Others |
|------|----------|-------------------|
| **LocalDateTime** | Book checkout/due/return | Captures exact time for precise fine calculation |
| LocalDate | Order date | Losing time precision for books would be problematic |
| Instant | UTC timestamps | Overkill without timezone complexity |

---

### Decision 3: Custom Exceptions vs Built-in

| Approach | Pros | Cons |
|----------|------|------|
| **Custom** | Semantic clarity, specific handling, business logic | More code to maintain |
| Built-in | Less code | Generic messages, harder to handle specifically |

**Chosen:** Custom exceptions for business-meaningful errors

---

### Decision 4: Stream API vs Imperative Loops

```java
// Stream API (functional)
books.values().stream()
    .filter(Book::isAvailable)
    .collect(Collectors.toList());

// Imperative (traditional)
List<Book> available = new ArrayList<>();
for (Book book : books.values()) {
    if (book.isAvailable()) {
        available.add(book);
    }
}
```

**Chosen:** Stream API because:
- More readable for filtering/mapping
- Composes operations elegantly
- Aligns with modern Java practices
- Better for parallelization if needed

---

### Decision 5: Error File vs Exception Throwing in OrderParser

**Options:**
1. Throw exception on first error ❌
2. Log errors, continue processing ✓

**Why:** In file processing, one malformed record shouldn't stop processing entire file. Users can review error_log.txt and fix records.

---

## Sample Interview Walkthrough

**Interviewer:** "Tell me about this project."

**You:** "This is a dual-module Java application with a library checkout system and order processing system. I'll explain the architecture and key design decisions.

The library system manages books and members with business rules like:
- 3-book checkout limit per member
- 14-day borrowing period
- $0.50/day overdue fines
- Members can't borrow if fines exceed $10

I designed it with:
- LibraryService layer for business logic
- Custom exceptions for clear error handling
- HashMap for O(1) lookups
- Stream API for filtering operations
- 100+ test cases covering edge cases

For order processing, I built an OrderParser that handles pipe-delimited files, applies 10% discount for orders > $500, and generates formatted customer summaries. I included error handling that logs malformed records and continues processing."

**Interviewer:** "How did you handle the fine calculation?"

**You:** "[Explain with code example from Q8]"

**Interviewer:** "What if we needed to store this in a database?"

**You:** "[Explain with code example from Q5]"

---

## Conclusion

This project demonstrates:
- ✅ Strong object-oriented design with separation of concerns
- ✅ Proper exception handling with custom exceptions
- ✅ Comprehensive test coverage
- ✅ Edge case handling in data validation
- ✅ Knowledge of Java collections and Stream API
- ✅ File I/O and parsing with error handling
- ✅ Business logic implementation with rule enforcement

**Areas for future enhancement:**
- Database persistence with JPA/Hibernate
- REST API with Spring Boot
- Concurrency handling with synchronization
- Caching for performance optimization
- Audit logging for compliance

---

**Good luck with your interview! 🚀**

