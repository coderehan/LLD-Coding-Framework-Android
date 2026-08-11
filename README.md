# 🚀 LLD Coding Interview Framework — DRSFSM

> A simple framework to approach **Low-Level Design (LLD) coding rounds**.

The biggest mistake in LLD interviews is starting with classes immediately.

Instead, start with a **real-world story** and convert that story into code.

The goal of this framework is not to memorize solutions.

The goal is to have a **repeatable way of thinking** whenever an interviewer gives you a new LLD problem.

---

# 📚 Table of Contents

* [The Flow](#-the-flow)
* [Memory Shortcut](#-memory-shortcut)
* [How the Files Communicate](#-how-the-files-communicate)
* [1. Domain Models](#1--domain-models-d)
* [2. Repository](#2--repository-r)
* [3. Strategy Pattern](#3--strategy-pattern-s)
* [4. Factory Pattern](#4--factory-pattern-f)
* [Strategy vs Factory](#-strategy-vs-factory)
* [5. Service Layer](#5--service-layer-s)
* [6. Main Function](#6--main-function-m)
* [Complete LLD Interview Flow](#-complete-lld-interview-flow)
* [Examples](#-examples-where-this-framework-applies)
* [Important Interview Reminder](#️-important-interview-reminder)
* [Final Memory Trick](#-final-memory-trick)

---

# 🌍 The Flow

Whenever you get an LLD problem, don't immediately start creating classes.

Start with the real-world story.

```text
Real World Problem
        |
        ↓
Identify Domain Objects
        |
        ↓
Create Repository
        |
        ↓
Identify Changing Behaviour
        |
        ↓
Create Strategy / Factory
        |
        ↓
Create Service Layer
        |
        ↓
Demonstrate using main()
```

---

# 🧠 Memory Shortcut

| **Letter** | **Meaning**   | **Simple Question**                       |
| ---------- | ------------- | ----------------------------------------- |
| **D**      | Domain Models | **What things exist?**                    |
| **R**      | Repository    | **Where does data come from?**            |
| **S**      | Strategy      | **What are the different ways to do it?** |
| **F**      | Factory       | **Which one should I create?**            |
| **S**      | Service       | **What action does the user perform?**    |
| **M**      | Main          | **How do I prove it works?**              |

### DRSFSM

```text
D → Domain Models
R → Repository
S → Strategy
F → Factory
S → Service
M → Main
```

---

# 🔄 How the Files Communicate

Understanding **who communicates with whom** is one of the most important parts of LLD.

For a typical LLD project:

```text
                         ┌─────────────┐
                         │   Main.kt   │
                         └──────┬──────┘
                                │
                                │ calls
                                ▼
                    ┌──────────────────────┐
                    │       Service        │
                    │ FlightBookingService │
                    └───────┬───────┬──────┘
                            │       │
                  uses      │       │ uses
                            ▼       ▼
                 ┌──────────────┐  ┌──────────────────┐
                 │  Repository  │  │    Strategy      │
                 │  Interface   │  │    Interface     │
                 └──────┬───────┘  └────────┬─────────┘
                        │                   │
                        │ implemented by    │ implemented by
                        ▼                   ▼
              ┌──────────────────┐   ┌────────────────┐
              │ RepositoryImpl   │   │  UpiPayment    │
              └────────┬─────────┘   │ CardPayment    │
                       │             └────────────────┘
                       │
                       │ works with
                       ▼
                 ┌─────────────┐
                 │   Domain    │
                 │   Models    │
                 │             │
                 │ Flight      │
                 │ Passenger   │
                 │ Booking     │
                 └─────────────┘

              ┌─────────────────────┐
              │       Factory       │
              │   PaymentFactory    │
              └──────────┬──────────┘
                         │
                         │ creates
                         ▼
                  PaymentStrategy
```

---

## 🧩 Simple Explanation

### 1. `Main.kt` → `Service`

```text
Main
  ↓
Service
```

`Main.kt` is the entry point.

It creates the required objects, injects dependencies, and calls the user's action.

```kotlin
val service = FlightBookingService(
    repository,
    paymentStrategy
)

service.bookFlight("F101")
```

**Simple meaning:**

> Main says: **"I want to book a flight."**

---

### 2. `Service` → `Repository`

```text
Service
   ↓
Repository
```

The Service needs data, so it asks the Repository.

```kotlin
val flight = repository.getFlight(flightId)
```

**Simple meaning:**

> Service says: **"Give me the flight details."**

The Service does not care whether the data comes from an API, database, cache, or fake data.

---

### 3. `Service` → `Strategy`

```text
Service
   ↓
Strategy
```

The Service needs some behaviour to be performed.

```kotlin
paymentStrategy.pay(flight.price)
```

**Simple meaning:**

> Service says: **"Perform the payment using the selected payment method."**

The Service doesn't need to know whether it is UPI, Card, Wallet, etc.

---

### 4. `RepositoryImpl` → `Repository`

```text
Repository Interface
        ▲
        │ implements
        │
RepositoryImpl
```

The Repository defines **what operations are available**.

The Repository Implementation defines **how those operations are performed**.

```kotlin
interface FlightRepository {
    fun getFlight(id: String): Flight?
}
```

```kotlin
class FlightRepositoryImpl : FlightRepository {

    override fun getFlight(id: String): Flight? {
        // Fake/static data
    }
}
```

**Simple meaning:**

> Interface says **"what can be done."**
> Implementation says **"how it is done."**

---

# Strategy & Factory — Simple Explanation

These two are easy to confuse in LLD interviews.

The easiest way to understand them is:

> **Strategy = Different ways**
> **Factory = Creates the selected way**

---

# 3. `Strategy` → Different Ways of Doing Something

```text
                    Payment
                       │
              "How should I pay?"
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
         UPI          Card        Wallet
```

The **Service doesn't want to know the payment details**.

It only says:

```kotlin
paymentStrategy.pay(flight.price)
```

### Simple Meaning

> **Strategy means: "There are multiple ways to do the same thing."**

For example:

```text
Pay → UPI
Pay → Card
Pay → Wallet
```

All of them do the same job — **payment** — but each does it differently.

---

## Strategy Interface

```kotlin
interface PaymentStrategy {
    fun pay(amount: Double)
}
```

---

## Strategy Implementations

```kotlin
class UpiPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paying using UPI")
    }
}
```

```kotlin
class CardPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paying using Card")
    }
}
```

---

### 🧠 Remember

> **Strategy = "What are the different ways to do this?"**

---

# 4. `Factory` → Choose and Create the Right One

Now imagine the user selects:

```text
Payment Method = UPI
```

Who creates the `UpiPayment` object?

That's where **Factory** comes in.

```text
User selects UPI
       ↓
   PaymentFactory
       ↓
Creates UpiPayment
```

---

## Payment Type

```kotlin
enum class PaymentType {
    UPI,
    CARD,
    WALLET
}
```

---

## Payment Factory

```kotlin
class PaymentFactory {

    fun create(type: PaymentType): PaymentStrategy {

        return when (type) {

            PaymentType.UPI ->
                UpiPayment()

            PaymentType.CARD ->
                CardPayment()

            PaymentType.WALLET ->
                WalletPayment()
        }
    }
}
```

Now:

```kotlin
val strategy =
    paymentFactory.create(PaymentType.UPI)
```

The Factory gives us:

```text
UpiPayment
```

Then Service uses it:

```kotlin
paymentStrategy.pay(amount)
```

---

# 🔥 Strategy + Factory Together

This is the part to **memorize for the interview**:

```text
                 USER
                  │
                  │ selects UPI
                  ↓
             ┌──────────┐
             │ Factory  │
             └────┬─────┘
                  │
                  │ creates
                  ↓
          UpiPaymentStrategy
                  │
                  │
                  ↓
              Service
                  │
                  │ pay()
                  ↓
              Payment
```

---

# 🧠 Easiest Memory Trick

> **Strategy = Different ways**

> **Factory = Creates the selected way**

Or simply:

```text
S → "HOW can I do it?"

F → "WHICH one should I create?"
```

---

# 🌍 Real-Life Example

Imagine booking a flight.

The user needs to make a payment.

There are different ways:

```text
                 PAYMENT
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
       UPI         CARD       WALLET
```

### Strategy

Each payment method is a different **way of performing the payment**.

```text
UPI   → Pay using UPI
CARD  → Pay using Card
WALLET → Pay using Wallet
```

### Factory

The Factory looks at the user's selection and **creates the correct payment object**.

```text
User selects CARD
       ↓
PaymentFactory
       ↓
CardPayment
```

### Service

The Service doesn't care about the implementation.

It simply says:

```kotlin
paymentStrategy.pay(amount)
```

So the complete flow is:

```text
User
 ↓
Select Payment Method
 ↓
Factory
 ↓
Creates Strategy
 ↓
Service
 ↓
paymentStrategy.pay()
 ↓
Payment completed
```

---

# 🎯 Final One-Line Understanding

```text
Strategy → Different ways of performing the same task.

Factory  → Creates the appropriate Strategy object.
```
---

### 7. Service → Domain Models

```text
Service
   ↓
Domain Models
```

The Service uses domain objects to perform business operations.

```kotlin
Booking(
    id = "B101",
    passenger = passenger,
    flight = flight,
    status = BookingStatus.CONFIRMED
)
```

**Simple meaning:**

> Domain Models represent the **real-world things** in the system.

---

# 🧠 Simplest Communication Flow

If the complete diagram feels too much, remember only this:

```text
                 MAIN
                  │
                  │ calls
                  ▼
               SERVICE
              /       \
             /         \
            ▼           ▼
      REPOSITORY      STRATEGY
          │               │
          ▼               ▼
     REPOSITORY       UPI / CARD
       IMPL
          │
          ▼
       DOMAIN
       MODELS

       FACTORY
          │
          │ creates
          ▼
       STRATEGY
```

### In one sentence:

> **Main calls the Service → Service uses Repository for data and Strategy for changing behaviour → Repository works with Domain Models → Factory creates the required Strategy object.**

---

# 1. 🧩 Domain Models (D)

> ## ❓ Question to ask
>
> **"What things exist in this real-world system?"**

Look for **nouns**.

Nouns usually become:

* 📦 Data classes
* 🚦 Enums
* 🔐 Sealed classes *(when different states contain different data)*

---

## ✈️ Example: Flight Booking System

### Real-world statement

> **"A passenger searches for a flight, selects a seat, makes payment, and receives a booking."**

### Important nouns

```text
Passenger
Flight
Seat
Payment
Booking
```

These become models.

### Example

```kotlin
data class Passenger(
    val id: String,
    val name: String
)

data class Flight(
    val id: String,
    val source: String,
    val destination: String,
    val price: Double
)

data class Booking(
    val id: String,
    val passenger: Passenger,
    val flight: Flight,
    val status: BookingStatus
)

enum class BookingStatus {
    CREATED,
    CONFIRMED,
    CANCELLED
}
```

---

# 2. 🗂 Repository (R)

> ## ❓ Question to ask
>
> **"Where does my data come from?"**

## 🌎 In real applications

```text
API
Database
Cache
File Storage
```

## 💻 In an interview

Use an **interface + fake implementation**.

The goal is to show **Separation of Concerns**.

---

## Example

### Repository Interface

```kotlin
interface FlightRepository {

    fun getFlight(id: String): Flight?
}
```

### Repository Implementation

```kotlin
class FlightRepositoryImpl : FlightRepository {

    override fun getFlight(id: String): Flight? {

        return Flight(
            id = "F101",
            source = "Chennai",
            destination = "Dubai",
            price = 5000.0
        )
    }
}
```

---

## ✅ Repository Responsibilities

* Fetch data
* Save data
* Update data
* Delete data

> 💡 **Remember**
>
> Repository should **NOT** contain business rules.

---

# 3. 🎯 Strategy Pattern (S)

> ## ❓ Question to ask
>
> **"What behaviour can change in the future?"**

Whenever you see:

* Multiple ways of doing something
* Future extensions
* Different algorithms/rules

👉 Use **Strategy Pattern**.

---

## 💳 Example: Payment

### Today

```text
UPI Payment
```

### Tomorrow

```text
Card Payment
Wallet Payment
Apple Pay
```

Instead of putting all the logic inside one class, create a strategy.

---

## Strategy Interface

```kotlin
interface PaymentStrategy {

    fun pay(amount: Double)
}
```

---

## Strategy Implementations

### UPI Payment

```kotlin
class UpiPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paid using UPI")
    }
}
```

### Card Payment

```kotlin
class CardPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paid using Card")
    }
}
```

---

## ✅ Benefit

Now adding a new payment method **does not affect existing code**.

For example, tomorrow you can simply add:

* WalletPayment
* ApplePayPayment
* GooglePayPayment

without modifying existing implementations.

---

# 4. 🏭 Factory Pattern (F)

> ## ❓ Question to ask
>
> **"Who is responsible for creating objects?"**

Factory is useful when **object creation logic becomes complex**.

---

## Example

Suppose the user selects:

```text
UPI
CARD
WALLET
```

Instead of writing lots of `if-else` or `when` statements throughout the codebase, let the Factory create the correct object.

---

## Payment Type

```kotlin
enum class PaymentType {
    UPI,
    CARD
}
```

---

## Factory Implementation

```kotlin
class PaymentFactory {

    fun create(type: PaymentType): PaymentStrategy {

        return when(type) {

            PaymentType.UPI ->
                UpiPayment()

            PaymentType.CARD ->
                CardPayment()
        }
    }
}
```

---

# ⚖️ Strategy vs Factory

This is one of the most common interview discussions.

| Strategy 🧠                       | Factory 🏭                          |
| --------------------------------- | ----------------------------------- |
| **How should something be done?** | **Which object should I create?**   |
| Focuses on behaviour              | Focuses on object creation          |
| Multiple algorithms               | Multiple object types               |
| Selected during execution         | Creates the required implementation |

---

## 🎯 Strategy

### Answers

> **"How should something be done?"**

### Examples

```text
Payment calculation
Pricing calculation
Sorting
Notification
```

---

## 🏭 Factory

### Answers

> **"Which object should I create?"**

### Examples

```text
Create UPI payment object
Create Card payment object
Create Parser object
```

---

## 🧠 Easy Way to Remember

Imagine you're ordering food.

### 🍕 Strategy

The food is already prepared.

Now you choose **how to pay**.

```text
Cash
Card
UPI
Wallet
```

👉 Behaviour changes.

---

### 🏭 Factory

You haven't received anything yet.

Someone has to prepare the correct food.

```text
Veg Pizza
Chicken Pizza
Paneer Pizza
```

The kitchen decides **what object to create**.

👉 Object creation changes.

---

# 5. 🧠 Service Layer (S)

> ## ❓ Question to ask
>
> **"What is the actual user action?"**

The **Service** contains the **business logic**.

This is where interviewers evaluate your design skills.

---

## ✈️ Example

### User wants:

```text
Book Flight
```

### Service handles:

```text
1. Validate flight
2. Check availability
3. Process payment
4. Create booking
5. Update status
```

---

## Service Implementation

```kotlin
class FlightBookingService(
    private val repository: FlightRepository,
    private val paymentStrategy: PaymentStrategy
) {

    fun bookFlight(flightId: String): Booking {

        val flight =
            repository.getFlight(flightId)
                ?: throw Exception("Flight not found")

        paymentStrategy.pay(flight.price)

        return Booking(
            id = "B101",
            passenger = Passenger(
                "P1",
                "User"
            ),
            flight = flight,
            status = BookingStatus.CONFIRMED
        )
    }
}
```

> 💡 **Remember**
>
> The **Service Layer** is the **brain of your application**.
>
> Repositories fetch data.
>
> Strategies define behaviour.
>
> Services coordinate everything and execute the business rules.

---

# 6. ▶️ Main Function (M)

> ## ❓ Question to ask
>
> **"How can I prove my design works?"**

The `main()` function acts as a **small client application**.

It should:

* Create objects
* Inject dependencies
* Call service methods
* Print output

---

## Example

```kotlin
fun main() {

    val repository =
        FlightRepositoryImpl()

    val payment =
        UpiPayment()

    val bookingService =
        FlightBookingService(
            repository,
            payment
        )

    val booking =
        bookingService.bookFlight("F101")

    println(booking)
}
```

---

## Output

```text
Paid using UPI

Booking(
 id=B101,
 status=CONFIRMED
)
```

> 💡 **Remember**
>
> `main()` is **not** where business logic belongs.
>
> It simply demonstrates that your design works by wiring the objects together and invoking the service.

---

# 🚀 Complete LLD Interview Flow

Whenever an interviewer gives you an LLD problem, follow this sequence.

---

## ✅ Step 1: Understand the Requirement

### Example

> **"Design Flight Booking System."**

Ask:

* What are the main use cases?
* What operations should be supported?

---

## ✅ Step 2: Identify Domain Models

Find the nouns.

```text
Flight
Passenger
Booking
Payment
Seat
```

Create:

```text
data class
enum
sealed class
```

---

## ✅ Step 3: Create Repository

Ask:

> **"How will I access data?"**

Create:

```text
Interface
Implementation
```

---

## ✅ Step 4: Identify Changing Behaviour

Ask:

> **"What will change frequently?"**

Create:

```text
Strategy
```

Examples:

```text
PaymentStrategy
PricingStrategy
SortingStrategy
SplitStrategy
```

---

## ✅ Step 5: Add Factory (If Required)

Ask:

> **"Is object creation becoming complex?"**

Create:

```text
Factory
```

> ⚠️ **Important:** Factory is optional. Do not force a Factory into every problem.

---

## ✅ Step 6: Create Service

Put **business rules** here.

Examples:

```text
BookingService
PaymentService
ExpenseService
SearchService
```

---

## ✅ Step 7: Create `main()`

Demonstrate the flow.

```text
Input
  ↓
Service Call
  ↓
Business Logic
  ↓
Output
```

---

# 🧩 Typical Kotlin Project Structure

When practising in a Kotlin Project environment, keep the structure simple.

```text
src/main/kotlin/
│
├── model/
│   └── Models.kt
│
├── repository/
│   └── FlightRepository.kt
│
├── strategy/
│   └── PaymentStrategy.kt
│
├── factory/
│   └── PaymentFactory.kt
│
├── service/
│   └── FlightBookingService.kt
│
└── Main.kt
```

### Remember

You **do not need every folder for every problem**.

For example, if Factory is not required:

```text
model/
repository/
strategy/
service/
Main.kt
```

That is perfectly valid.

> **DRSFSM is a thinking framework, not a mandatory folder structure.**

---

# 📌 Interview Coding Checklist

Before you stop coding, quickly verify:

* [ ] Requirements understood
* [ ] Domain models created
* [ ] Repository interface + implementation
* [ ] Strategy used where behaviour changes
* [ ] Factory added only if needed
* [ ] Business logic inside Service
* [ ] `main()` demonstrates the complete flow
* [ ] Code is clean and readable

---

# 🌍 Examples Where This Framework Applies

The DRSFSM Framework can be applied to many LLD interview problems.

---

## 🏡 Property Listing Service

### Domain

```text
Property
User
Listing
Booking
```

### Strategy

```text
SearchStrategy
PricingStrategy
```

### Service

```text
PropertyListingService
```

---

## ✈️ Flight Aggregation System

### Domain

```text
Flight
Airline
SearchRequest
```

### Strategy

```text
SortingStrategy
FilterStrategy
```

### Service

```text
FlightSearchService
```

---

## 💰 Expense Management System

### Domain

```text
User
Expense
Group
Split
```

### Strategy

```text
SplitStrategy

Equal Split
Percentage Split
Exact Split
```

### Service

```text
ExpenseService
```

---

## 🏨 Hotel Booking System

### Domain

```text
Hotel
Room
Guest
Reservation
```

### Strategy

```text
PricingStrategy
PaymentStrategy
```

### Service

```text
HotelBookingService
```

---

## ✈️ Flight Booking System

### Domain

```text
Passenger
Flight
Seat
Payment
Booking
```

### Strategy

```text
PaymentStrategy
```

### Service

```text
FlightBookingService
```

---

## 🗺️ Travel Itinerary Management

### Domain

```text
Traveler
Trip
Itinerary
Activity
Destination
```

### Strategy

```text
SortingStrategy
PlanningStrategy
```

### Service

```text
ItineraryService
```

---

# ⚠️ Important Interview Reminder

❌ Do **NOT** spend time implementing:

* Real APIs
* Database
* Retrofit
* Networking
* UI
* Authentication

---

## ✅ Focus On

* Clean object design
* SOLID principles
* Separation of responsibility
* Extensibility
* Business logic

> 💡 **Remember**
>
> The interviewer is evaluating your **design thinking**, not production infrastructure.

---

# 🧪 How to Practise

For each LLD problem:

```text
1. Understand the story
        ↓
2. Identify nouns
        ↓
3. Create Domain Models
        ↓
4. Create Repository
        ↓
5. Identify changing behaviour
        ↓
6. Add Strategy
        ↓
7. Add Factory if required
        ↓
8. Create Service
        ↓
9. Create main()
        ↓
10. Run and verify output
```

The goal is not to finish as quickly as possible.

The goal is to understand:

> **Who is communicating with whom?**

and:

> **Why does each class exist?**

---

# 🧠 Final Memory Trick

Whenever you see an LLD problem, simply ask yourself:

```text
What things exist?
        ↓
DOMAIN

Where does data come from?
        ↓
REPOSITORY

What behaviour changes?
        ↓
STRATEGY

Who creates objects?
        ↓
FACTORY

What action does user perform?
        ↓
SERVICE

How do I prove it works?
        ↓
MAIN
```

---

# 🎯 One-Line Summary

```text
Story
   ↓
Domain
   ↓
Repository
   ↓
Strategy
   ↓
Factory (if required)
   ↓
Service
   ↓
Main()
```

---

# 🧠 The Mental Model to Remember in the Interview

If you forget everything else, remember this:

```text
               REAL WORLD STORY
                      │
                      ▼
                  DOMAIN
               "What exists?"
                      │
                      ▼
                REPOSITORY
             "Where is the data?"
                      │
                      ▼
                 STRATEGY
           "What behaviour varies?"
                      │
                      ▼
                  FACTORY
          "Who creates the object?"
                      │
                      ▼
                  SERVICE
          "What does the user do?"
                      │
                      ▼
                   MAIN
           "Does my design work?"
```

You are simply converting:

```text
REAL WORLD
    ↓
OBJECTS
    ↓
RELATIONSHIPS
    ↓
BEHAVIOUR
    ↓
BUSINESS LOGIC
    ↓
RUNNABLE CODE
```

---

# 🚀 Conclusion

Master this flow and you can approach most **Low-Level Design (LLD) coding interviews** confidently.

Instead of memorizing solutions, learn to think systematically.

Every LLD problem is simply another real-world story waiting to be converted into clean, extensible code.

> **Don't memorize the solution.**
>
> **Memorize the process.**

Happy Coding! 🚀
