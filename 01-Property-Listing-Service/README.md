# 🏠 LLD — Property Listing Service

> **Design and implement a Property Listing Service using Kotlin and the DRSFSM LLD framework.**

A Property Listing Service is similar to features found in applications such as:

- 🏠 Airbnb
- 🏡 Zillow
- 🏘️ Realtor
- 🏢 Booking.com
- 🌍 Real-estate/property marketplaces

The system allows users to **list properties, search properties, filter results, sort results, shortlist properties, and mark properties as sold.**

---

# 🎯 Functional Requirements

The system should support the following operations.

## ✅ Core Requirements

1. **User can list a property**
2. **Search properties**
3. **Search properties by price range**
4. **Search properties by number of rooms**
5. **Sort properties by price**
6. **Sort properties by number of rooms**

## ⭐ Bonus Requirements

7. **Mark property as Sold**
8. **Shortlist a property**

---

# 🧠 Real-Life Understanding

So before coding, think:
```text

             PROPERTY LISTING PLATFORM
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
       🏠 HOST / OWNER      👤 CUSTOMER
              │                 │
              │                 │
        Manage properties    Discover properties
              │                 │
        ┌─────┼─────┐       ┌───┼─────────────┐
        ▼     ▼     ▼       ▼   ▼   ▼   ▼     ▼
       Add  Update Sold   Search Filter Sort View
                                      │
                                      ▼
                                  Shortlist
```
---

# 🔄 System Communication Flow

This is the **rough diagram to remember during the interview**.

```text
                         ┌──────────────┐
                         │    App.kt    │
                         │   main()     │
                         └──────┬───────┘
                                │
                                ▼
                  ┌─────────────────────────┐
                  │ PropertyListingService  │
                  │                         │
                  │  Business Logic ⭐      │
                  └───────┬─────────┬───────┘
                          │         │
                    calls │         │ calls
                          │         │
              ┌───────────┘         └───────────────┐
              ▼                                     ▼
   ┌──────────────────────┐              ┌──────────────────────┐
   │ PropertyRepository   │              │  Filter / Sort       │
   │     Interface        │              │     Strategies       │
   └──────────┬───────────┘              └──────────┬───────────┘
              │                                     │
              │ implemented by                      │
              ▼                                     │
   ┌──────────────────────────┐                     │
   │ InMemoryPropertyRepo     │                     │
   └────────────┬─────────────┘                     │
                │                                   │
                │ stores / retrieves                │
                ▼                                   ▼
        ┌────────────────────────────────────────────────┐
        │                 Domain Models                  │
        │                                                │
        │        User   •   Property   •   Status       │
        └────────────────────────────────────────────────┘
```

### 🧠 Remember

```text
App
 ↓
Service
 ├──→ Repository
 │      ↓
 │   InMemoryRepository
 │
 └──→ Filter / Sort Strategies

Everything works with Domain Models.
```

---

# 🧩 DRSFSM Framework

We will follow the same framework used throughout this LLD repository.

| Step | Component | Responsibility |
|---|---|---|
| 1️⃣ | **Domain** | Define the nouns/entities |
| 2️⃣ | **Repository** | Store and retrieve data |
| 3️⃣ | **Strategy** | Handle changing filter/sort behaviour |
| 4️⃣ | **Factory** | Create objects when object creation becomes complex |
| 5️⃣ | **Service** | Contains business logic |
| 6️⃣ | **Main** | Demonstrates the complete flow |

### For this problem

```text
Domain
   ↓
Repository
   ↓
Strategy
   ↓
Service ⭐
   ↓
Main
```

> **Factory is not necessary here because object creation is simple.**

---

# 1️⃣ DOMAIN — What Things Exist?

Before writing code, ask:

> **"What are the nouns involved in this system?"**

The main nouns are:

```text
User
Property
PropertyStatus
Shortlist
```

---

## 👤 User

A user can list properties and shortlist properties.

```kotlin
data class User(
    val id: String,
    val name: String
)
```

### Example

```text
User
 ├── id = U1
 └── name = Rehan
```

---

# 🏠 Property Status

A property can be available or sold.

```kotlin
enum class PropertyStatus {
    AVAILABLE,
    SOLD
}
```

### Why use an enum?

Instead of:

```kotlin
status = "available"
```

or:

```kotlin
status = "sold"
```

we use a controlled set of values:

```kotlin
PropertyStatus.AVAILABLE
PropertyStatus.SOLD
```

