# 🚀 LLD Coding Interview Framework (DRSFSM)

> A simple framework to approach **Low-Level Design (LLD) coding rounds**.

The biggest mistake in LLD interviews is starting with classes immediately. Instead, start with a **real-world story** and convert that story into code.

---

# 📚 Table of Contents

- [The Flow](#-the-flow)
- [Memory Shortcut](#-memory-shortcut)
- [1. Domain Models (D)](#1--domain-models-d)
- [2. Repository (R)](#2--repository-r)

---

# 🌍 The Flow

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

| Letter | Meaning |
|---------|---------|
| **D** | Domain Models |
| **R** | Repository |
| **S** | Strategy |
| **F** | Factory |
| **S** | Service |
| **M** | Main |

---

# 1. 🧩 Domain Models (D)

> ## ❓ Question to ask
>
> **"What things exist in this real-world system?"**

Look for **nouns**.

Nouns usually become:

- 📦 Data classes
- 🚦 Enums
- 🔐 Sealed classes *(when different states contain different data)*

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

---

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

---

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

- Fetch data
- Save data
- Update data
- Delete data

---

> 💡 **Remember**
>
> Repository should **NOT** contain business rules.

---

---

# 3. 🎯 Strategy Pattern (S)

> ## ❓ Question to ask
>
> **"What behaviour can change in the future?"**

Whenever you see:

- Multiple ways of doing something
- Future extensions
- Different algorithms/rules

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

- WalletPayment
- ApplePayPayment
- GooglePayPayment

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

Instead of writing lots of `if-else` or `when` statements throughout the codebase, let the **Factory** create the correct object.

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

| Strategy 🧠 | Factory 🏭 |
|-------------|------------|
| **How should something be done?** | **Which object should I create?** |
| Focuses on **behaviour** | Focuses on **object creation** |
| Multiple algorithms | Multiple object types |
| Selected during execution | Creates the required implementation |

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

Now you choose **how** to pay.

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

---

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

---

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

- Create objects
- Inject dependencies
- Call service methods
- Print output

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

---

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

- What are the main use cases?
- What operations should be supported?

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
Output
```

---

# 📌 Interview Coding Checklist

Before you stop coding, quickly verify:

- ✅ Domain models created
- ✅ Repository interface + implementation
- ✅ Strategy used where behaviour changes
- ✅ Factory added only if needed
- ✅ Business logic inside Service
- ✅ `main()` demonstrates the complete flow
- ✅ Code is clean and readable

---

## 📖 Summary So Far

| Step | Ask Yourself |
|------|--------------|
| 🧩 Domain | What things exist? |
| 🗂 Repository | Where does data come from? |
| 🎯 Strategy | What behaviour changes? |
| 🏭 Factory | Who creates objects? |
| 🧠 Service | What action does the user perform? |
| ▶️ Main | How do I prove my design works? |

---

# ⚠️ Important Interview Reminder

❌ Do **NOT** spend time implementing:

- Real APIs
- Database
- Retrofit
- Networking
- UI
- Authentication

---

## ✅ Focus On

- Clean object design
- SOLID principles
- Separation of responsibility
- Extensibility
- Business logic

---

> 💡 **Remember**
>
> The interviewer is evaluating your **design thinking**, not production infrastructure.

---

# 📝 Quick Interview Checklist

Before ending the interview, verify:

- ✅ Requirements are understood
- ✅ Domain models are created
- ✅ Repository is separated
- ✅ Strategy is used where behaviour varies
- ✅ Factory is added only if required
- ✅ Service contains business logic
- ✅ `main()` demonstrates the complete flow
- ✅ Code is clean and readable

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

# 🚀 Conclusion

Master this flow and you can approach most **Low-Level Design (LLD)** coding interviews confidently.

Instead of memorizing solutions, learn to think systematically.

Every LLD problem is simply another real-world story waiting to be converted into clean, extensible code.

Happy Coding! 🚀
