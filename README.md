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