This avoids invalid status values.

---

# 🏠 Property

This represents one property listed by a user.

```kotlin
data class Property(
    val id: String,
    val owner: User,
    val title: String,
    val city: String,
    val price: Double,
    val rooms: Int,
    var status: PropertyStatus = PropertyStatus.AVAILABLE
)
```

### Why these fields?

```text
id
→ uniquely identifies the property

owner
→ user who listed the property

title
→ property title

city
→ property location

price
→ property price

rooms
→ number of rooms

status
→ AVAILABLE or SOLD
```

---

# ⭐ Shortlist

A user should be able to save a property for later.

For a simple interview implementation, we can maintain:

```kotlin
data class Shortlist(
    val userId: String,
    val propertyIds: MutableSet<String> = mutableSetOf()
)
```
Think of Shortlist = Bookmark / Save for later.

```text
User searches properties
        ↓
Likes "Cozy Studio"
        ↓
⭐ Shortlist / Bookmark
        ↓
Property saved to user's shortlist
        ↓
User can quickly view it later
```
---

# 2️⃣ REPOSITORY — Where Is The Data?

Ask:

> **"Where does the property data come from?"**

In a real application:

```text
Database
API
Cache
```

But in a CoderPad interview, we can keep it simple:

```text
In-Memory List
```

---

# 📦 PropertyRepository

The Repository interface defines what the Service can do with property data.

```kotlin
interface PropertyRepository {

    // Save a newly listed property.
    fun addProperty(property: Property)

    // Find a property using its unique ID.
    fun getProperty(id: String): Property?

    // Return all properties.
    fun getAllProperties(): List<Property>

    // Update an existing property.
    fun updateProperty(property: Property)
}
```

### Repository responsibility

```text
Repository = DATA ACCESS

✅ Save
✅ Get
✅ Update

❌ Filter business rules
❌ Sorting business rules
❌ Shortlisting logic
❌ Business decisions
```

---

# 🗄️ InMemoryPropertyRepository

For the interview, our fake database is simply a mutable list.

```kotlin
class InMemoryPropertyRepository : PropertyRepository {

    // Acts as our temporary in-memory database.
    private val properties = mutableListOf<Property>()

    override fun addProperty(property: Property) {
        properties.add(property)
    }

    override fun getProperty(id: String): Property? {
        return properties.find {
            it.id == id
        }
    }

    override fun getAllProperties(): List<Property> {
        // Return a copy so callers don't directly
        // modify our internal list.
        return properties.toList()
    }

    override fun updateProperty(property: Property) {
        val index = properties.indexOfFirst {
            it.id == property.id
        }

        if (index != -1) {
            properties[index] = property
        }
    }
}
```

---

# 3️⃣ STRATEGY — What Behaviour Can Change?

This is where the problem becomes interesting.

The user can search properties in different ways:

```text
FILTER
 ├── Price Range
 └── Number of Rooms

SORT
 ├── Price
 └── Number of Rooms
```

Instead of writing every rule inside the Service, we can separate the changing behaviour.

---

# 🔍 Filter Strategy

```kotlin
interface PropertyFilterStrategy {

    // Receives the current properties
    // and returns only matching properties.
    fun filter(
        properties: List<Property>
    ): List<Property>
}
```

---

# 💰 Price Range Filter

Example:

```text
Minimum Price = ₹20 Lakh
Maximum Price = ₹50 Lakh
```

Only properties inside that range should be returned.

```kotlin
class PriceRangeFilterStrategy(
    private val minPrice: Double,
    private val maxPrice: Double
) : PropertyFilterStrategy {

    override fun filter(
        properties: List<Property>
    ): List<Property> {

        return properties.filter { property ->

            property.price in minPrice..maxPrice &&
                property.status == PropertyStatus.AVAILABLE
        }
    }
}
```

### Real-life example

```text
Properties:

₹15L
₹25L  ← included
₹35L  ← included
₹60L

Range = ₹20L - ₹50L

Result:

₹25L
₹35L
```

---

# 🚪 Number Of Rooms Filter

```kotlin
class RoomFilterStrategy(
    private val requiredRooms: Int
) : PropertyFilterStrategy {

    override fun filter(
        properties: List<Property>
    ): List<Property> {

        return properties.filter { property ->

            property.rooms == requiredRooms &&
                property.status == PropertyStatus.AVAILABLE
        }
    }
}
```

### Example

