LLD Coding Interview Framework (DRSFSM)

A simple framework to approach Low-Level Design (LLD) coding rounds.

The biggest mistake in LLD interviews is starting with classes immediately. Instead, start with a real-world story and convert that story into code.

The flow:

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

The memory shortcut:

D → Domain Models
R → Repository
S → Strategy
F → Factory
S → Service
M → Main

---

1. D → Domain Models

Question to ask:

«"What things exist in this real-world system?"»

Look for nouns.

Nouns usually become:

- Data classes
- Enums
- Sealed classes (when different states contain different data)

---

Example: Flight Booking System

Real-world statement:

«"A passenger searches for a flight, selects a seat, makes payment, and receives a booking."»

Important nouns:

Passenger
Flight
Seat
Payment
Booking

These become models.

Example:

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

---

2. R → Repository

Question to ask:

«"Where does my data come from?"»

In real applications:

API
Database
Cache
File Storage

In an interview:

Use an interface + fake implementation.

The goal is to show separation of concerns.

---

Example:

interface FlightRepository {

    fun getFlight(id: String): Flight?
}

Implementation:

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

Repository responsibility:

Fetch data
Save data
Update data
Delete data

Repository should NOT contain business rules.

---

3. S → Strategy Pattern

Question to ask:

«"What behaviour can change in the future?"»

Whenever you see:

- Multiple ways of doing something
- Future extensions
- Different algorithms/rules

Use Strategy.

---

Example: Payment

Today:

UPI Payment

Tomorrow:

Card Payment
Wallet Payment
Apple Pay

Create an interface:

interface PaymentStrategy {

    fun pay(amount: Double)
}

Implementations:

class UpiPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paid using UPI")
    }
}


class CardPayment : PaymentStrategy {

    override fun pay(amount: Double) {
        println("Paid using Card")
    }
}

Now adding a new payment method does not affect existing code.

---

4. F → Factory Pattern

Question to ask:

«"Who is responsible for creating objects?"»

Factory is useful when object creation logic becomes complex.

---

Example:

User selects:

UPI
CARD
WALLET

Factory creates the correct object.

enum class PaymentType {
    UPI,
    CARD
}

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

---

Difference between Strategy and Factory

Strategy

Answers:

«"How should something be done?"»

Example:

Payment calculation
Pricing calculation
Sorting
Notification

---

Factory

Answers:

«"Which object should I create?"»

Example:

Create UPI payment object
Create Card payment object
Create Parser object

---

5. S → Service Layer

Question to ask:

«"What is the actual user action?"»

The service contains the business logic.

This is where interviewers evaluate your design skills.

---

Example:

User wants:

Book Flight

Service handles:

1. Validate flight
2. Check availability
3. Process payment
4. Create booking
5. Update status

Example:

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

---

6. M → Main Function

Question to ask:

«"How can I prove my design works?"»

The main function acts as a small client application.

It should:

- Create objects
- Inject dependencies
- Call service methods
- Print output

---

Example:

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

Output:

Paid using UPI

Booking(
 id=B101,
 status=CONFIRMED
)

---

Complete LLD Interview Flow

When a problem is given:

Step 1: Understand the requirement

Example:

"Design Flight Booking System"

Ask:

- What are the main use cases?
- What operations should be supported?

---

Step 2: Identify Domain Models

Find nouns:

Flight
Passenger
Booking
Payment
Seat

Create:

data class
enum
sealed class

---

Step 3: Create Repository

Ask:

"How will I access data?"

Create:

Interface
Implementation

---

Step 4: Identify Changing Behaviour

Ask:

"What will change frequently?"

Create:

Strategy

Examples:

PaymentStrategy
PricingStrategy
SortingStrategy
SplitStrategy

---

Step 5: Add Factory if required

Ask:

"Is object creation becoming complex?"

Create:

Factory

---

Step 6: Create Service

Put business rules here.

Examples:

BookingService
PaymentService
ExpenseService
SearchService

---

Step 7: Create main()

Demonstrate:

Input
    ↓
Service call
    ↓
Output

---

Examples Where This Framework Applies

Property Listing Service

Domain:

Property
User
Listing
Booking

Strategy:

SearchStrategy
PricingStrategy

Service:

PropertyListingService

---

Flight Aggregation System

Domain:

Flight
Airline
SearchRequest

Strategy:

SortingStrategy
FilterStrategy

Service:

FlightSearchService

---

Expense Management System

Domain:

User
Expense
Group
Split

Strategy:

SplitStrategy

Equal Split
Percentage Split
Exact Split

Service:

ExpenseService

---

Hotel Booking System

Domain:

Hotel
Room
Guest
Reservation

Strategy:

PricingStrategy
PaymentStrategy

Service:

HotelBookingService

---

Important Interview Reminder

Do NOT spend time implementing:

- Real APIs
- Database
- Retrofit
- Networking
- UI
- Authentication

Focus on:

✅ Clean object design
✅ SOLID principles
✅ Separation of responsibility
✅ Extensibility
✅ Business logic

The interviewer is evaluating your design thinking, not production infrastructure.

---

Final Memory Trick

Whenever you see an LLD problem:

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

Master this flow and you can approach most LLD coding interviews confidently.
