# 🏨 LLD TOPIC: HOTEL BOOKING SYSTEM

### *(Hotel reservations - Booking.com, Agoda, Expedia, Airbnb, OYO, MakeMyTrip, Goibibo, Trivago, Hotels.com, Marriott, Hilton, Taj, Hotels)*

---

## 🎯 Overview

Very similar to **Flight Booking**, but the **"unit" being booked is a Room instead of a Seat**, and price often depends on dates *(dynamic/seasonal pricing)* → good place to show **Strategy pattern for PRICING instead of payment**.

---

## 🧩 FRAMEWORK

| Step  | Component            | Purpose                                   |
| ----- | -------------------- | ----------------------------------------- |
| **1** | Enums + Data classes | Model rooms, hotels, guests, and bookings |
| **2** | Repository           | Fetch/search hotels                       |
| **3** | Strategy interface   | How the price is calculated               |
| **4** | Service class        | Booking + room locking + pricing          |
| **5** | `main()`             | Proves everything works                   |

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

---

# 2️⃣ REPOSITORY

```kotlin
interface HotelRepository {

    fun getById(id: String): Hotel?

    fun searchByCity(city: String): List<Hotel>
}
```

### In-Memory Hotel Repository

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

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = **HOW the price per night is calculated.**

For example:

* Normal pricing
* Weekend pricing
* Seasonal surge pricing

### 💰 Pricing Strategy

```kotlin
interface PricingStrategy {

    fun calculatePrice(
        basePrice: Double,
        nights: Int
    ): Double
}
```

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

---

## 📈 Surge Pricing Strategy

> Weekend/seasonal pricing: adds a **20% surge** on top

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

        val hotel = hotelRepo.getById(hotelId)
            ?: throw NoSuchElementException("Hotel not found")

        val room = hotel.rooms.find {
            it.roomNo == roomNo
        } ?: throw NoSuchElementException("Room not found")

        // lock/check availability - thread-safe block, same idea as flight seat locking
        synchronized(room) {

            if (room.status != RoomStatus.AVAILABLE) {
                throw IllegalStateException("Room already booked")
            }

            room.status = RoomStatus.BOOKED
        }

        // Use the pricing strategy to calculate final price (this is our Strategy pattern)
        val totalPrice = pricingStrategy.calculatePrice(
            room.basePrice,
            nights
        )

        val booking = Booking(
            id = "HB-${bookings.size + 1}",
            hotel = hotel,
            room = room,
            guest = guest,
            nights = nights,
            totalPrice = totalPrice
        )

        bookings[booking.id] = booking

        return booking
    }

    fun cancelBooking(bookingId: String) {

        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        booking.status = BookingStatus.CANCELLED

        booking.room.status =
            RoomStatus.AVAILABLE // free the room again
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

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

    val repo = InMemoryHotelRepository(
        mutableListOf(hotel)
    )

    val service = HotelBookingService(repo)

    val guest = Guest(
        "G1",
        "Rehan"
    )

    // Book with standard pricing
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

    // Book another room with surge pricing (e.g. weekend rate)
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

    // Cancel and confirm room is free again
    service.cancelBooking(booking.id)

    println(
        "Room ${booking.room.roomNo} status after cancel: " +
        "${booking.room.status}"
    )
}
```

---

## 🧠 Interview Focus

> 🏨 **Room instead of Seat** → same booking concept as Flight Booking.

> 🔒 **Room locking** → `synchronized(room)` prevents two users from booking the same room simultaneously.

> 💰 **Pricing Strategy** → allows standard, weekend, or seasonal pricing without changing `HotelBookingService`.

> 🔄 **Booking Status** → represents the booking lifecycle.

> 🧩 **Service Class** → coordinates room availability, pricing, booking creation, and cancellation.

