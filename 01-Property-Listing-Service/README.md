# 🏠 LLD TOPIC: PROPERTY LISTING SERVICE

### (like Airbnb, Vrbo, Booking.com, Agoda, Expedia, MakeMyTrip, OYO, Zillow, Realtor.com, Redfin)

---

## 🧩 FRAMEWORK USED

Same 5 steps every time:

| Step | Component | Purpose |
|---|---|---|
| 1 | Enums + Data classes | The "nouns" of this domain |
| 2 | Repository | Where data is stored/fetched |
| 3 | Strategy interface | The one thing that "varies" |
| 4 | Service class | The actual business logic |
| 5 | `main()` | Proves everything works |

---

## 1️⃣ ENUMS + DATA CLASSES

> Enums = fixed set of options. Data classes = simple objects that just hold data.

```kotlin
/**
 * Represents the different types of properties
 * that can be listed on the platform.
 */
enum class PropertyType {
    APARTMENT, // Property inside a residential building
    VILLA,     // Independent/larger house
    STUDIO     // Compact single-room living space
}

/**
 * Represents the person who owns and lists
 * the property on the platform.
 */
data class Host(
    val id: String,
    val name: String
)

/**
 * Represents the actual property being listed.
 *
 * A property belongs to a Host and contains
 * the information guests need while searching.
 */
data class Property(
    val id: String,
    val host: Host,
    val title: String,
    val city: String,
    val pricePerNight: Double,
    val type: PropertyType,

    // true  → property can still be booked
    // false → property is no longer available
    var isAvailable: Boolean = true
)
````

---

## 2️⃣ REPOSITORY

> Repository's ONLY job = store data and let us search/fetch it. It does NOT contain business logic (that goes in Service, step 4).

```kotlin
interface PropertyRepository {

    // Store a newly listed property.
    fun addProperty(property: Property)

    // Find one specific property using its unique ID.
    // Returns null if the property does not exist.
    fun getById(id: String): Property?

    // Search all available properties in a particular city.
    fun searchByCity(city: String): List<Property>
}
```

### In-Memory Repository

> "InMemory" means we are just using a list instead of a real database. This is what you'll do in CoderPad since there's no real DB.

```kotlin
/**
 * In-memory implementation of PropertyRepository.
 *
 * In a real application, property data would normally
 * come from a database or backend service.
 *
 * For this LLD, we keep the data inside a MutableList.
 */
class InMemoryPropertyRepository : PropertyRepository {

    // Stores all properties that have been listed.
    private val properties = mutableListOf<Property>()

    /**
     * Adds a new property to the repository.
     */
    override fun addProperty(property: Property) {

        properties.add(property)
    }

    /**
     * Finds one property using its unique property ID.
     *
     * find() returns:
     * - Property → when a matching property exists
     * - null     → when no property is found
     */
    override fun getById(id: String): Property? {

        return properties.find {
            it.id == id
        }
    }

    /**
     * Searches for available properties in a city.
     *
     * Multiple properties can exist in the same city,
     * therefore this method returns a List<Property>.
     *
     * Only currently available properties are returned.
     */
    override fun searchByCity(city: String): List<Property> {

        return properties.filter {
            it.city == city && it.isAvailable
        }
    }
}
```

---

## 3️⃣ STRATEGY INTERFACE

> Here, the thing that "varies" is HOW we filter/sort search results.

> Today it might be by price, tomorrow by rating → so we make it an interface.

```kotlin
/**
 * Strategy interface for filtering property search results.
 *
 * Different filtering rules can be implemented without
 * changing the ListingService.
 */
interface PropertyFilterStrategy {

    /**
     * Filters the list of properties according
     * to the selected strategy.
     */
    fun filter(properties: List<Property>): List<Property>
}
```

### Concrete Strategy: Filter by Max Price

```kotlin
/**
 * Filters properties based on a maximum price per night.
 *
 * Example:
 * maxPrice = ₹2000
 *
 * Only properties costing ₹2000 or less will be returned.
 */
