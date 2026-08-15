# 🔄 SYSTEM COMMUNICATION FLOW

```text
                         ┌──────────────┐
                         │    App.kt    │
                         │   main()     │
                         └──────┬───────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │  PropertyListing    │
                    │      Service        │
                    └──────┬────────┬─────┘
                           │        │
                    calls  │        │  calls
                           │        │
              ┌────────────┘        └──────────────┐
              ▼                                    ▼
    ┌────────────────────┐              ┌────────────────────┐
    │ PropertyRepository │              │ PropertyStrategy   │
    │    (interface)     │              │    (interface)     │
    └─────────┬──────────┘              └──────────┬─────────┘
              │                                    │
              │ implemented by                    │ implemented by
              ▼                                    ▼
    ┌────────────────────────┐       ┌─────────────┬──────────────┐
    │ InMemoryPropertyRepo   │       │   Filter    │    Sort      │
    └───────────┬────────────┘       │ Strategies  │  Strategies  │
                │                    └─────────────┴──────────────┘
                │
                │ stores / retrieves
                ▼
        ┌──────────────────┐
        │  Domain Models   │
        │                  │
        │  User            │
        │  Property        │
        │  Status          │
        └──────────────────┘


---

🎯 REQUIREMENTS

The system should support:

Core Requirements

User can list a property

Search properties

Search by price range

Search by number of rooms

Sort by number of rooms

Sort by price


Bonus Requirements

Mark property as Sold

Shortlist property



---

🧠 REAL-WORLD STORY

Imagine a user wants to sell/rent a property.

The user can:

List Property
     ↓
Search Properties
     ↓
Filter by Price / Rooms
     ↓
Sort Results
     ↓
Shortlist Property
     ↓
Mark Property as Sold

The important thing is that most operations work on the same collection of Property objects.


---

🧩 FRAMEWORK USED

Step	Component	Purpose

1	Domain Models	The "nouns" of the system
2	Repository	Stores and retrieves properties
3	Strategy	Handles changing filter/sort behaviour
4	Service	Contains business logic
5	main()	Demonstrates the complete flow



---

1️⃣ DOMAIN MODELS

Ask:

> "What things exist in this system?"



Important nouns:

User
Property
Shortlist
PropertyStatus


---

Property Status

A property can have different states.

enum class PropertyStatus {
    AVAILABLE,
    SOLD
}


---

User

A user can list properties and shortlist properties.

data class User(
    val id: String,
    val name: String
)


---

Property

Represents one property listed by a user.

data class Property(
    val id: String,
    val owner: User,
    val title: String,
    val city: String,
    val price: Double,
    val rooms: Int,
    var status: PropertyStatus = PropertyStatus.AVAILABLE
)

Why these fields?

id
→ uniquely identifies the property

owner
→ who listed the property

title
→ property name/title

city
→ where the property is located

price
→ property price

rooms
→ number of rooms

status
→ AVAILABLE or SOLD


---

2️⃣ REPOSITORY

Ask:

> "Where does my property data come from?"



For CoderPad, we don't need a real database.

We'll use an in-memory list.

interface PropertyRepository {

    // Store a newly listed property.
    fun addProperty(property: Property)

    // Find one property using its ID.
    fun getProperty(id: String): Property?

    // Return all properties.
    fun getAllProperties(): List<Property>
}


---

🗄️ IN-MEMORY REPOSITORY

class InMemoryPropertyRepository : PropertyRepository {

    // Fake database for our interview.
    private val properties = mutableListOf<Property>()

    override fun addProperty(property: Property) {
        properties.add(property)
    }

    override fun getProperty(id: String): Property? {
        return properties.find { it.id == id }
    }

    override fun getAllProperties(): List<Property> {
        return properties.toList()
    }
}

Important

The Repository only handles:

STORE
GET

It should NOT contain business rules such as:

Filter by price
Sort by rooms
Mark as sold

Those belong in the Service.


---

3️⃣ STRATEGY

Ask:

> "What behaviour can change?"



Here we have different ways to process search results.

Filter by price
Filter by rooms

Sort by price
Sort by rooms

Instead of putting every rule directly inside the Service, we can use Strategy.


---

Strategy Interface

interface PropertyStrategy {

    // Takes the current property list
    // and returns the processed result.
    fun apply(properties: List<Property>): List<Property>
}


---

💰 PRICE RANGE FILTER

class PriceRangeFilterStrategy(
    private val minPrice: Double,
    private val maxPrice: Double
) : PropertyStrategy {

    override fun apply(
        properties: List<Property>
    ): List<Property> {

        // Keep only properties whose price
        // falls inside the requested range.
        return properties.filter {
            it.price in minPrice..maxPrice &&
            it.status == PropertyStatus.AVAILABLE
        }
    }
}

Example:

₹1,000,000 → ₹5,000,000

Only properties inside this range are returned.


---

🚪 ROOM FILTER

class RoomFilterStrategy(
    private val rooms: Int
) : PropertyStrategy {

    override fun apply(
        properties: List<Property>
    ): List<Property> {

        // Return properties having the
        // requested number of rooms.
        return properties.filter {
            it.rooms == rooms &&
            it.status == PropertyStatus.AVAILABLE
        }
    }
}


---

💵 SORT BY PRICE

class PriceSortStrategy : PropertyStrategy {

    override fun apply(
        properties: List<Property>
    ): List<Property> {

        // Sort properties from cheapest to most expensive.
        return properties.sortedBy {
            it.price
        }
    }
}


---

🚪 SORT BY ROOMS

class RoomSortStrategy : PropertyStrategy {

    override fun apply(
        properties: List<Property>
    ): List<Property> {

        // Sort properties from fewer rooms
        // to more rooms.
        return properties.sortedBy {
            it.rooms
        }
    }
}


---

4️⃣ SERVICE CLASS ⭐

This is the most important class.

Ask:

> "What actions can the user perform?"



Our Service needs to handle:

1. List Property
2. Search Properties
3. Filter by Price
4. Filter by Rooms
5. Sort by Price
6. Sort by Rooms
7. Mark Property as Sold
8. Shortlist Property


---

PropertyListingService

class PropertyListingService(
    private val repository: PropertyRepository
) {

    // ------------------------------------------------
    // 1. LIST PROPERTY
    // ------------------------------------------------
    // A user creates a new property listing.
    fun listProperty(
        owner: User,
        title: String,
        city: String,
        price: Double,
        rooms: Int
    ): Property {

        val property = Property(
            id = UUID.randomUUID().toString(),
            owner = owner,
            title = title,
            city = city,
            price = price,
            rooms = rooms
        )

        // Save the newly created property.
        repository.addProperty(property)

        return property
    }


    // ------------------------------------------------
    // 2. SEARCH PROPERTIES
    // ------------------------------------------------
    // Returns available properties.
    fun searchProperties(
        city: String? = null,
        strategy: PropertyStrategy? = null
    ): List<Property> {

        // Start with all properties.
        var properties = repository.getAllProperties()

        // Only available properties should appear
        // in normal search results.
        properties = properties.filter {
            it.status == PropertyStatus.AVAILABLE
        }

        // Optional city filter.
        city?.let { requestedCity ->

            properties = properties.filter {
                it.city.equals(
                    requestedCity,
                    ignoreCase = true
                )
            }
        }

        // Apply the requested filter/sort strategy.
        return strategy?.apply(properties)
            ?: properties
    }


    // ------------------------------------------------
    // 3. MARK PROPERTY AS SOLD
    // ------------------------------------------------
    // Once sold, the property should no longer
    // appear in normal search results.
    fun markAsSold(propertyId: String) {

        val property =
            repository.getProperty(propertyId)
                ?: throw NoSuchElementException(
                    "Property not found"
                )

        property.status = PropertyStatus.SOLD
    }


    // ------------------------------------------------
    // 4. SHORTLIST PROPERTY
    // ------------------------------------------------
    // A user can save a property for later.
    //
    // For this interview implementation,
    // we simply demonstrate the action.
    fun shortlistProperty(
        userId: String,
        propertyId: String
    ) {

        val property =
            repository.getProperty(propertyId)
                ?: throw NoSuchElementException(
                    "Property not found"
                )

        println(
            "User $userId shortlisted: ${property.title}"
        )
    }
}


---

🧠 SERVICE BUSINESS ACTIONS

Don't memorize the entire Service code.

Remember the actions:

LIST PROPERTY
      ↓
Create Property
      ↓
Repository.addProperty()


SEARCH
      ↓
Get All Properties
      ↓
Filter Available
      ↓
Optional City Filter
      ↓
Apply Strategy
      ↓
Return Results


MARK SOLD
      ↓
Find Property
      ↓
Change Status → SOLD


SHORTLIST
      ↓
Find Property
      ↓
Add to user's shortlist

🔥 Memory Shortcut

> LIST → SEARCH → FILTER → SORT → SOLD → SHORTLIST




---

5️⃣ MAIN FUNCTION

main() demonstrates how all components communicate.

fun main() {

    // ---------------------------------------------
    // Create Repository
    // ---------------------------------------------

    val repository =
        InMemoryPropertyRepository()


    // ---------------------------------------------
    // Inject Repository into Service
    // ---------------------------------------------

    val service =
        PropertyListingService(repository)


    // ---------------------------------------------
    // Create Users
    // ---------------------------------------------

    val rehan =
        User(
            id = "U1",
            name = "Rehan"
        )

    val ali =
        User(
            id = "U2",
            name = "Ali"
        )


    // ---------------------------------------------
    // LIST PROPERTIES
    // ---------------------------------------------

    val studio =
        service.listProperty(
            owner = rehan,
            title = "Cozy Studio",
            city = "Bengaluru",
            price = 1500000.0,
            rooms = 1
        )

    val apartment =
        service.listProperty(
            owner = ali,
            title = "Modern Apartment",
            city = "Bengaluru",
            price = 3500000.0,
            rooms = 2
        )

    val villa =
        service.listProperty(
            owner = rehan,
            title = "Luxury Villa",
            city = "Bengaluru",
            price = 8000000.0,
            rooms = 4
        )

    val chennaiHome =
        service.listProperty(
            owner = ali,
            title = "Chennai Home",
            city = "Chennai",
            price = 4500000.0,
            rooms = 3
        )


    // ---------------------------------------------
    // SEARCH ALL BENGALURU PROPERTIES
    // ---------------------------------------------

    println("🏠 Properties in Bengaluru:")

    service.searchProperties(
        city = "Bengaluru"
    ).forEach {

        println(
            " - ${it.title} | ₹${it.price} | ${it.rooms} rooms"
        )
    }


    // ---------------------------------------------
    // SEARCH BY PRICE RANGE
    // ---------------------------------------------

    println(
        "\n💰 Properties between ₹2M and ₹5M:"
    )

    service.searchProperties(
        city = "Bengaluru",
        strategy = PriceRangeFilterStrategy(
            minPrice = 2000000.0,
            maxPrice = 5000000.0
        )
    ).forEach {

        println(
            " - ${it.title} | ₹${it.price}"
        )
    }


    // ---------------------------------------------
    // SEARCH BY NUMBER OF ROOMS
    // ---------------------------------------------

    println(
        "\n🚪 Properties with 2 rooms:"
    )

    service.searchProperties(
        city = "Bengaluru",
        strategy = RoomFilterStrategy(
            rooms = 2
        )
    ).forEach {

        println(
            " - ${it.title} | ${it.rooms} rooms"
        )
    }


    // ---------------------------------------------
    // SORT BY PRICE
    // ---------------------------------------------

    println(
        "\n💵 Bengaluru properties sorted by price:"
    )

    service.searchProperties(
        city = "Bengaluru",
        strategy = PriceSortStrategy()
    ).forEach {

        println(
            " - ${it.title} | ₹${it.price}"
        )
    }


    // ---------------------------------------------
    // SORT BY NUMBER OF ROOMS
    // ---------------------------------------------

    println(
        "\n🚪 Bengaluru properties sorted by rooms:"
    )

    service.searchProperties(
        city = "Bengaluru",
        strategy = RoomSortStrategy()
    ).forEach {

        println(
            " - ${it.title} | ${it.rooms} rooms"
        )
    }


    // ---------------------------------------------
    // SHORTLIST PROPERTY
    // ---------------------------------------------

    println("\n⭐ Shortlist:")

    service.shortlistProperty(
        userId = "U2",
        propertyId = studio.id
    )


    // ---------------------------------------------
    // MARK PROPERTY AS SOLD
    // ---------------------------------------------

    println("\n🔴 Marking property as SOLD:")

    service.markAsSold(
        propertyId = apartment.id
    )

    println(
        " - ${apartment.title} is now SOLD"
    )


    // ---------------------------------------------
    // SEARCH AGAIN
    // ---------------------------------------------
    // Sold properties should no longer appear
    // in normal search results.

    println(
        "\n🔎 Available Bengaluru properties after sale:"
    )

    service.searchProperties(
        city = "Bengaluru"
    ).forEach {

        println(
            " - ${it.title} | ₹${it.price}"
        )
    }
}


---

🖥️ SAMPLE OUTPUT

🏠 Properties in Bengaluru:
 - Cozy Studio | ₹1500000.0 | 1 rooms
 - Modern Apartment | ₹3500000.0 | 2 rooms
 - Luxury Villa | ₹8000000.0 | 4 rooms

💰 Properties between ₹2M and ₹5M:
 - Modern Apartment | ₹3500000.0

🚪 Properties with 2 rooms:
 - Modern Apartment | 2 rooms

💵 Bengaluru properties sorted by price:
 - Cozy Studio | ₹1500000.0
 - Modern Apartment | ₹3500000.0
 - Luxury Villa | ₹8000000.0

🚪 Bengaluru properties sorted by rooms:
 - Cozy Studio | 1 rooms
 - Modern Apartment | 2 rooms
 - Luxury Villa | 4 rooms

⭐ Shortlist:
User U2 shortlisted: Cozy Studio

🔴 Marking property as SOLD:
 - Modern Apartment is now SOLD

🔎 Available Bengaluru properties after sale:
 - Cozy Studio | ₹1500000.0
 - Luxury Villa | ₹8000000.0


---

🎯 COMPLETE LLD FLOW

App.kt
                           │
                           ▼
                 PropertyListingService
                    │              │
                    │              │
                    ▼              ▼
           PropertyRepository   PropertyStrategy
                    │              │
                    ▼               ┌─────┴───────────────┐
          InMemoryRepository  │                     │
                              ▼                     ▼
                       Filter Strategies      Sort Strategies
                              │                     │
                       ┌──────┴──────┐       ┌──────┴──────┐
                       ▼             ▼       ▼             ▼
                    Price          Rooms   Price         Rooms
                    Filter         Filter  Sort          Sort
                             
                    All components work
                    with Domain Models
                           │
                           ▼
                  User / Property / Status


---