```text
Required rooms = 2

1 room ❌
2 rooms ✅
3 rooms ❌
4 rooms ❌
```

---

# ↕️ Sort Strategy

Filtering and sorting are different behaviours.

Therefore, we create a separate strategy.

```kotlin
interface PropertySortStrategy {

    // Receives properties and returns
    // them in the requested order.
    fun sort(
        properties: List<Property>
    ): List<Property>
}
```

---

# 💵 Sort By Price

```kotlin
class PriceSortStrategy : PropertySortStrategy {

    override fun sort(
        properties: List<Property>
    ): List<Property> {

        // Cheapest property comes first.
        return properties.sortedBy {
            it.price
        }
    }
}
```

---

# 🚪 Sort By Rooms

```kotlin
class RoomSortStrategy : PropertySortStrategy {

    override fun sort(
        properties: List<Property>
    ): List<Property> {

        // Property with fewer rooms comes first.
        return properties.sortedBy {
            it.rooms
        }
    }
}
```

---

# 4️⃣ FACTORY

For this problem, a Factory is **optional**.

We don't need one because creating a Property is straightforward:

```kotlin
Property(
    id = ...,
    owner = ...,
    title = ...,
    city = ...,
    price = ...,
    rooms = ...
)
```

### Interview explanation

> "I don't see enough object-creation complexity here to justify a Factory, so I'll keep creation inside the Service. If property creation later becomes dependent on property type or multiple construction rules, I can introduce a Factory."

This is better than forcing a Factory just because the framework contains one.

---

# 5️⃣ SERVICE CLASS ⭐

This is the **most important class**.

Ask:

> **"What actions can the user perform?"**

```text
1. List Property
2. Search Properties
3. Filter By Price
4. Filter By Rooms
5. Sort By Price
6. Sort By Rooms
7. Mark Property As Sold
8. Shortlist Property
```

All business decisions belong here.

---

# 🏠 PropertyListingService

```kotlin
class PropertyListingService(
    private val propertyRepository: PropertyRepository
) {

    // Stores shortlisted property IDs per user.
    // Example:
    //
    // U1 → [P1, P2]
    // U2 → [P3]
    //
    private val shortlists =
        mutableMapOf<String, MutableSet<String>>()


    // =================================================
    // 1️⃣ LIST PROPERTY
    // =================================================

    fun listProperty(
        owner: User,
        title: String,
        city: String,
        price: Double,
        rooms: Int
    ): Property {

        // Create a new property.
        val property = Property(
            id = UUID.randomUUID().toString(),
            owner = owner,
            title = title,
            city = city,
            price = price,
            rooms = rooms
        )

        // Store the property through the repository.
        propertyRepository.addProperty(property)

        return property
    }


    // =================================================
    // 2️⃣ SEARCH PROPERTIES
    // =================================================

    fun searchProperties(
        city: String? = null,
        filterStrategy: PropertyFilterStrategy? = null,
        sortStrategy: PropertySortStrategy? = null
    ): List<Property> {

        // Start with all properties.
        var properties =
            propertyRepository.getAllProperties()

        // Sold properties should not appear
        // in normal search results.
        properties = properties.filter {
            it.status == PropertyStatus.AVAILABLE
        }

        // Apply city filter if provided.
        city?.let { requestedCity ->

            properties = properties.filter {
                it.city.equals(
                    requestedCity,
                    ignoreCase = true
                )
            }
        }

        // Apply additional filter strategy.
        filterStrategy?.let {
            properties = it.filter(properties)
        }

        // Apply sorting strategy.
        sortStrategy?.let {
            properties = it.sort(properties)
        }

        return properties
    }


    // =================================================
    // 3️⃣ MARK PROPERTY AS SOLD
    // =================================================

    fun markPropertyAsSold(
        propertyId: String
    ) {

        val property =
            propertyRepository.getProperty(propertyId)
                ?: throw IllegalArgumentException(
                    "Property not found"
                )

        // Change the property state.
        property.status = PropertyStatus.SOLD

        // Save the updated property.
        propertyRepository.updateProperty(property)
    }


    // =================================================
    // 4️⃣ SHORTLIST PROPERTY
    // =================================================

    fun shortlistProperty(
        userId: String,
        propertyId: String
    ) {

        // Make sure property exists.
        propertyRepository.getProperty(propertyId)
            ?: throw IllegalArgumentException(
                "Property not found"
            )

        // Create shortlist for the user
        // if it doesn't already exist.
        val shortlist =
            shortlists.getOrPut(userId) {
                mutableSetOf()
            }

        // Set automatically prevents duplicates.
        shortlist.add(propertyId)
    }


    // =================================================
    // 5️⃣ GET USER SHORTLIST
    // =================================================

    fun getShortlistedProperties(
        userId: String
    ): List<Property> {

        val propertyIds =
            shortlists[userId].orEmpty()

        return propertyIds.mapNotNull { propertyId ->

            propertyRepository.getProperty(propertyId)
        }
    }
}
```

