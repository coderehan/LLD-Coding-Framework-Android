# 🏨 LLD TOPIC: HOTEL BOOKING SYSTEM

### (Hotel reservations - Booking.com, Agoda, Expedia, Airbnb, OYO, MakeMyTrip, Goibibo, Trivago, Hotels.com, Marriott, Hilton, Taj, Hotels)

---

## 🎯 Overview

Very similar to Flight Booking, but the "unit" being booked is a Room instead of a Seat, and price often depends on dates (dynamic/seasonal pricing) → good place to show Strategy pattern for PRICING instead of payment.

### 🔑 Key ideas to show:

- 🔒 Room locking before booking (concurrency awareness)
- 💰 Strategy pattern for pricing
- 🔄 Booking / Room status for lifecycle
- 🏨 Service class for booking + cancellation

---

## 🧩 FRAMEWORK

| Step | Component | Purpose |
|---|---|---|
| 1 | Enums + Data classes | Model rooms, hotels, guests, and bookings |
| 2 | Repository | Fetch/search hotels |
| 3 | Strategy interface | How the price is calculated |
| 4 | Service class | Booking + room locking + pricing |
| 5 | `main()` | Proves everything works |

---

## 📁 PROJECT STRUCTURE

```text
HotelBookingSystem/
│
└── src/main/kotlin/
    │
    ├── model/
    │   ├── Guest.kt
    │   ├── Room.kt
    │   ├── Hotel.kt
    │   ├── Booking.kt
    │   ├── RoomType.kt
    │   ├── RoomStatus.kt
    │   └── BookingStatus.kt
    │
    ├── repository/
    │   ├── HotelRepository.kt
    │   └── InMemoryHotelRepository.kt
    │
    ├── strategy/
    │   ├── PricingStrategy.kt
    │   ├── StandardPricingStrategy.kt
    │   └── SurgePricingStrategy.kt
    │
    ├── service/
    │   └── HotelBookingService.kt
    │
    └── Main.kt
```

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

enum class RoomType {
    SINGLE,
    DOUBLE,
    SUITE
}

enum class RoomStatus {
    AVAILABLE,
    BOOKED
}

enum class BookingStatus {
    CONFIRMED,
    CANCELLED
}

data class Guest(
    val id: String,
    val name: String
)

data class Room(
    val roomNo: String,
    val type: RoomType,
    val basePrice: Double,

    // 'var status' because a room's status changes over time.
    var status: RoomStatus = RoomStatus.AVAILABLE
)

data class Hotel(
    val id: String,
    val name: String,
    val city: String,
    val rooms: MutableList<Room>
)

data class Booking(
    val id: String,
    val hotel: Hotel,
    val room: Room,
    val guest: Guest,
    val nights: Int,
    val totalPrice: Double,

    // Booking starts as CONFIRMED when the booking is successfully created.
    var status: BookingStatus = BookingStatus.CONFIRMED
)
```

### 💡 What these models represent

- `Guest` → person who books the hotel room.
- `Hotel` → contains hotel information and its rooms.
- `Room` → represents an individual room that can be booked.
- `Booking` → connects the guest, hotel, room, number of nights, and final price.
- `RoomStatus` → tells whether a room is available or already booked.
- `BookingStatus` → represents whether the booking is confirmed or cancelled.

---

# 2️⃣ REPOSITORY

```kotlin
interface HotelRepository {

    // Get one specific hotel using its unique hotel ID.
    fun getById(id: String): Hotel?

    // Search all hotels available in a particular city.
    fun searchByCity(city: String): List<Hotel>
}
```

### 💡 Repository responsibility

The repository is responsible for accessing `Hotel` data.

- `getById()` → gets one specific hotel.
- `searchByCity()` → returns multiple hotels matching the requested city.

---

## 🏨 In-Memory Hotel Repository

```kotlin
/**
 * In-memory implementation of HotelRepository.
 *
 * Repository is responsible for accessing Hotel data.
 * In a real application, this data could come from an API or database.
 * For this LLD interview, we use in-memory data instead.
 */
