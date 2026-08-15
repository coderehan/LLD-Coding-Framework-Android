Yes. I’ve now used the **exact content/code currently in your Hotel Booking README** and I’m **not changing the implementation, names, methods, or logic**. I’m only making the comments more meaningful and adding one clear sample output. ([GitHub][1])

Below is the complete `README.md` you can copy-paste.

````markdown
# 🏨 LLD TOPIC: HOTEL BOOKING SYSTEM

### (Hotel reservations - Booking.com, Agoda, Expedia, Airbnb, OYO, MakeMyTrip, Goibibo, Trivago, Hotels.com, Marriott, Hilton, Taj, Hotels)

---

## 🎯 Overview

Very similar to Flight Booking, but the "unit" being booked is a Room instead of a Seat, and price often depends on dates (dynamic/seasonal pricing) → good place to show Strategy pattern for PRICING instead of payment.

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

    fun getById(id: String): Hotel?

    fun searchByCity(city: String): List<Hotel>
}
```

### 💡 Repository responsibility

The repository is responsible for accessing hotel data.

- `getById()` → gets one specific hotel using its unique ID.
- `searchByCity()` → returns all hotels available in a particular city.

---

## In-Memory Hotel Repository

```kotlin
class InMemoryHotelRepository(
    private val hotels: MutableList<Hotel>
) : HotelRepository {

    override fun getById(id: String) =
        hotels.find { it.id == id }

    override fun searchByCity(city: String) =
        hotels.filter { it.city == city }
}
```

### 💡 Meaning

`InMemoryHotelRepository` is our temporary data source for the LLD exercise.

Instead of connecting to a real database or API, we keep hotels inside a `MutableList`.

The repository works with the `Hotel` domain model to:

- Find one hotel by ID.
- Find multiple hotels by city.

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = HOW the price per night is calculated.

For example:

- Normal pricing
- Weekend pricing
- Seasonal surge pricing

### 💰 Pricing Strategy

```kotlin
interface PricingStrategy {

    fun calculatePrice(
        basePrice: Double,
        nights: Int
    ): Double
}
```

### 💡 Why Strategy?

The `HotelBookingService` should not contain different pricing formulas.

Instead, we can provide different pricing strategies.

```text
HotelBookingService
        ↓
PricingStrategy
        ↓
 ┌───────────────┐
 │               │
Standard      Surge
Pricing       Pricing
```

The service simply asks:

> "Calculate the price."

The selected strategy decides **how** the price is calculated.

---

## 💵 Standard Pricing Strategy

> Simple pricing: `basePrice * nights`, no surge

```kotlin
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

> Weekend/seasonal pricing: adds a 20% surge on top

```kotlin
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
class HotelBookingService(
    private val hotelRepo: HotelRepository,
    private val bookings: MutableMap<String, Booking> = mutableMapOf()
) {

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

        // Check and update the room status inside a synchronized block.
        //
        // This makes the operation thread-safe and prevents two users
        // from booking the same room at the same time.
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
        // HotelBookingService does not need to know whether the pricing
        // is standard, weekend, or seasonal.
        val totalPrice = pricingStrategy.calculatePrice(
            room.basePrice,
            nights
        )

        // Create a Booking containing all information related to this reservation.
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

    fun cancelBooking(bookingId: String) {

        // Find the existing booking using its unique booking ID.
        // If it doesn't exist, cancellation cannot continue.
        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        // Mark the booking as cancelled.
        booking.status = BookingStatus.CANCELLED

        // Release the room so another guest can book it.
        booking.room.status =
            RoomStatus.AVAILABLE // free the room again
    }
}
```

### 🧠 Service business logic

The main booking flow is:

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

Cancellation:

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

> Always write a `main()` to PROVE your code compiles and runs.

```kotlin
fun main() {

    // Create a sample hotel with two rooms.
    // In a real application, this information would come from
    // an API or database.
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

    // Create an in-memory repository containing our hotel.
    // The repository provides access to the Hotel domain model.
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
    // Hotel found → Room found → Room available
    // → Room booked → Price calculated → Booking confirmed
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
    //
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
    // The booking becomes CANCELLED and room 101 becomes AVAILABLE again.
    service.cancelBooking(booking.id)

    println(
        "Room ${booking.room.roomNo} status after cancel: " +
        "${booking.room.status}"
    )
}
```

---

# 🖥️ Sample Output

```text
Booking: Room 101, Total = ₹6000.0
Booking: Room 102, Total = ₹14400.0
Room 101 status after cancel: AVAILABLE
```

---

# 🧠 Interview Focus

> 🏨 **Room instead of Seat** → same booking concept as Flight Booking.

> 🔒 **Room locking** → `synchronized(room)` prevents two users from booking the same room simultaneously.

> 💰 **Pricing Strategy** → allows standard, weekend, or seasonal pricing without changing `HotelBookingService`.

> 🔄 **Booking Status** → represents the booking lifecycle.

> 🧩 **Service Class** → coordinates room availability, pricing, booking creation, and cancellation.
````


[1]: https://github.com/coderehan/LLD-Coding-Framework-Android/blob/master/07-Hotel-Booking-System/README.md "LLD-Coding-Framework-Android/07-Hotel-Booking-System/README.md at master · coderehan/LLD-Coding-Framework-Android · GitHub"
