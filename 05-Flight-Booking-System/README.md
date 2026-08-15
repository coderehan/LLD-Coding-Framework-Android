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
/**
 * Service responsible for handling the main flight booking business logic.
 *
 * It coordinates:
 * - Flight data through FlightRepository
 * - Seat availability and state
 * - Payment through PaymentStrategy
 * - Booking creation and cancellation
 */
class BookingService(
    private val flightRepo: FlightRepository,
    private val bookings: MutableMap<String, Booking> = mutableMapOf()
) {

    /**
     * Books a specific seat for a passenger.
     *
     * Booking flow:
     * 1. Find the requested flight using its flight number.
     * 2. Find the requested seat.
     * 3. Lock the seat to prevent another booking attempt.
     * 4. Process payment.
     * 5. If payment succeeds → BOOKED + CONFIRMED.
     * 6. If payment fails → AVAILABLE + CANCELLED.
     * 7. Create and store the booking.
     */
    fun bookSeat(
        flightNo: String,
        seatNo: String,
        passenger: Passenger,
        payment: PaymentStrategy
    ): Booking {

        // Get the requested flight using its flight number.
        // If the flight doesn't exist, booking cannot continue.
        val flight = flightRepo.getByFlightNumber(flightNo)
            ?: throw NoSuchElementException("Flight not found")

        // Find the requested seat inside the selected flight.
        // Booking cannot continue if the seat doesn't exist.
        val seat = flight.seats.find {
            it.seatNo == seatNo
        } ?: throw NoSuchElementException("Seat not found")

        /**
         * Temporarily lock the seat before processing payment.
         *
         * This prevents another thread/user from booking
         * the same seat at the same time.
         */
        synchronized(seat) {

            // Only AVAILABLE seats can be booked.
            if (seat.status != SeatStatus.AVAILABLE) {
                throw IllegalStateException(
                    "Seat already booked or locked"
                )
            }

            // Lock the seat while payment is being processed.
            seat.status = SeatStatus.LOCKED
        }

        // Payment is handled through PaymentStrategy,
        // so BookingService doesn't depend on a specific payment method.
        val paymentSuccess = payment.pay(flight.price)

        // Payment result determines the final seat state.
        //
        // Successful payment → seat becomes BOOKED.
        // Failed payment → seat becomes AVAILABLE again.
        seat.status =
            if (paymentSuccess) {
                SeatStatus.BOOKED
            } else {
                SeatStatus.AVAILABLE
            }

        // Create a booking based on the payment result.
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

        // Store the booking so it can be retrieved or cancelled later.
        bookings[booking.id] = booking

        return booking
    }

    /**
     * Cancels an existing booking.
     *
     * Cancellation flow:
     * 1. Find the booking using its booking ID.
     * 2. Mark the booking as CANCELLED.
     * 3. Release the seat so another passenger can book it.
     */
    fun cancelBooking(bookingId: String) {

        // Find the existing booking using its unique booking ID.
        // Cancellation cannot continue if it doesn't exist.
        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        // Mark the booking as cancelled.
        booking.status = BookingStatus.CANCELLED

        // Release the seat and make it available for future bookings.
        booking.seat.status = SeatStatus.AVAILABLE
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

    // Create a sample Flight with two seats.
    // In a real application, this data would come from an API or database.
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

    // Use an in-memory repository for this LLD exercise.
    // The repository provides access to our Flight domain model.
    val repo = InMemoryFlightRepository(
        mutableListOf(flight)
    )

    // Create the BookingService and inject the repository.
    // The service contains the main flight booking business logic.
    val service = BookingService(repo)

    // Create a sample passenger who will book the flight.
    val passenger = Passenger(
        "P1",
        "Rehan"
    )

    // Book seat 1A using Credit Card payment.
    //
    // Expected flow:
    // Flight found → Seat available → Seat locked
    // → Payment successful → Seat booked → Booking confirmed
    val booking = service.bookSeat(
        "AI101",
        "1A",
        passenger,
        CreditCardPayment()
    )

    println(
        "Booking status: ${booking.status}, " +
        "Seat: ${booking.seat.seatNo}"
    )

    // Try to book the SAME seat again.
    // The seat is already BOOKED, so the booking should fail.
    try {

        service.bookSeat(
            "AI101",
            "1A",
            passenger,
            UpiPayment()
        )

    } catch (e: IllegalStateException) {

        // Expected error because the seat is no longer AVAILABLE.
        println(
            "Expected error: ${e.message}"
        )
    }

    // Cancel the existing booking.
    // Cancellation changes the booking status to CANCELLED
    // and releases seat 1A back to AVAILABLE.
    service.cancelBooking(booking.id)

    // Book the same seat again using UPI payment.
    // This demonstrates that a cancelled seat can be booked again.
    val newBooking = service.bookSeat(
        "AI101",
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