class InMemoryHotelRepository(
    private val hotels: MutableList<Hotel>
) : HotelRepository {

    /**
     * Finds one specific Hotel using its unique hotel ID.
     *
     * Returns:
     * - Hotel → if a matching hotel exists
     * - null  → if no hotel is found
     */
    override fun getById(id: String): Hotel? {
        return hotels.find { it.id == id }
    }
    /**
     * Searches Hotel models based on city.
     *
     * Returns a List because multiple hotels
     * can exist in the same city.
     */
    override fun searchByCity(city: String): List<Hotel> {
        return hotels.filter { it.city == city }
    }
```

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = **HOW the price is calculated.**

For example:

- Normal pricing
- Weekend pricing
- Seasonal surge pricing

### 💰 Pricing Strategy

```kotlin
/**
 * Strategy interface for calculating hotel booking prices.
 *
 * Different pricing strategies can provide their own
 * implementation without changing HotelBookingService.
 */
interface PricingStrategy {

    /**
     * Calculates the final price for the booking.
     *
     * @param basePrice Base price of the selected room.
     * @param nights Number of nights the guest wants to stay.
     * @return Final calculated booking price.
     */
    fun calculatePrice(
        basePrice: Double,
        nights: Int
    ): Double
}
```

---

## 💵 Standard Pricing Strategy

> Simple pricing: `basePrice * nights`, no surge.

```kotlin
/**
 * Standard pricing implementation.
 *
 * The guest simply pays:
 *
 * Base price × Number of nights
 */
class StandardPricingStrategy : PricingStrategy {

    override fun calculatePrice(
        basePrice: Double,
        nights: Int
    ): Double {

        return basePrice * nights
    }
}
```

### 💡 Example

```text
Base price = ₹2,000
Nights     = 3

Total = ₹2,000 × 3
      = ₹6,000
```

---

## 📈 Surge Pricing Strategy

> Weekend/seasonal pricing: adds a 20% surge on top.

```kotlin
/**
 * Surge pricing implementation.
 *
 * Adds a 20% increase to the normal calculated price.
 */
class SurgePricingStrategy : PricingStrategy {

    override fun calculatePrice(
        basePrice: Double,
        nights: Int
    ): Double {

        return basePrice * nights * 1.2
    }
}
```

### 💡 Example

```text
Base price = ₹6,000
Nights     = 2

Normal = ₹6,000 × 2
       = ₹12,000

Surge = ₹12,000 × 1.2
      = ₹14,400
```

---

# 4️⃣ SERVICE CLASS

```kotlin
/**
 * Service responsible for handling the main hotel booking business logic.
 *
 * It coordinates:
 * - Hotel data through HotelRepository
 * - Room availability and state
 * - Pricing through PricingStrategy
 * - Booking creation and cancellation
 */
class HotelBookingService(
    private val hotelRepo: HotelRepository,
    private val bookings: MutableMap<String, Booking> = mutableMapOf()
) {

    /**
     * Books a specific room for a guest.
     *
     * Booking flow:
     * 1. Find the requested hotel using its hotel ID.
     * 2. Find the requested room.
     * 3. Check room availability.
     * 4. Lock/book the room to prevent another booking attempt.
     * 5. Calculate the total price using PricingStrategy.
     * 6. Create and store the booking.
     */
    fun bookRoom(
        hotelId: String,
        roomNo: String,
        guest: Guest,
        nights: Int,
        pricingStrategy: PricingStrategy
    ): Booking {

        // Get the requested hotel from the repository.
        // If the hotel doesn't exist, booking cannot continue.
        val hotel = hotelRepo.getById(hotelId)
            ?: throw NoSuchElementException("Hotel not found")

        // Find the requested room inside the selected hotel.
        // If the room doesn't exist, booking cannot continue.
        val room = hotel.rooms.find {
            it.roomNo == roomNo
        } ?: throw NoSuchElementException("Room not found")

        /**
         * Lock/check room availability inside a synchronized block.
         *
         * This makes the availability check and status update
         * thread-safe and prevents two users from booking
         * the same room at the same time.
         */
        synchronized(room) {

            // Only AVAILABLE rooms can be booked.
            if (room.status != RoomStatus.AVAILABLE) {
                throw IllegalStateException("Room already booked")
            }

            // Mark the room as BOOKED after confirming availability.
            room.status = RoomStatus.BOOKED
        }

        // Use the selected PricingStrategy to calculate the final price.
        //
        // HotelBookingService does not need to know how the price
        // is calculated. The selected strategy handles that logic.
        val totalPrice = pricingStrategy.calculatePrice(
            room.basePrice,
            nights
        )

        // Create a Booking containing the selected hotel, room,
        // guest, number of nights, and calculated total price.
        val booking = Booking(
            id = "HB-${bookings.size + 1}",
            hotel = hotel,
            room = room,
            guest = guest,
            nights = nights,
            totalPrice = totalPrice
        )

        // Store the booking using its unique booking ID.
        // This allows us to find the booking later for cancellation.
        bookings[booking.id] = booking

        return booking
    }

    /**
     * Cancels an existing booking.
     *
     * Cancellation flow:
     * 1. Find the booking using its booking ID.
     * 2. Mark the booking as CANCELLED.
     * 3. Release the room so another guest can book it.
     */
    fun cancelBooking(bookingId: String) {

        // Find the existing booking using its unique booking ID.
        // Cancellation cannot continue if the booking doesn't exist.
        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        // Mark the booking as cancelled.
        booking.status = BookingStatus.CANCELLED

        // Release the room and make it available for future bookings.
        booking.room.status =
            RoomStatus.AVAILABLE // free the room again
    }
}
```

### 🧠 Service business logic

#### Booking

```text
Hotel ID
   ↓
Find Hotel
   ↓
Room Number
   ↓
Find Room
   ↓
Check AVAILABLE
   ↓
BOOK Room
   ↓
PricingStrategy
   ↓
Calculate Total Price
   ↓
Create Booking
   ↓
Store Booking
```

#### Cancellation

```text
Booking ID
    ↓
Find Booking
    ↓
CANCELLED
    ↓
Room → AVAILABLE
```

---

# 5️⃣ `main()`

> Always write a `main()` to **PROVE your code compiles and runs.**

```kotlin
fun main() {

    // Create a sample Hotel with two rooms.
    // In a real application, this information would come from an API or database.
    val hotel = Hotel(
        id = "HT1",
        name = "Taj Residency",
        city = "Bengaluru",
        rooms = mutableListOf(
            Room(
                "101",
                RoomType.SINGLE,
                basePrice = 2000.0
            ),
            Room(
                "102",
                RoomType.SUITE,
                basePrice = 6000.0
            )
        )
    )

    // Use an in-memory repository for this LLD exercise.
    // The repository provides access to our Hotel domain model.
    val repo = InMemoryHotelRepository(
        mutableListOf(hotel)
    )

    // Create the HotelBookingService and inject the repository.
    // The service contains the main hotel booking business logic.
    val service = HotelBookingService(repo)

    // Create a sample guest who will book the hotel.
    val guest = Guest(
        "G1",
        "Rehan"
    )

    // Book room 101 for 3 nights using standard pricing.
    //
    // Expected flow:
    // Hotel found → Room available → Room booked
    // → Price calculated → Booking confirmed
    val booking = service.bookRoom(
        "HT1",
        "101",
        guest,
        nights = 3,
        pricingStrategy = StandardPricingStrategy()
    )

    println(
        "Booking: Room ${booking.room.roomNo}, " +
        "Total = ₹${booking.totalPrice}"
    )

    // Book another room for 2 nights using surge pricing.
    // This demonstrates that we can change the pricing behavior
    // without changing HotelBookingService.
    val surgeBooking = service.bookRoom(
        "HT1",
        "102",
        guest,
        nights = 2,
        pricingStrategy = SurgePricingStrategy()
    )

    println(
        "Booking: Room ${surgeBooking.room.roomNo}, " +
        "Total = ₹${surgeBooking.totalPrice}"
    )

    // Cancel the first booking.
    // Cancellation changes the booking status to CANCELLED
    // and releases room 101 back to AVAILABLE.
    service.cancelBooking(booking.id)

    println(
        "Room ${booking.room.roomNo} status after cancel: " +
        "${booking.room.status}"
    )
}
```

---

# 🖥️ Output

```text
Booking: Room 101, Total = ₹6000.0
Booking: Room 102, Total = ₹14400.0
Room 101 status after cancel: AVAILABLE
```

---

# 🧠 Interview Focus

The important parts of this topic are:

> 🏨 **Room instead of Seat** → same booking concept as Flight Booking.

> 🔒 **Room Locking** → `synchronized(room)` prevents two users from booking the same room simultaneously.

> 💰 **Pricing Strategy** → allows standard, weekend, or seasonal pricing without changing `HotelBookingService`.

> 🔄 **Booking Status** → represents the booking lifecycle.

> 🧩 **Service Class** → coordinates room availability, pricing, booking creation, and cancellation.
