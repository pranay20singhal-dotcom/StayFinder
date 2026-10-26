# StayFinder — Comprehensive C++17 OOP Viva Examination Guide

This guide is prepared specifically for college oral examinations (viva voce), technical presentations, and project defenses for **StayFinder**.

---

## 1. Project Overview & Problem Statement

**StayFinder** is a full-stack accommodation marketplace platform connecting guests seeking boutique stays across India with property hosts.

### The Engineering Challenge
Many student web applications rely entirely on dynamic scripting languages (Node.js, Python, PHP) with little emphasis on foundational software architecture. StayFinder bridges high-performance systems programming with a consumer-facing web product:
- **Core Engine:** Written in standard **ISO C++17**.
- **Data Safety:** Atomic transactions and mutex locking to prevent double bookings.
- **Persistence:** Embedded SQLite 3 database with zero external database server overhead.
- **Frontend:** Semantic HTML5, responsive CSS3 (warm ivory & earthy charcoal aesthetic), and vanilla JavaScript communicating via JSON REST endpoints.

---

## 2. Technology Stack & Design Decisions

| Layer | Technology | Engineering Justification |
|---|---|---|
| **Backend Language** | C++17 (MSVC / MinGW g++) | Memory efficiency, deterministic execution, RAII resource lifetimes, compile-time type safety. |
| **HTTP Server** | `cpp-httplib` (single-header) | Lightweight embedded multi-threaded HTTP/1.1 server. No runtime framework dependencies. |
| **JSON Serialization** | `nlohmann/json` | Modern modern C++ syntax, type-safe conversions, clean exception propagation. |
| **Database** | Embedded SQLite 3 (`sqlite3.c`) | ACID-compliant, embedded in-process storage; single file (`stayfinder.db`), zero installation. |
| **Concurrency** | `std::mutex`, `std::lock_guard` | Thread serialization across incoming HTTP worker threads during booking validation & insert. |
| **Frontend** | HTML5, CSS3, Vanilla ES6+ | Lightweight, fast rendering, zero npm/node/webpack bloat. |

---

## 3. UML Class Hierarchy & Relationships

```
                        +---------------------------+
                        |      «Abstract» User      |
                        +---------------------------+
                        | - name_: std::string      |
                        | - email_: std::string     |
                        +---------------------------+
                        | + virtual ~User()         |
                        | + virtual roleName() = 0  |
                        | + virtual bookingMsg() = 0|
                        +-------------+-------------+
                                      ^
                                      | (Inheritance)
                    +-----------------+-----------------+
                    |                                   |
        +-----------+-----------+           +-----------+-----------+
        |         Guest         |           |         Host          |
        +-----------------------+           +-----------------------+
        | + roleName(): "Guest" |           | + roleName(): "Host"  |
        | + bookingMsg(): ovr   |           | + bookingMsg(): ovr   |
        +-----------+-----------+           +-----------+-----------+
                    ^                                   ^
                    : (Composition)                     : (Composition)
                    :                                   :
        +-----------+-----------+           +-----------+-----------+
        |        Booking        | ◇-------->|       Property        |
        +-----------------------+           +-----------------------+
        | - id_: int            |           | - id_: int            |
        | - checkIn_, checkOut_ |           | - name_, city_: string|
        | - nights_, guests_    |           | - pricePerNight_: int |
        | - status_: string     |           | - host_: Host         |
        | - property_: Property |           +-----------------------+
        | - guest_: Guest       |
        +-----------------------+

        +-----------------------------+
        | «Interface» PricingStrategy |
        +-----------------------------+
        | + calculateTotal(...) = 0   |
        | + calculateDiscount(...) = 0|
        +--------------+--------------+
                       ^
                       | (Inheritance)
        +--------------+------------------+
        |                                 |
+-------+------------------+   +----------+--------------------+
|  StandardPricingStrategy |   |  ExtendedStayPricingStrategy  |
|  (rate * nights)         |   |  (10% off for >= 5 nights)    |
+--------------------------+   +-------------------------------+
```

---

## 4. The Four Pillars of OOP in StayFinder

