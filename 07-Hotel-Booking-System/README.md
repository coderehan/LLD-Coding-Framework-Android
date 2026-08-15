Yes, understood. I checked your current **Hotel Booking System README**. ([GitHub][1])

Just like we did for Flight Booking, I'll keep your **existing logic/content unchanged** and improve only the **meaningful comments** and the **Main output** so it's easier to understand and remember during CoderPad practice.

### `HotelBookingService.kt`

```kotlin
/**
 * Service responsible for handling the main hotel booking business logic.
 *
 * It coordinates:
 * - Hotel and room data through HotelRepository
 * - Room availability and locking
 * - Dynamic pricing through PricingStrategy
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
     * 2. Find the requested room inside the hotel.
     * 3. Check whether the room is available.
     * 4. Lock/book the room to prevent another booking.
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
        // Booking cannot continue if the room doesn't exist.
        val room = hotel.rooms.find {
            it.roomNo == roomNo
        } ?: throw NoSuchElementException("Room not found")

        /**
         * Lock the room before completing the booking.
         *
         * synchronized() makes the availability check and status update
         * thread-safe, preventing two users from booking the same room
         * at the same time.
         */
        synchronized(room) {

            // Only AVAILABLE rooms can be booked.
            if (room.status != RoomStatus.AVAILABLE) {
                throw IllegalStateException("Room already booked")
            }

            // Mark the room as BOOKED once it has been selected.
            room.status = RoomStatus.BOOKED
        }

        // PricingStrategy decides how the final price is calculated.
        //
        // Example:
        // StandardPricingStrategy → base price × nights
        // SurgePricingStrategy    → base price × nights × 1.2
        //
        // HotelBookingService doesn't need to know the pricing formula.
        val totalPrice = pricingStrategy.calculatePrice(
            room.basePrice,
            nights
        )

        // Create the booking using the selected hotel, room,
        // guest, number of nights, and calculated price.
        val booking = Booking(
            id = "HB-${bookings.size + 1}",
            hotel = hotel,
            room = room,
            guest = guest,
            nights = nights,
            totalPrice = totalPrice
        )

        // Store the booking using its unique booking ID.
        // This allows us to find it later for cancellation.
        bookings[booking.id] = booking

        return booking
    }

    /**
     * Cancels an existing hotel booking.
     *
     * Cancellation flow:
     * 1. Find the booking using its booking ID.
     * 2. Mark the booking as CANCELLED.
     * 3. Release the room so another guest can book it.
     */
    fun cancelBooking(bookingId: String) {

        // Find the booking using its unique booking ID.
        // Cancellation cannot continue if the booking doesn't exist.
        val booking = bookings[bookingId]
            ?: throw NoSuchElementException("Booking not found")

        // Mark the booking as cancelled.
        booking.status = BookingStatus.CANCELLED

        // Release the room and make it available again.
        booking.room.status = RoomStatus.AVAILABLE
    }
}
```

### `Main.kt` — meaningful comments + complete output

```kotlin
fun main() {

    // Create a sample hotel with two rooms.
    // In a real application, this information would come from an API/database.
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

    // Create a sample guest who will book the hotel rooms.
    val guest = Guest(
        "G1",
        "Rehan"
    )

    // Book room 101 for 3 nights using standard pricing.
    //
    // Expected flow:
    // Hotel found → Room available → Room booked
    // → Standard price calculated → Booking confirmed
    val booking = service.bookRoom(
        "HT1",
        "101",
        guest,
        nights = 3,
        pricingStrategy = StandardPricingStrategy()
    )

    println(
        """
        ===== Booking Details =====
        Booking ID : ${booking.id}
        Guest      : ${booking.guest.name}
        Hotel      : ${booking.hotel.name}
        City       : ${booking.hotel.city}
        Room       : ${booking.room.roomNo}
        Room Type  : ${booking.room.type}
        Nights     : ${booking.nights}
        Price      : ₹${booking.totalPrice}
        Status     : ${booking.status}
        ===========================
        """.trimIndent()
    )

    // Book another room using surge pricing.
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
        """
        ===== Booking Details =====
        Booking ID : ${surgeBooking.id}
        Guest      : ${surgeBooking.guest.name}
        Hotel      : ${surgeBooking.hotel.name}
        City       : ${surgeBooking.hotel.city}
        Room       : ${surgeBooking.room.roomNo}
        Room Type  : ${surgeBooking.room.type}
        Nights     : ${surgeBooking.nights}
        Price      : ₹${surgeBooking.totalPrice}
        Status     : ${surgeBooking.status}
        ===========================
        """.trimIndent()
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

### Expected output

```text
===== Booking Details =====
Booking ID : HB-1
Guest      : Rehan
Hotel      : Taj Residency
City       : Bengaluru
Room       : 101
Room Type  : SINGLE
Nights     : 3
Price      : ₹6000.0
Status     : CONFIRMED
===========================

===== Booking Details =====
Booking ID : HB-2
Guest      : Rehan
Hotel      : Taj Residency
City       : Bengaluru
Room       : 102
Room Type  : SUITE
Nights     : 2
Price      : ₹14400.0
Status     : CONFIRMED
===========================

Room 101 status after cancel: AVAILABLE
```

### 🧠 The Hotel Booking story to remember

```text
Guest
  ↓
Select Hotel
  ↓
Select Room
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
```

The biggest difference from your **Flight Booking LLD** is:

```text
Flight Booking → PaymentStrategy
Hotel Booking  → PricingStrategy
```
