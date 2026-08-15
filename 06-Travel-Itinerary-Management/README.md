# ✈️ LLD TOPIC: TRAVEL ITINERARY MANAGEMENT

### (Navan, TripIt, Wanderlog, Google Travel, Expedia, Booking.com, MakeMyTrip, Tripadvisor, Airbnb)

---

## 🎯 Goal

Build a multi-day travel plan made up of activities:

- Flights
- Hotels
- Sightseeing
- Meals

Grouped by day.

> This topic is a great place to use the BUILDER pattern, because an `Itinerary` is a complex object built step-by-step (add day 1 activities, then day 2, etc.) instead of created all at once in a constructor.

---

## 🧩 FRAMEWORK

| Step | Component | Purpose |
|---|---|---|
| 1 | Enums + Data classes | The data/domain objects |
| 2 | Repository | Stores and fetches itineraries |
| 3 | Builder | Constructs the complex `Itinerary` step-by-step |
| 4 | Service class | Business logic |
| 5 | `main()` | Proves everything works |

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

/**
 * Represents the different types of activities
 * that can be added to a travel itinerary.
 */
enum class ActivityType {
    FLIGHT,
    HOTEL,
    SIGHTSEEING,
    MEAL
}

/**
 * Represents one activity in the travel plan.
 *
 * Example:
 * Flight to Goa
 * Check-in at Hotel
 * Beach visit
 */
data class Activity(
    val name: String,
    val type: ActivityType,
    val startTime: String, // Keeping as String for simplicity, e.g. "10:00 AM"
    val location: String
)

/**
 * Represents one day of the trip.
 *
 * Each day can contain multiple activities.
 */
data class DayPlan(
    val dayNumber: Int,
    val activities: MutableList<Activity> = mutableListOf()
)

/**
 * Represents the complete travel itinerary.
 *
 * An itinerary contains multiple DayPlan objects,
 * and each DayPlan contains multiple activities.
 */
data class Itinerary(
    val id: String,
    val tripName: String,
    val days: MutableList<DayPlan> = mutableListOf()
)
```

### 💡 Domain relationship

```text
Itinerary
    │
    ├── Day 1
    │     ├── Flight
    │     ├── Hotel
    │     └── Sightseeing
    │
    └── Day 2
          ├── Sightseeing
          └── Meal
```

---

# 2️⃣ REPOSITORY

```kotlin
/**
 * Repository responsible for storing and retrieving
 * travel itineraries.
 */
interface ItineraryRepository {

    /**
     * Saves a new itinerary or updates an existing itinerary.
     */
    fun save(itinerary: Itinerary)

    /**
     * Finds an itinerary using its unique ID.
     *
     * Returns:
     * - Itinerary → if found
     * - null → if no itinerary exists with that ID
     */
    fun getById(id: String): Itinerary?
}
```

### 💡 Repository responsibility

The repository handles **data access**, while the Service handles **business logic**.

```text
Service
   ↓
Repository
   ↓
Itinerary data
```

---

## 🗂️ In-Memory Repository

```kotlin
/**
 * In-memory implementation of ItineraryRepository.
 *
 * In a real application, itinerary data could come from
 * a database or backend API.
 *
 * For this LLD interview, we keep the data in memory.
 */
class InMemoryItineraryRepository : ItineraryRepository {

    // Stores all itineraries in memory.
    private val itineraries = mutableListOf<Itinerary>()

    /**
     * Saves an itinerary.
     *
     * If an itinerary with the same ID already exists,
     * remove the old version first and store the updated one.
     */
    override fun save(itinerary: Itinerary) {

        // Remove the previous version of this itinerary, if present.
        itineraries.removeAll {
            it.id == itinerary.id
        }

        // Store the latest version.
        itineraries.add(itinerary)
    }

    /**
     * Finds an itinerary using its unique ID.
     *
     * find() returns:
     * - matching itinerary if found
     * - null if not found
     */
    override fun getById(id: String): Itinerary? {

        return itineraries.find {
            it.id == id
        }
    }
}
```

---

# 3️⃣ BUILDER

> Instead of a Strategy interface, this topic fits the **BUILDER pattern** better, because we are constructing a complex `Itinerary` object step by step.

## 🧱 Itinerary Builder

```kotlin
/**
 * Builder responsible for constructing an Itinerary
 * step by step.
 *
 * Instead of passing all activities into the Itinerary
 * constructor at once, we can gradually add activities
 * for different days.
 */
