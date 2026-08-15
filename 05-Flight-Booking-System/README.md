# ✈️ LLD TOPIC: FLIGHT BOOKING SYSTEM

### *(Airline / flight reservation - , IndiGo, Air India, Emirates, Qatar Airways, Singapore Airlines, Lufthansa, British Airways, United, Airlines, Delta Air Lines, American Airlines, MakeMyTrip, Cleartrip, ixigo)*

---

## 🎯 Overview

This is the **classic booking flow topic**.

### 🔑 Key ideas to show:

* 🔒 **Seat locking before payment** *(concurrency awareness)*
* 💳 **Strategy pattern** for payment method
* 🔄 **State-like enum** for booking lifecycle

---

## 🧩 FRAMEWORK

| Step  | Component            | Purpose                                        |
| ----- | -------------------- | ---------------------------------------------- |
| **1** | Enums + Data classes | Model seats, passengers, flights, and bookings |
| **2** | Repository           | Fetch/search flights                           |
| **3** | Strategy interface   | How payment is done                            |
| **4** | Service class        | Booking + seat locking + payment flow          |
| **5** | `main()`             | Proves everything works                        |

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

enum class SeatType {
    ECONOMY,
    BUSINESS
}

enum class SeatStatus {
    AVAILABLE,
    LOCKED,
    BOOKED
}

enum class BookingStatus {
    PENDING,
    CONFIRMED,
    CANCELLED
}

data class Passenger(
    val id: String,
    val name: String
)

// 'var status' because a seat's status changes over time
data class Seat(
    val seatNo: String,
    val type: SeatType,
    var status: SeatStatus = SeatStatus.AVAILABLE
)

data class Flight(
    val id: String,
    val flightNumber: String,
    val from: String,
    val to: String,
    val price: Double,
    val seats: MutableList<Seat>
)

data class Booking(
    val id: String,
    val flight: Flight,
    val passenger: Passenger,
    val seat: Seat,
    var status: BookingStatus = BookingStatus.PENDING
)
```

---

# 2️⃣ REPOSITORY

```kotlin
interface FlightRepository {

    // Get one specific flight using its flight number
    fun getByFlightNumber(flightNo: String): Flight?

    // Search all available flights between two locations
    fun searchFlights(
        source: String,
        destination: String
    ): List<Flight>
}
```

### In-Memory Flight Repository

```kotlin
/**
 * In-memory implementation of FlightRepository.
 *
 * Repository is responsible for accessing Flight data.
 * In a real application, this data could come from an API or database.
 * For this LLD interview, we use in-memory data instead.
 */
class InMemoryFlightRepository(
    private val flights: MutableList<Flight>
) : FlightRepository {

    /**
     * Finds one specific Flight using its flight number.
     *
     * Returns:
     * - Flight → if a matching flight exists
     * - null   → if no flight is found
     */
    override fun getByFlightNumber(flightNo: String): Flight? {
        return flights.find { flight ->
            flight.flightNo == flightNo
        }
    }

    /**
     * Searches Flight models based on source and destination.
     *
     * Returns all matching flights because
     * multiple flights can exist for the same route.
     */
    override fun searchFlights(
        source: String,
        destination: String
    ): List<Flight> {
        return flights.filter { flight ->
            flight.source == source &&
            flight.destination == destination
        }
    }
}
```

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = **HOW payment is done.**

### 💳 Payment Strategy

```kotlin
/**
 * Strategy interface for handling different payment methods.
 *
 * Different payment methods such as UPI, Card, or Wallet
 * can provide their own implementation of this interface.
 */
interface PaymentStrategy {

    /**
     * Processes payment for the given amount.
     *
     * @param amount Amount that needs to be paid.
     * @return true  → payment was successful
     *         false → payment failed
     */
    fun pay(amount: Double): Boolean
}
```

---

## 💳 Credit Card Payment

```kotlin
class CreditCardPayment : PaymentStrategy {

    override fun pay(amount: Double): Boolean {

        println("Charged ₹$amount to credit card")

        return true // pretend payment always succeeds for this demo
    }
}
```

---

## 📱 UPI Payment

```kotlin
class UpiPayment : PaymentStrategy {

    override fun pay(amount: Double): Boolean {

        println("Charged ₹$amount via UPI")

        return true
    }
}
```

---

# 4️⃣ SERVICE CLASS

```kotlin
class BookingService(
    private val flightRepo: FlightRepository,
    private val bookings: MutableMap<String, Booking> = mutableMapOf()
) {

    fun bookSeat(
        flightId: String,
        seatNo: String,
        passenger: Passenger,
        payment: PaymentStrategy
    ): Booking {

        val flight = flightRepo.getById(flightId)
            ?: throw NoSuchElementException("Flight not found")

        val seat = flight.seats.find {
            it.seatNo == seatNo
        } ?: throw NoSuchElementException("Seat not found")

        // LOCK the seat first so two people can't book the same seat at once.
        // 'synchronized' makes this block thread-safe - good to mention in interview.
        synchronized(seat) {

            if (seat.status != SeatStatus.AVAILABLE) {
                throw IllegalStateException(
                    "Seat already booked or locked"
                )
            }

            seat.status = SeatStatus.LOCKED
        }

        // Now try payment
        val paymentSuccess = payment.pay(flight.price)

        // Update seat + booking status based on payment result
        seat.status =
            if (paymentSuccess) {
                SeatStatus.BOOKED
            } else {
                SeatStatus.AVAILABLE
            }

        val booking = Booking(
            id = "BK-${bookings.size + 1}",
            flight = flight,
            passenger = passenger,
            seat = seat,
            status =
                if (paymentSuccess) {
                    BookingStatus.CONFIRMED
                } else {
                    BookingStatus.CANCELLED
                }
        )

        bookings[booking.id] = booking

        return booking
    }

    fun cancelBooking(bookingId: String) {

        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        booking.status = BookingStatus.CANCELLED

        booking.seat.status =
            SeatStatus.AVAILABLE // free up the seat again
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

    val flight = Flight(
        id = "F1",
        flightNumber = "AI101",
        from = "BLR",
        to = "DEL",
        price = 4500.0,
        seats = mutableListOf(
            Seat("1A", SeatType.ECONOMY),
            Seat("1B", SeatType.BUSINESS)
        )
    )

    val repo = InMemoryFlightRepository(
        mutableListOf(flight)
    )

    val service = BookingService(repo)

    val passenger = Passenger(
        "P1",
        "Rehan"
    )

    val booking = service.bookSeat(
        "F1",
        "1A",
        passenger,
        CreditCardPayment()
    )

    println(
        "Booking status: ${booking.status}, " +
        "Seat: ${booking.seat.seatNo}"
    )

    // Try to book the SAME seat again -> should fail because it's already BOOKED
    try {

        service.bookSeat(
            "F1",
            "1A",
            passenger,
            UpiPayment()
        )

    } catch (e: IllegalStateException) {

        println(
            "Expected error: ${e.message}"
        )
    }

    // Cancel and re-book
    service.cancelBooking(booking.id)

    val newBooking = service.bookSeat(
        "F1",
        "1A",
        passenger,
        UpiPayment()
    )

    println(
        "Re-booked successfully: ${newBooking.status}"
    )
}
```

---

## 🧠 Interview Focus

The important parts of this topic are:

> 🔒 **Seat Locking** → prevents two users from booking the same seat.

> 💳 **Payment Strategy** → allows different payment methods without changing booking logic.

> 🔄 **Booking / Seat Status** → represents the lifecycle of the booking and seat.

> 🧩 **Service Class** → coordinates the complete booking flow.

