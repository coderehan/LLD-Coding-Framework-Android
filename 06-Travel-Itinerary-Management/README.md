# ✈️ LLD TOPIC: TRAVEL ITINERARY MANAGEMENT

### *()*

---

## 🎯 Goal

Build a **multi-day travel plan** made up of activities:

* Flights
* Hotels
* Sightseeing

Grouped by day.

> This topic is a great place to use the **BUILDER pattern**, because an `Itinerary` is a complex object built **step-by-step** (add day 1 activities, then day 2, etc.) instead of created all at once in a constructor.

---

## 🧩 FRAMEWORK

| Step  | Component            | Purpose                                         |
| ----- | -------------------- | ----------------------------------------------- |
| **1** | Enums + Data classes | The data/domain objects                         |
| **2** | Repository           | Stores and fetches itineraries                  |
| **3** | Builder              | Constructs the complex `Itinerary` step-by-step |
| **4** | Service class        | Business logic                                  |
| **5** | `main()`             | Proves everything works                         |

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

enum class ActivityType {
    FLIGHT,
    HOTEL,
    SIGHTSEEING,
    MEAL
}

data class Activity(
    val name: String,
    val type: ActivityType,
    val startTime: String, // keeping as String for simplicity, e.g. "10:00 AM"
    val location: String
)

// One day of the trip, holding a list of activities
data class DayPlan(
    val dayNumber: Int,
    val activities: MutableList<Activity> = mutableListOf()
)

data class Itinerary(
    val id: String,
    val tripName: String,
    val days: MutableList<DayPlan> = mutableListOf()
)
```

---

# 2️⃣ REPOSITORY

```kotlin
interface ItineraryRepository {

    fun save(itinerary: Itinerary)

    fun getById(id: String): Itinerary?
}
```

### In-Memory Repository

```kotlin
class InMemoryItineraryRepository : ItineraryRepository {

    private val itineraries = mutableListOf<Itinerary>()

    override fun save(itinerary: Itinerary) {

        // remove old version if it exists, then add updated one
        itineraries.removeAll { it.id == itinerary.id }

        itineraries.add(itinerary)
    }

    override fun getById(id: String) =
        itineraries.find { it.id == id }
}
```

---

# 3️⃣ BUILDER

> Instead of a Strategy interface, this topic fits the **BUILDER pattern** better, because we are constructing a complex `Itinerary` object step by step.

### 🧱 Itinerary Builder

```kotlin
class ItineraryBuilder(
    private val tripName: String
) {

    private val itinerary = Itinerary(
        id = UUID.randomUUID().toString(),
        tripName = tripName
    )

    // Add an activity to a specific day. If the day doesn't exist yet, create it.
    fun addActivity(
        dayNumber: Int,
        activity: Activity
    ): ItineraryBuilder {

        var day = itinerary.days.find {
            it.dayNumber == dayNumber
        }

        if (day == null) {
            day = DayPlan(dayNumber)
            itinerary.days.add(day)
        }

        day.activities.add(activity)

        return this // returning 'this' allows chaining: .addActivity().addActivity()...
    }

    fun build(): Itinerary {

        // sort days in order before finalizing
        itinerary.days.sortBy {
            it.dayNumber
        }

        return itinerary
    }
}
```

---

# 4️⃣ SERVICE CLASS

```kotlin
class ItineraryService(
    private val repository: ItineraryRepository
) {

    fun createItinerary(itinerary: Itinerary) {
        repository.save(itinerary)
    }

    fun getItinerary(id: String): Itinerary {
        return repository.getById(id)
            ?: throw NoSuchElementException("Itinerary not found")
    }

    // print a simple day-by-day view (useful to demo in interview)
    fun printItinerary(id: String) {

        val itinerary = getItinerary(id)

        println("Trip: ${itinerary.tripName}")

        for (day in itinerary.days) {

            println("Day ${day.dayNumber}:")

            for (activity in day.activities) {

                println(
                    "  ${activity.startTime} - " +
                    "${activity.name} (${activity.type}) " +
                    "at ${activity.location}"
                )
            }
        }
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

### 🏗️ Using the Builder

Using the Builder to construct a **2-day itinerary step by step**:

```kotlin
fun main() {

    // Using the Builder to construct a 2-day itinerary step by step
    val itinerary = ItineraryBuilder("Goa Trip")
        .addActivity(
            1,
            Activity(
                "Flight to Goa",
                ActivityType.FLIGHT,
                "8:00 AM",
                "Airport"
            )
        )
        .addActivity(
            1,
            Activity(
                "Check-in at Hotel",
                ActivityType.HOTEL,
                "11:00 AM",
                "Beach Resort"
            )
        )
        .addActivity(
            1,
            Activity(
                "Beach visit",
                ActivityType.SIGHTSEEING,
                "4:00 PM",
                "Baga Beach"
            )
        )
        .addActivity(
            2,
            Activity(
                "Fort visit",
                ActivityType.SIGHTSEEING,
                "10:00 AM",
                "Aguada Fort"
            )
        )
        .addActivity(
            2,
            Activity(
                "Dinner cruise",
                ActivityType.MEAL,
                "7:00 PM",
                "Mandovi River"
            )
        )
        .build()

    val repo = InMemoryItineraryRepository()
    val service = ItineraryService(repo)

    service.createItinerary(itinerary)

    service.printItinerary(itinerary.id)
}
```