### 1. Encapsulation
- **What it is:** Bundling data with methods that operate on that data and restricting direct access to internal components.
- **In StayFinder:**
  - In `Property` and `Booking`, all fields (`name_`, `pricePerNight_`, `status_`, `guests_`) are strictly `private`.
  - State modification requires validation:
    ```cpp
    void Booking::cancel() {
        if (status_ == "CANCELLED") {
            throw std::logic_error("Reservation #" + std::to_string(id_) + " is already cancelled.");
        }
        status_ = "CANCELLED";
    }
    ```
  - Outside code cannot tamper with private database handles or booking statuses.

### 2. Abstraction
- **What it is:** Exposing only the essential interface while hiding complex underlying mechanics.
- **In StayFinder:**
  - `BookingService` is a pure abstract class (`booking_service.h`).
  - `main.cpp` only calls `bookingService.createBooking(...)` and knows nothing about SQLite queries, connection handles, or table structures.
  - `PricingStrategy` abstracts pricing calculation algorithms from the reservation logic.

### 3. Inheritance
- **What it is:** Deriving a class from a base class to establish an "is-a" relationship and promote code reuse.
- **In StayFinder:**
  - `Guest` and `Host` inherit from `User`. They inherit the member variables `name_` and `email_` and the constructor logic via constructor delegation (`User(std::move(name), std::move(email))`).
  - Domain exceptions (`ValidationError`, `NotFoundError`, `ConflictError`, `DatabaseError`) inherit from `std::runtime_error`.

### 4. Polymorphism
- **Compile-Time Polymorphism (Function Overloading):**
  - `BookingService::searchProperties(const std::string& city)` vs.
    `BookingService::searchProperties(const std::string& city, int minGuests, int maxPrice)`.
  - Resolved at compile time by function signature matching.
- **Runtime Polymorphism (Virtual Dispatch via vptr/vtable):**
  - `User*` pointers dispatch stakeholder messages:
    ```cpp
    const std::vector<const User*> stakeholders = { &booking.guest(), &booking.property().host() };
    for (const User* person : stakeholders) {
        messages.push_back({
            {"to", person->roleName()},
            {"text", person->bookingMessage(booking.property().name(), booking.guest().name())}
        });
    }
    ```
  - `PricingStrategy*` pointers dynamically select either `StandardPricingStrategy` or `ExtendedStayPricingStrategy` based on duration.

---

## 5. Other Key C++ OOP & Systems Concepts

### Strategy Pattern (Design Pattern)
- Decouples pricing calculation algorithms from the booking service.
- If a guest books 5 or more nights, `SqliteBookingService::selectStrategy(nights)` selects `ExtendedStayPricingStrategy` which applies a 10% discount. Stays under 5 nights use `StandardPricingStrategy`.

### Operator Overloading
- Overloaded stream insertion `operator<<` for `Property` and `Booking` enables clean, declarative logging:
  ```cpp
  friend std::ostream& operator<<(std::ostream& os, const Booking& b) {
      os << "Booking #" << b.id_ << " [" << b.property_.name()
         << " | Guest: " << b.guest_.name()
         << " | " << b.checkIn_ << " to " << b.checkOut_
         << " | Total: ₹" << b.totalPrice_ << "]";
      return os;
  }
  ```

### RAII (Resource Acquisition Is Initialization)
- The `Database` class opens SQLite in its constructor and guarantees closure in `~Database()`.
- Thread locks use `std::lock_guard<std::mutex> lock(mutex_);`, automatically unlocking when leaving scope, even if an exception is thrown.

### Virtual Destructors
- Base classes with virtual functions (`User`, `BookingService`, `PricingStrategy`) declare `virtual ~ClassName() = default;`.
- **Viva significance:** Deleting a derived object via a base pointer without a virtual destructor results in **undefined behavior** and resource leaks because the derived destructor is never invoked.

---

## 6. Database Schema & Concurrency Safety