class PriceFilterStrategy(
    private val maxPrice: Double
) : PropertyFilterStrategy {

    override fun filter(
        properties: List<Property>
    ): List<Property> {

        return properties.filter {
            it.pricePerNight <= maxPrice
        }
    }
}
```

---

## 4️⃣ SERVICE CLASS

> This is where the REAL business logic lives.

> Interviewers pay most attention to this class.

```kotlin
/**
 * Service responsible for handling the main
 * property listing business logic.
 *
 * It coordinates:
 * - Property creation
 * - Property storage through PropertyRepository
 * - Property searching
 * - Optional filtering through PropertyFilterStrategy
 * - Property availability updates
 */
class ListingService(
    private val repository: PropertyRepository
) {

    /**
     * Creates a new property listing for a Host.
     *
     * The Service creates the Property object and
     * delegates storage to the Repository.
     */
    fun listProperty(
        host: Host,
        title: String,
        city: String,
        price: Double,
        type: PropertyType
    ): Property {

        // Generate a unique ID for the new property.
        val property = Property(
            id = UUID.randomUUID().toString(),
            host = host,
            title = title,
            city = city,
            pricePerNight = price,
            type = type
        )

        // Store the newly created property.
        repository.addProperty(property)

        return property
    }

    /**
     * Searches for properties in a particular city.
     *
     * A filter strategy is optional.
     *
     * Without a strategy:
     * → return all available properties.
     *
     * With a strategy:
     * → apply the selected filtering rule.
     */
    fun search(
        city: String,
        filterStrategy: PropertyFilterStrategy? = null
    ): List<Property> {

        // First get all available properties from the repository.
        val results = repository.searchByCity(city)

        // Apply the filter only when a strategy is provided.
        // Otherwise, return the complete search result.
        return filterStrategy?.filter(results) ?: results
    }

}
```

---

## 5️⃣ `main()`

> Always write a `main()` to PROVE your code compiles and runs.
> This is what panels want to see working live.

```kotlin
fun main() {

    // Create the in-memory repository.
    val repo = InMemoryPropertyRepository()

    // Inject the repository into the Service.
    val service = ListingService(repo)

    // Create the Host who will list the properties.
    val host = Host(
        id = "H1",
        name = "Rehan"
    )

    // Host lists the first property.
    service.listProperty(
        host,
        "Cozy Studio",
        "Bengaluru",
        1500.0,
        PropertyType.STUDIO
    )

    // Host lists the second property.
    service.listProperty(
        host,
        "Luxury Villa",
        "Bengaluru",
        8000.0,
        PropertyType.VILLA
    )

    // Guest searches for all available properties
    // in Bengaluru without applying any filter.
    println("===== All Properties in Bengaluru =====")

    service.search("Bengaluru")
        .forEach {
            println(
                " - ${it.title}: ₹${it.pricePerNight}"
            )
        }

    // Guest searches again, this time using
    // PriceFilterStrategy to find affordable properties.
    println("\n===== Properties Under ₹2000 =====")

    service.search(
        "Bengaluru",
        PriceFilterStrategy(maxPrice = 2000.0)
    ).forEach {
        println(
            " - ${it.title}: ₹${it.pricePerNight}"
        )
    }
}
```

---

# 🖥️ Sample Output

```text
===== All Properties in Bengaluru =====
 - Cozy Studio: ₹1500.0
 - Luxury Villa: ₹8000.0

===== Properties Under ₹2000 =====
 - Cozy Studio: ₹1500.0
```

---

## 🧠 Interview Flow

```text
Host
  │
  │ lists property
  ▼
ListingService
  │
  │ creates Property
  ▼
PropertyRepository
  │
  │ stores property
  ▼
InMemoryPropertyRepository


Guest
  │
  │ searches Bengaluru
  ▼
ListingService
  │
  ├───────────────► PropertyRepository
  │                       │
  │                       ▼
  │                  Available Properties
  │
  └───────────────► PropertyFilterStrategy
                          │
                          ▼
                     Filtered Results
```

### ⭐ Key point to remember

> **Property Listing → Strategy is used because the search filtering rule can vary.**

Today:

```text
Filter by maximum price
```

Tomorrow:

```text
Filter by rating
Filter by property type
Filter by distance
Sort by price
```

The `ListingService` doesn't need to change. We simply provide another `PropertyFilterStrategy`.