---

# 🧠 Service Business Logic

You don't need to memorize every line.

Remember the **actions**.

## LIST PROPERTY

```text
User
 ↓
Service
 ↓
Create Property
 ↓
Repository.addProperty()
```

---

## SEARCH

```text
Service
 ↓
Repository.getAllProperties()
 ↓
Remove SOLD properties
 ↓
City filter
 ↓
Price / Room filter
 ↓
Price / Room sort
 ↓
Return results
```

---

## MARK SOLD

```text
Service
 ↓
Find Property
 ↓
Change status
 ↓
AVAILABLE → SOLD
 ↓
Repository.updateProperty()
```

---

## SHORTLIST

```text
Service
 ↓
Find Property
 ↓
Find user's shortlist
 ↓
Add property ID
 ↓
Set prevents duplicates
```

---

# 🔥 BUSINESS ACTION MEMORY

```text
┌───────────────────────────────────────┐
│       PROPERTY LISTING SERVICE        │
├───────────────────────────────────────┤
│                                       │
│  🏠 LIST PROPERTY                     │
│           ↓                           │
│  🔎 SEARCH PROPERTIES                 │
│           ↓                           │
│  💰 FILTER BY PRICE                   │
│           ↓                           │
│  🚪 FILTER BY ROOMS                   │
│           ↓                           │
│  💵 SORT BY PRICE                     │
│           ↓                           │
│  🚪 SORT BY ROOMS                     │
│           ↓                           │
│  ⭐ SHORTLIST PROPERTY                │
│           ↓                           │
│  🔴 MARK PROPERTY AS SOLD             │
│                                       │
└───────────────────────────────────────┘
```

### 🧠 One-Line Memory Trick

> **LIST → SEARCH → FILTER → SORT → SHORTLIST → SOLD**

---

# 6️⃣ MAIN FUNCTION

`main()` acts as our small client application.

It demonstrates how all components communicate.

```kotlin
fun main() {

    // =================================================
    // 1️⃣ CREATE REPOSITORY
    // =================================================

    val repository =
        InMemoryPropertyRepository()


    // =================================================
    // 2️⃣ CREATE SERVICE
    // =================================================

    // Repository is injected into the Service.
    val service =
        PropertyListingService(repository)


    // =================================================
    // 3️⃣ CREATE USERS
    // =================================================

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


    // =================================================
    // 4️⃣ LIST PROPERTIES
    // =================================================

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

    service.listProperty(
        owner = ali,
        title = "Chennai Home",
        city = "Chennai",
        price = 4500000.0,
        rooms = 3
    )


    // =================================================
    // 5️⃣ SEARCH PROPERTIES
    // =================================================

    println("🏠 Properties in Bengaluru")

    service.searchProperties(
        city = "Bengaluru"
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "₹${property.price} | " +
            "${property.rooms} rooms"
        )
    }


    // =================================================
    // 6️⃣ FILTER BY PRICE RANGE
    // =================================================

    println(
        "\n💰 Properties between ₹20L and ₹50L"
    )

    service.searchProperties(
        city = "Bengaluru",

        filterStrategy =
            PriceRangeFilterStrategy(
                minPrice = 2_000_000.0,
                maxPrice = 5_000_000.0
            )
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "₹${property.price}"
        )
    }


    // =================================================
    // 7️⃣ FILTER BY NUMBER OF ROOMS
    // =================================================

    println(
        "\n🚪 Properties with 2 rooms"
    )

    service.searchProperties(
        city = "Bengaluru",

        filterStrategy =
            RoomFilterStrategy(
                requiredRooms = 2
            )
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "${property.rooms} rooms"
        )
    }


    // =================================================
    // 8️⃣ SORT BY PRICE
    // =================================================

    println(
        "\n💵 Bengaluru properties sorted by price"
    )

    service.searchProperties(
        city = "Bengaluru",

        sortStrategy =
            PriceSortStrategy()
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "₹${property.price}"
        )
    }


    // =================================================
    // 9️⃣ SORT BY ROOMS
    // =================================================

    println(
        "\n🚪 Bengaluru properties sorted by rooms"
    )

    service.searchProperties(
        city = "Bengaluru",

        sortStrategy =
            RoomSortStrategy()
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "${property.rooms} rooms"
        )
    }


    // =================================================
    // 🔟 SHORTLIST PROPERTY
    // =================================================

    println(
        "\n⭐ Shortlisting property"
    )

    service.shortlistProperty(
        userId = ali.id,
        propertyId = studio.id
    )

    val shortlisted =
        service.getShortlistedProperties(
            userId = ali.id
        )

    println("Ali's shortlist:")

    shortlisted.forEach { property ->

        println(
            "  • ${property.title}"
        )
    }


    // =================================================
    // 1️⃣1️⃣ MARK PROPERTY AS SOLD
    // =================================================

    println(
        "\n🔴 Marking property as SOLD"
    )

    service.markPropertyAsSold(
        propertyId = apartment.id
    )

    println(
        "  • ${apartment.title} → SOLD"
    )


    // =================================================
    // 1️⃣2️⃣ SEARCH AGAIN
    // =================================================

    println(
        "\n🔎 Available Bengaluru properties after sale"
    )

    service.searchProperties(
        city = "Bengaluru"
    ).forEach { property ->

        println(
            "  • ${property.title} | " +
            "₹${property.price} | " +
            "${property.rooms} rooms"
        )
    }
}
```