### SQLite Tables
```sql
CREATE TABLE properties (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT    NOT NULL,
    city            TEXT    NOT NULL,
    description     TEXT    NOT NULL,
    price_per_night INTEGER NOT NULL,
    max_guests      INTEGER NOT NULL,
    image           TEXT    NOT NULL,
    host_name       TEXT    NOT NULL,
    host_email      TEXT    NOT NULL
);

CREATE TABLE bookings (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    property_id  INTEGER NOT NULL REFERENCES properties(id),
    guest_name   TEXT    NOT NULL,
    guest_email  TEXT    NOT NULL,
    check_in     TEXT    NOT NULL,
    check_out    TEXT    NOT NULL,
    nights       INTEGER NOT NULL,
    guests       INTEGER NOT NULL DEFAULT 1,
    total_price  INTEGER NOT NULL,
    status       TEXT    NOT NULL DEFAULT 'CONFIRMED',
    created_at   TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

### Overlap Detection & Atomic Transactions
- **Overlap Formula:**
  Two reservations $[A_{in}, A_{out})$ and $[B_{in}, B_{out})$ overlap if and only if:
  $$A_{in} < B_{out} \quad \text{AND} \quad A_{out} > B_{in}$$
- **Adjacent Stays:**
  If Reservation A departs on `2026-11-13` and Reservation B arrives on `2026-11-13`:
  $A_{out} = 2026-11-13$ and $B_{in} = 2026-11-13$.
  Since $2026-11-13 > 2026-11-13$ is **false**, overlap is zero. Adjacent stays are permitted!
- **Cancellation Impact:**
  `hasOverlap()` filters by `status = 'CONFIRMED'`, ensuring cancelled reservations immediately unlock calendar dates.

---

## 7. 30 Realistic Viva Questions & Model Answers

### Question 1: What is the main objective of StayFinder?
> **Answer:** StayFinder is an accommodation marketplace demonstrating a production-grade C++17 object-oriented backend communicating with an interactive HTML5/CSS3 frontend. It models domain entities, enforces server-authoritative pricing via the Strategy Pattern, and prevents booking conflicts using transactional SQLite persistence.

### Question 2: Why choose C++ instead of Node.js, Python, or Go for this project?
> **Answer:** C++ offers deterministic resource management through RAII, zero garbage-collection pauses, high throughput, and direct control over memory layout. In an academic context, it clearly demonstrates core OOP fundamentals—such as vtables, object slicing prevention, constructor delegation, and operator overloading—which are often obscured in interpreted languages.

### Question 3: How does the C++ backend serve web requests without Node.js or Apache?
> **Answer:** We embed `cpp-httplib`, a lightweight single-header C++ multi-threaded web server. It uses standard OS sockets (`Winsock ws2_32` on Windows) to bind to port 8080, listens for incoming TCP connections, parses HTTP requests, and routes them to C++ lambda handlers.

### Question 4: Explain Encapsulation using an example from your codebase.
> **Answer:** In `backend/models.h`, the `Booking` class maintains private variables like `status_` and `totalPrice_`. The status cannot be arbitrarily changed from the outside; it can only transition to `"CANCELLED"` through the member function `cancel()`, which enforces that an already-cancelled booking cannot be cancelled again.

### Question 5: What is Abstraction and where is it implemented?
> **Answer:** Abstraction provides a clean interface while hiding underlying implementation details. In `booking_service.h`, `BookingService` is an abstract interface with pure virtual methods like `createBooking()` and `searchProperties()`. `main.cpp` interacts only with `BookingService&`, completely insulated from SQLite implementation details.

### Question 6: What makes a class abstract in C++?
> **Answer:** A class is abstract if it contains at least one **pure virtual function**, declared with `= 0` (e.g., `virtual std::string roleName() const = 0;`). Abstract classes cannot be instantiated directly; only derived classes that implement all pure virtual methods can be instantiated.

### Question 7: Why does the `User` class have a virtual destructor?
> **Answer:** When an object of a derived class (like `Guest` or `Host`) is deleted through a base class pointer (`User*`), a non-virtual destructor would result in **undefined behavior** because only `User::~User()` would be invoked, leaking derived class resources. Marking it `virtual ~User() = default;` ensures the derived destructor runs first, followed by the base destructor.

### Question 8: Explain the difference between Function Overloading and Function Overriding.
> **Answer:**
> - **Function Overloading** is compile-time polymorphism where functions in the same scope share a name but have different parameter signatures (e.g., `searchProperties(city)` vs. `searchProperties(city, minGuests, maxPrice)`).
> - **Function Overriding** is runtime polymorphism where a derived class provides a specific implementation of a virtual method defined in its base class with the exact same signature (e.g., `Guest::roleName()` overriding `User::roleName()`).

### Question 9: How does Runtime Polymorphism work internally in C++?
> **Answer:** For any class containing virtual functions, the compiler creates a **Virtual Method Table (vtable)** containing function pointers to the virtual methods. Each instance of the class contains an invisible pointer called **vptr** pointing to that vtable. When a virtual function is called through a base pointer (`person->bookingMessage(...)`), the call is resolved at runtime by dereferencing the vptr to find the correct method address.

### Question 10: What is the difference between Composition and Inheritance?
> **Answer:**
> - **Inheritance** represents an **IS-A** relationship (e.g., `Guest` is a `User`).
> - **Composition** represents a **HAS-A** relationship where one object contains another as a member (e.g., `Property` has a `Host`, and `Booking` has a `Property` and a `Guest`). Composition is favored when an entity is composed of components rather than being a subtype.

### Question 11: What is the Strategy Pattern and how is it used in StayFinder?
> **Answer:** The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. We define an abstract `PricingStrategy` interface with `StandardPricingStrategy` (linear rate) and `ExtendedStayPricingStrategy` (10% discount on stays of 5+ nights). The booking service chooses the appropriate strategy at runtime, keeping pricing algorithms modular and extensible.

### Question 12: Why did you overload `operator<<` for `Property` and `Booking`?
> **Answer:** Overloading `operator<<` allows our domain objects to be formatted directly into standard streams like `std::cout` or log files:
> `std::cout << "[server] " << booking << "\n";`
> It is declared as a `friend` function so it can access private members while preserving encapsulation.

### Question 13: What is RAII and where is it applied?
> **Answer:** Resource Acquisition Is Initialization (RAII) ties the lifecycle of a resource to the lifetime of an object. In `database.cpp`, `Database::Database()` opens the SQLite connection, and `Database::~Database()` automatically closes it. In `sqlite_booking_service.cpp`, `std::lock_guard<std::mutex> lock(mutex_)` acquires the mutex upon creation and automatically releases it upon scope exit or if an exception is thrown.

### Question 14: How does StayFinder prevent double bookings in a multi-threaded environment?
> **Answer:** When multiple users submit bookings simultaneously:
> 1. A `std::mutex` serializes critical check-and-insert code sections.
> 2. SQLite transactions (`BEGIN TRANSACTION`, `COMMIT`, `ROLLBACK`) ensure that checking `hasOverlap()` and executing `insertBooking()` happen atomically. If an overlap is detected, the transaction rolls back and throws `ConflictError`.

### Question 15: How are date overlaps mathematically determined?
> **Answer:** Two intervals $[S_1, E_1)$ and $[S_2, E_2)$ overlap if and only if $S_1 < E_2$ and $E_1 > S_2$. In SQL:
> `SELECT COUNT(*) FROM bookings WHERE property_id = ?1 AND status = 'CONFIRMED' AND check_in < ?3 AND check_out > ?2;`

### Question 16: Can a guest check in on the day another guest checks out?
> **Answer:** Yes, adjacent stays are permitted. If Booking A is Oct 10–13 and Booking B is Oct 13–16:
> $A_{out} = \text{Oct 13}$ and $B_{in} = \text{Oct 13}$. The condition $\text{Oct 13} > \text{Oct 13}$ is false, so overlap count is 0.

### Question 17: How is SQL Injection prevented?
> **Answer:** By using **parameterized queries** with SQLite prepared statements (`sqlite3_prepare_v2` and `sqlite3_bind_text` / `sqlite3_bind_int`). User input is never concatenated directly into SQL query strings.

### Question 18: How does the exception handling hierarchy work?
> **Answer:** We define domain exceptions derived from `std::runtime_error`:
> - `ValidationError` → HTTP 400 Bad Request
> - `NotFoundError` → HTTP 404 Not Found
> - `ConflictError` → HTTP 409 Conflict
> - `DatabaseError` → HTTP 500 Internal Server Error
> In `main.cpp`, a centralized `runRoute()` lambda catches these exceptions and maps them to clean JSON error responses.

### Question 19: Why should the backend calculate the final price instead of trusting the frontend?
> **Answer:** Security. A malicious user can tamper with JavaScript variables or HTTP request payloads to submit a total price of ₹1. The backend must be the authoritative source of truth, recalculating the price using the `PricingStrategy` and property rate stored in the database.

### Question 20: How are cancelled bookings handled in the database?
> **Answer:** Rather than deleting the row (which destroys booking history), the status column is updated to `'CANCELLED'`. Since `hasOverlap()` only counts rows where `status = 'CONFIRMED'`, cancelled dates immediately become available for other guests.

### Question 21: What STL containers are used and why?
> **Answer:**
> - `std::vector`: For dynamic arrays of properties and bookings with contiguous memory and fast iteration.
> - `std::string`: For dynamic text handling and safe string manipulation.
> - `std::unique_ptr`: For exclusive ownership of `PricingStrategy` instances without manual `delete`.
> - `std::optional`: For database lookups (`findPropertyById`, `findBookingById`) that might not find a matching row, avoiding raw null pointers.

### Question 22: What happens if the database schema needs to change on an existing installation?
> **Answer:** In `Database::migrateSchemaIfNecessary()`, we query `PRAGMA table_info(bookings)` to inspect existing columns. If `status` or `guests` are missing, we run `ALTER TABLE bookings ADD COLUMN ... DEFAULT ...`. This is non-destructive and preserves existing user records.

### Question 23: What is the purpose of the `noexcept` specifier on getters?
> **Answer:** It informs the compiler that the function will never throw exceptions, allowing optimizations like avoiding exception unwind frames and enabling more efficient move operations in STL containers.

### Question 24: What is the role of constructor initializer lists?
> **Answer:** In `Property` and `Booking`, constructor initializer lists initialize member variables directly rather than default-constructing them first and then assigning values in the constructor body. This avoids redundant object construction and allows initialization of `const` or reference members.

### Question 25: How does client-side and server-side validation cooperate?
> **Answer:** Client-side validation in `script.js` provides immediate feedback (e.g., date pickers enforcing min dates, checking email format). Server-side validation in `sqlite_booking_service.cpp` provides authoritative validation. Even if client-side validation is bypassed, the backend re-validates all rules.

### Question 26: What is the role of `std::move` in your constructors?
> **Answer:** In `User`, `Property`, and `Booking`, parameters like `std::string name` are taken by value and then moved into member variables using `name_(std::move(name))`. This enables move semantics when rvalues are passed, transferring ownership of the string buffer without deep copying.

### Question 27: What is the difference between `std::unique_ptr` and `std::shared_ptr`?
> **Answer:**
> - `std::unique_ptr` enforces single, exclusive ownership with zero overhead over a raw pointer.
> - `std::shared_ptr` implements reference-counted shared ownership, carrying extra memory overhead for the control block.
> In StayFinder, `SqliteBookingService` exclusively owns its `PricingStrategy` instances, so `std::unique_ptr` is the proper choice.

### Question 28: What is object slicing and how do you prevent it?
> **Answer:** Object slicing occurs when a derived class object is assigned to a base class object by value, stripping away the derived class members and vtable. In StayFinder, we store pointers or references to base classes (`const User*`, `BookingService&`, `PricingStrategy&`), completely avoiding object slicing.

### Question 29: How did you test your application?
> **Answer:** We implemented an automated verification suite in `test_suite.py` that executes 18 integration tests against the live C++ backend, verifying health status, property lookups, standard pricing, extended stay discounts, valid bookings, overlap rejections, adjacent stay allowances, past date rejections, guest capacity constraints, and lifecycle cancellations.

### Question 30: What are the current limitations and future scope of StayFinder?
> **Answer:**
> - **Current limitations:** Simulated offline payment; plain HTTP rather than HTTPS (suitable for a local college demo); no user authentication sessions (JWT/cookies).
> - **Future scope:** Adding bcrypt-hashed passwords for host/guest login, Stripe/Razorpay payment gateway integration, and customer reviews/ratings.

---

## 8. Build & Run Instructions for Windows

### Option A: Using MSYS2 MinGW GCC (Current Setup)
```powershell
cd StayFinder\backend
cmake -S . -B build -G "MinGW Makefiles"
cmake --build build
build\stayfinder.exe
```

### Option B: Using Visual Studio 2022
```powershell
cd StayFinder\backend
cmake -S . -B build -G "Visual Studio 17 2022"
cmake --build build --config Release
build\Release\stayfinder.exe
```

### Accessing the Web Application
- **Customer Marketplace:** [http://localhost:8080](http://localhost:8080)
- **OOP Architecture Explorer:** [http://localhost:8080/oop.html](http://localhost:8080/oop.html)
- **API Health Endpoint:** [http://localhost:8080/api/health](http://localhost:8080/api/health)
