/**
 * ============================================================
 * LLD TOPIC: PROPERTY LISTING SERVICE (like Airbnb listings)
 * ============================================================
 * FRAMEWORK USED (same 5 steps every time):
 * 1. Enums + Data classes   -> the "nouns" of this domain
 * 2. Repository             -> where data is stored/fetched
 * 3. Strategy interface     -> the one thing that "varies"
 * 4. Service class          -> the actual business logic
 * 5. main()                 -> proves everything works
 * ============================================================
 */

import java.util.UUID

// ---------- STEP 1: ENUMS + DATA CLASSES ----------
// Enums = fixed set of options. Data classes = simple objects that just hold data.

enum class PropertyType { APARTMENT, VILLA, STUDIO }

// A Host is the person who owns/lists the property
data class Host(
    val id: String,
    val name: String
)

// A Property is the actual listing (like an Airbnb apartment)
data class Property(
    val id: String,
    val host: Host,
    val title: String,
    val city: String,
    val pricePerNight: Double,
    val type: PropertyType,
    var isAvailable: Boolean = true
)

// ---------- STEP 2: REPOSITORY ----------
// Repository's ONLY job = store data and let us search/fetch it.
// It does NOT contain business logic (that goes in Service, step 4).

interface PropertyRepository {
    fun addProperty(property: Property)
    fun getById(id: String): Property?
    fun searchByCity(city: String): List<Property>
}

// "InMemory" means we are just using a list instead of a real database.
// This is what you'll do in CoderPad since there's no real DB.
class InMemoryPropertyRepository : PropertyRepository {
    private val properties = mutableListOf<Property>()

    override fun addProperty(property: Property) {
        properties.add(property)
    }

    override fun getById(id: String): Property? {
        return properties.find { it.id == id }
    }

    override fun searchByCity(city: String): List<Property> {
        // only return properties that are available AND match the city
        return properties.filter { it.city == city && it.isAvailable }
    }
}

// ---------- STEP 3: STRATEGY INTERFACE ----------
// Here, the thing that "varies" is HOW we filter/sort search results.
// Today it might be by price, tomorrow by rating -> so we make it an interface.

interface PropertyFilterStrategy {
    fun filter(properties: List<Property>): List<Property>
}

// One concrete strategy: filter by max price
class PriceFilterStrategy(private val maxPrice: Double) : PropertyFilterStrategy {
    override fun filter(properties: List<Property>): List<Property> {
        return properties.filter { it.pricePerNight <= maxPrice }
    }
}

// ---------- STEP 4: SERVICE CLASS ----------
// This is where the REAL business logic lives.
// Interviewers pay most attention to this class.

class ListingService(private val repository: PropertyRepository) {

    // Host lists a new property
    fun listProperty(host: Host, title: String, city: String, price: Double, type: PropertyType): Property {
        val property = Property(
            id = UUID.randomUUID().toString(),
            host = host,
            title = title,
            city = city,
            pricePerNight = price,
            type = type
        )
        repository.addProperty(property)
        return property
    }

    // Guest searches for a property in a city, optionally applying a filter strategy
    fun search(city: String, filterStrategy: PropertyFilterStrategy? = null): List<Property> {
        val results = repository.searchByCity(city)
        // If a filter strategy is passed, apply it. Otherwise return everything.
        return filterStrategy?.filter(results) ?: results
    }

    // Mark a property as booked (not available anymore)
    fun markUnavailable(propertyId: String) {
        val property = repository.getById(propertyId)
            ?: throw NoSuchElementException("Property not found")
        property.isAvailable = false
    }
}

// ---------- STEP 5: main() ----------
// Always write a main() to PROVE your code compiles and runs.
// This is what panels want to see working live.

fun main() {
    val repo = InMemoryPropertyRepository()
    val service = ListingService(repo)

    val host = Host(id = "H1", name = "Rehan")

    // Host lists two properties
    service.listProperty(host, "Cozy Studio", "Bengaluru", 1500.0, PropertyType.STUDIO)
    service.listProperty(host, "Luxury Villa", "Bengaluru", 8000.0, PropertyType.VILLA)

    // Guest searches without filter
    println("All properties in Bengaluru:")
    service.search("Bengaluru").forEach { println(" - ${it.title}: ₹${it.pricePerNight}") }

    // Guest searches WITH a price filter (this is our Strategy pattern in action)
    println("\nProperties under ₹2000 in Bengaluru:")
    service.search("Bengaluru", PriceFilterStrategy(maxPrice = 2000.0))
        .forEach { println(" - ${it.title}: ₹${it.pricePerNight}") }
}