---

# 🖥️ Sample Output

```text
🏠 Properties in Bengaluru

  • Cozy Studio | ₹1500000.0 | 1 rooms
  • Modern Apartment | ₹3500000.0 | 2 rooms
  • Luxury Villa | ₹8000000.0 | 4 rooms


💰 Properties between ₹20L and ₹50L

  • Modern Apartment | ₹3500000.0


🚪 Properties with 2 rooms

  • Modern Apartment | 2 rooms


💵 Bengaluru properties sorted by price

  • Cozy Studio | ₹1500000.0
  • Modern Apartment | ₹3500000.0
  • Luxury Villa | ₹8000000.0


🚪 Bengaluru properties sorted by rooms

  • Cozy Studio | 1 rooms
  • Modern Apartment | 2 rooms
  • Luxury Villa | 4 rooms


⭐ Shortlisting property

Ali's shortlist:

  • Cozy Studio


🔴 Marking property as SOLD

  • Modern Apartment → SOLD


🔎 Available Bengaluru properties after sale

  • Cozy Studio | ₹1500000.0 | 1 rooms
  • Luxury Villa | ₹8000000.0 | 4 rooms
```

---

# 🔗 Complete Communication Flow

```text
                         ┌──────────────┐
                         │    App.kt    │
                         │   main()     │
                         └──────┬───────┘
                                │
                                ▼
                 ┌──────────────────────────┐
                 │ PropertyListingService   │
                 │                          │
                 │  listProperty()          │
                 │  searchProperties()      │
                 │  markPropertyAsSold()    │
                 │  shortlistProperty()     │
                 └───────┬───────────┬──────┘
                         │           │
                         │           │
                         ▼           ▼
             ┌────────────────┐   ┌────────────────────┐
             │  Repository    │   │    Strategies      │
             │   Interface    │   │                    │
             └───────┬────────┘   │ Filter              │
                     │            │  ├─ Price            │
                     │            │  └─ Rooms            │
                     ▼            │                    │
          ┌────────────────────┐  │ Sort               │
          │ InMemoryRepository │  │  ├─ Price           │
          └──────────┬─────────┘  │  └─ Rooms           │
                     │            └────────────────────┘
                     │
                     ▼
          ┌────────────────────────┐
          │     Domain Models      │
          │                        │
          │ User                   │
          │ Property               │
          │ PropertyStatus         │
          │ Shortlist              │
          └────────────────────────┘
```

---

# 🎤 How To Explain This In The Interview

If the interviewer asks:

> **"Design a Property Listing Service."**

Don't immediately start coding.

Start with:

### 1️⃣ Clarify Requirements

```text
"Let me first clarify the functional requirements."

• Can users list properties?
• What fields are required?
• Can users search by price?
• Can users search by number of rooms?
• What sorting options do we need?
• Should sold properties appear in search?
• Can users shortlist properties?
```

