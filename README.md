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