class ItineraryBuilder(
    private val tripName: String
) {

    /**
     * Create the Itinerary that will be built.
     *
     * UUID generates a unique ID for the itinerary.
     */
    private val itinerary = Itinerary(
        id = UUID.randomUUID().toString(),
        tripName = tripName
    )

    /**
     * Adds an activity to a specific day.
     *
     * If the requested day doesn't exist yet,
     * a new DayPlan is created automatically.
     *
     * Returning 'this' allows method chaining:
     *
     * .addActivity()
     * .addActivity()
     * .addActivity()
     * .build()
     */
    fun addActivity(
        dayNumber: Int,
        activity: Activity
    ): ItineraryBuilder {

        // Check whether the requested day already exists.
        var day = itinerary.days.find {
            it.dayNumber == dayNumber
        }

        // If the day doesn't exist, create it.
        if (day == null) {
            day = DayPlan(dayNumber)

            itinerary.days.add(day)
        }

        // Add the activity to the selected day.
        day.activities.add(activity)

        // Return the same Builder object so method chaining is possible.
        return this
    }

    /**
     * Finalizes the itinerary and returns it.
     *
     * Days are sorted before returning so that
     * Day 1 comes before Day 2, Day 3, etc.
     */
    fun build(): Itinerary {

        itinerary.days.sortBy {
            it.dayNumber
        }

        return itinerary
    }
}
```

### 🧠 Builder flow

```text
ItineraryBuilder("Goa Trip")
        │
        ├── addActivity(Day 1, Flight)
        │
        ├── addActivity(Day 1, Hotel)
        │
        ├── addActivity(Day 1, Sightseeing)
        │
        ├── addActivity(Day 2, Sightseeing)
        │
        ├── addActivity(Day 2, Meal)
        │
        ▼
      build()
        │
        ▼
   Itinerary
```

---

# 4️⃣ SERVICE CLASS

```kotlin
/**
 * Service responsible for handling the main
 * travel itinerary business logic.
 *
 * It coordinates:
 * - Itinerary creation
 * - Saving itineraries through ItineraryRepository
 * - Retrieving itineraries
 * - Printing the itinerary in a day-by-day format
 */
class ItineraryService(
    private val repository: ItineraryRepository
) {

    /**
     * Saves the newly created itinerary.
     *
     * The Service delegates data storage to the Repository.
     */
    fun createItinerary(itinerary: Itinerary) {

        repository.save(itinerary)
    }

    /**
     * Retrieves an itinerary using its unique ID.
     *
     * If the itinerary doesn't exist,
     * throw an exception instead of returning null
     * to the caller.
     */
    fun getItinerary(id: String): Itinerary {

        return repository.getById(id)
            ?: throw NoSuchElementException("Itinerary not found")
    }
}
```

---

# 5️⃣ `main()`

> Always write a `main()` to **PROVE your code compiles and runs.**

## 🏗️ Using the Builder

```kotlin
fun main() {

    /**
     * Build a 2-day Goa itinerary step by step.
     *
     * Builder allows us to add activities one by one
     * without creating a large Itinerary constructor.
     */
    val itinerary = ItineraryBuilder("Goa Trip")

        // Day 1 → Flight
        .addActivity(
            1,
            Activity(
                "Flight to Goa",
                ActivityType.FLIGHT,
                "8:00 AM",
                "Airport"
            )
        )

        // Day 1 → Hotel check-in
        .addActivity(
            1,
            Activity(
                "Check-in at Hotel",
                ActivityType.HOTEL,
                "11:00 AM",
                "Beach Resort"
            )
        )

        // Day 1 → Sightseeing
        .addActivity(
            1,
            Activity(
                "Beach visit",
                ActivityType.SIGHTSEEING,
                "4:00 PM",
                "Baga Beach"
            )
        )

        // Day 2 → Sightseeing
        .addActivity(
            2,
            Activity(
                "Fort visit",
                ActivityType.SIGHTSEEING,
                "10:00 AM",
                "Aguada Fort"
            )
        )

        // Day 2 → Meal
        .addActivity(
            2,
            Activity(
                "Dinner cruise",
                ActivityType.MEAL,
                "7:00 PM",
                "Mandovi River"
            )
        )

        // Finalize the itinerary.
        .build()

    // Create the in-memory repository.
    val repo = InMemoryItineraryRepository()

    // Inject the repository into the Service.
    val service = ItineraryService(repo)

    // Save the completed itinerary.
    service.createItinerary(itinerary)

    // Retrieve and print the itinerary.
    service.printItinerary(itinerary.id)
}
```

---

# 🖥️ Sample Output

```text
===== Travel Itinerary =====
Trip: Goa Trip

Day 1:
  8:00 AM - Flight to Goa (FLIGHT) at Airport
  11:00 AM - Check-in at Hotel (HOTEL) at Beach Resort
  4:00 PM - Beach visit (SIGHTSEEING) at Baga Beach

Day 2:
  10:00 AM - Fort visit (SIGHTSEEING) at Aguada Fort
  7:00 PM - Dinner cruise (MEAL) at Mandovi River

=============================
```

---

# 🧠 Interview Focus

The most important thing to remember for this LLD:

### Why Builder?

`Itinerary` is a **complex object** that is built gradually.

Instead of:

```kotlin
Itinerary(
    id,
    tripName,
    day1Activities,
    day2Activities,
    day3Activities,
    ...
)
```

we can build it step-by-step:

```kotlin
ItineraryBuilder("Goa Trip")
    .addActivity(...)
    .addActivity(...)
    .addActivity(...)
    .build()
```

### Pattern used

```text
             Main.kt
                │
                ▼
       ItineraryBuilder
                │
          build step-by-step
                │
                ▼
           Itinerary
                │
                ▼
        ItineraryService
                │
                ▼
      ItineraryRepository
                │
                ▼
   InMemoryItineraryRepository
```

**Key takeaway:**  
> **Flight Booking → Strategy** for payment  
> **Hotel Booking → Strategy** for pricing  
> **Travel Itinerary → Builder** for constructing a complex itinerary