---

### 2️⃣ Identify Domain Models

Say:

> "The main nouns I see are User, Property, PropertyStatus and Shortlist."

---

### 3️⃣ Draw Communication Flow

```text
App
 ↓
Service
 ├── Repository
 └── Filter / Sort Strategies
```

---

### 4️⃣ Start DRSFSM

```text
Domain
   ↓
Repository
   ↓
Strategy
   ↓
Factory (only if needed)
   ↓
Service
   ↓
main()
```

---

### 5️⃣ Prioritize Core Features

If the interviewer gives you **one hour**, don't panic when you see many requirements.

Implement in this order:

```text
                    60 MINUTES
                        │
                        ▼
              ┌──────────────────┐
              │  Core First ⭐    │
              └────────┬─────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       LIST         SEARCH       FILTER
                                   │
                              ┌────┴────┐
                              ▼         ▼
                           PRICE      ROOMS
                              │
                              └────┬────┘
                                   ▼
                                  SORT
                              ┌────┴────┐
                              ▼         ▼
                           PRICE      ROOMS

                    Then if time remains:

                         ⭐ SHORTLIST
                         ⭐ SOLD
```

### 🧠 Senior-level statement

> **"I'll first make the core listing and search flow work, including filtering and sorting. Once that is complete and tested, I'll use the remaining time to implement the bonus features."**

This shows **prioritization**, not just coding.

---

# ⚡ Interview Memory Card

Before coding, remember only this:

```text
┌─────────────────────────────────────────┐
│       🏠 PROPERTY LISTING SERVICE       │
├─────────────────────────────────────────┤
│                                         │
│ DOMAIN                                  │
│ → User                                  │
│ → Property                              │
│ → Status                                │
│ → Shortlist                             │
│                                         │
│ REPOSITORY                              │
│ → Add                                   │
│ → Get                                   │
│ → Update                                │
│                                         │
│ STRATEGY                                │
│ → Filter Price                          │
│ → Filter Rooms                          │
│ → Sort Price                            │
│ → Sort Rooms                            │
│                                         │
│ SERVICE ⭐                              │
│ → List                                  │
│ → Search                                │
│ → Mark Sold                             │
│ → Shortlist                             │
│                                         │
│ MAIN                                    │
│ → Demonstrate everything                │
│                                         │
└─────────────────────────────────────────┘
```

---

# 🔥 Final Memory Trick

Don't memorize the code.

Memorize the **story**:

```text
                 PROPERTY

                    ↓

                  LIST
                    ↓
                 SEARCH
                    ↓
               ┌────┴────┐
               ↓         ↓
             FILTER     SORT
               │         │
          ┌────┴───┐ ┌───┴────┐
          ↓        ↓ ↓        ↓
        PRICE    ROOMS PRICE  ROOMS
                    │
                    ↓
               ┌────┴────┐
               ↓         ↓
           SHORTLIST    SOLD
```

### 🧠 One sentence

> **"A user lists a property, other users search it, filter it by price or rooms, sort the results, shortlist it, and eventually the property can be marked as sold."**

If you can explain that story, identify the nouns, draw the flow, and implement the business actions using your DRSFSM framework, **you understand the LLD.**

---

# ✅ Interview Checklist

Before moving to the next LLD, make sure you can do this **without looking at the README**:

- [ ] Explain functional requirements
- [ ] Identify domain models
- [ ] Draw communication flow
- [ ] Create Repository
- [ ] Create Filter Strategy
- [ ] Create Sort Strategy
- [ ] Explain why Factory is not required
- [ ] Implement `listProperty()`
- [ ] Implement `searchProperties()`
- [ ] Filter by price
- [ ] Filter by rooms
- [ ] Sort by price
- [ ] Sort by rooms
- [ ] Implement `markPropertyAsSold()`
- [ ] Implement `shortlistProperty()`
- [ ] Run `main()`
- [ ] Explain code while writing
- [ ] Complete the core flow within the interview time

---

# 🎯 CORE BUSINESS ACTIONS

```text
LIST
  ↓
SEARCH
  ↓
FILTER
  ├── Price
  └── Rooms
  ↓
SORT
  ├── Price
  └── Rooms
  ↓
SHORTLIST ⭐
  ↓
SOLD ⭐
```

> **Remember: Don't memorize the implementation. Remember the business actions.**

> **LIST → SEARCH → FILTER → SORT → SHORTLIST → SOLD**
