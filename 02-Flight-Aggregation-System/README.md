# ✈️ LLD TOPIC: FLIGHT AGGREGATION SYSTEM

### (Skyscanner, Google Flights, Kayak, Momondo, Expedia, Wego, Cleartrip, MakeMyTrip, ixigo, EaseMyTrip)

---

## 🎯 Goal

Pull flight data from MULTIPLE airline sources and show combined, ranked results to the user.

> This topic is slightly different from others: instead of ONE repository, we have MULTIPLE data sources (one per airline).

We use the Repository / Data Source pattern per source, and a Strategy pattern to RANK the combined results.

---

# 🧩 FRAMEWORK

| Step | Component | Purpose |
|---|---|---|
| 1 | Enums + Data classes | The data/domain objects |
| 2 | Repository / Data Source | One per airline / data source |
| 3 | Strategy interface | How we rank/sort results |
| 4 | Service class | Aggregates all sources + applies ranking |
| 5 | `main()` | Proves everything works |

---

# Rough Diagram
    Main.kt
       │
       ▼
    FlightAggregationService.kt
       │
       ├───────────────────────────────┐
       │                               │
       ▼                               ▼
    FlightDataSource.kt          RankingStrategy.kt
       │                               │
       ├──────────────┐                ├──────────────┐
       ▼              ▼                ▼              ▼
    AirlineA       AirlineB      CheapestFirst    FastestFirst
    Source         Source          Strategy          Strategy
       │              │
       └──────┬───────┘
          ▼
       Flight Data Sources
          │
          ▼
    FlightAggregationService
          │
          ▼
       Aggregated + Ranked
            Flight List
---

## 1️⃣ ENUMS + DATA CLASSES

> Data class represents a flight returned by any airline source.

```kotlin
/**
 * Represents a flight returned by an airline or
 * external flight data provider.
 *
 * The aggregation system converts data from different
 * sources into this common Flight model.
 */
data class Flight(
    val flightNumber: String,
    val airline: String,
    val from: String,
    val to: String,
    val price: Double,
    val durationMinutes: Int
)
````

---

## 2️⃣ REPOSITORY / DATA SOURCE

### (one per airline / data source)

> Each airline has its OWN way of providing flight data.

> We hide those differences behind the SAME interface.
> This is the Adapter-style idea used in the aggregation layer.

### ✈️ Flight Data Source Interface

```kotlin
/**
 * Common interface used by the aggregation service
 * to communicate with different airline data sources.
 *
 * The service doesn't need to know how each airline
 * internally provides its flight data.
 */
interface FlightDataSource {

    /**
     * Fetch flights for the requested route.
     *
     * Multiple flights can be returned,
     * therefore the result is a List<Flight>.
     */
    fun fetchFlights(
        from: String,
        to: String
    ): List<Flight>
}
```

### 🛫 Airline A Source

> Fake "Airline A" source.

```kotlin
/**
 * Represents one airline's flight data source.
 *
 * In a real application, this class could call
 * Airline A's REST API and convert its response
 * into our common Flight model.
 *
 * For this LLD, we use hardcoded data.
 */
class AirlineASource : FlightDataSource {

    override fun fetchFlights(
        from: String,
        to: String
    ): List<Flight> {

        // Simulate fetching flight data from Airline A's API.
        return listOf(
            Flight(
                flightNumber = "AA101",
                airline = "Airline A",
                from = from,
                to = to,
                price = 4500.0,
                durationMinutes = 120
            )
        )
    }
}
```

### 🛬 Airline B Source

> Fake "Airline B" source.

```kotlin
/**
 * Represents another airline's flight data source.
 *
 * It follows the same FlightDataSource contract,
 * even though its real API could be completely different.
 */
class AirlineBSource : FlightDataSource {

    override fun fetchFlights(
        from: String,
        to: String
    ): List<Flight> {

        // Simulate fetching flight data from Airline B's API.
        return listOf(
            Flight(
                flightNumber = "BB202",
                airline = "Airline B",
                from = from,
                to = to,
                price = 3900.0,
                durationMinutes = 150
            )
        )
    }
}
```

---

## 3️⃣ STRATEGY INTERFACE

> The thing that "varies" here = HOW we rank/sort the combined results.

### 🔌 Ranking Strategy

```kotlin
/**
 * Defines how aggregated flights should be ranked.
 *
 * Different ranking rules can be introduced without
 * changing FlightAggregationService.
 */
interface RankingStrategy {

    /**
     * Receives all flights from all sources
     * and returns them in the desired order.
     */
    fun rank(
        flights: List<Flight>
    ): List<Flight>
}
```

### 💰 Cheapest First Strategy

```kotlin
/**
 * Ranks flights from the lowest price
 * to the highest price.
 */
class CheapestFirstStrategy: RankingStrategy{
  override fun rank(
    flights: List<Flight>
  ): List<Flight>{
    return flights.sortedBy{
      it.price
    }
  }
}
```

### ⚡ Fastest First Strategy

```kotlin
/**
 * Ranks flights from the shortest duration
 * to the longest duration.
 */
class FastestFirstStrategy: RankingStrategy{
  override fun rank(
    flights: List<Flight>
  ): List<Flight>{
    return flights.sortedBy{
      it.durationMinutes
    }
  }
}
```

---

## 4️⃣ SERVICE CLASS

> This is the **Aggregator** — it collects flights from ALL sources and applies the selected ranking strategy.

```kotlin
/**
 * Service responsible for aggregating flight data
 * from multiple airline sources.
 *
 * It coordinates:
 * - Multiple FlightDataSource implementations
 * - Combining results from all sources
 * - Ranking results using RankingStrategy
 */
class FlightAggregationService(
    private val sources: List<FlightDataSource>
) {

    /**
     * Searches for flights across ALL registered
     * airline data sources.
     *
     * The caller decides how the final results
     * should be ranked.
     */
    fun searchFlights(
        from: String,
        to: String,
        rankingStrategy: RankingStrategy
    ): List<Flight> {

        // Collect flight results from every airline source.
        val allFlights = mutableListOf<Flight>()

        for (source in sources) {

            // Each source may return multiple flights.
            // Add all results into one combined list.
            allFlights.addAll(
                source.fetchFlights(
                    from,
                    to
                )
            )
        }

        // Apply the ranking strategy selected by the caller.
        //
        // Example:
        // CheapestFirstStrategy → lowest price first
        // FastestFirstStrategy  → shortest duration first
        return rankingStrategy.rank(allFlights)
    }
}
```

---

## 5️⃣ `main()`

> Always write a `main()` to PROVE your code compiles and runs.

```kotlin
fun main() {

    // Register all airline sources supported
    // by our aggregation system.
    val sources = listOf(
        AirlineASource(),
        AirlineBSource()
    )

    // Inject all data sources into the aggregation service.
    val aggregationService =
        FlightAggregationService(sources)

    /**
     * Search BLR → DEL flights
     * and display the cheapest flights first.
     */
    println("===== Cheapest First =====")

    aggregationService
        .searchFlights(
            "BLR",
            "DEL",
            CheapestFirstStrategy()
        )
        .forEach {

            println(
                "Airline   : ${it.airline}\n" +
                "Flight    : ${it.flightNumber}\n" +
                "Route     : ${it.from} → ${it.to}\n" +
                "Price     : ₹${it.price}\n" +
                "Duration  : ${it.durationMinutes} min\n"
            )
        }

    /**
     * Search the same route again,
     * but rank flights by shortest duration.
     */
    println("===== Fastest First =====")

    aggregationService
        .searchFlights(
            "BLR",
            "DEL",
            FastestFirstStrategy()
        )
        .forEach {

            println(
                "Airline   : ${it.airline}\n" +
                "Flight    : ${it.flightNumber}\n" +
                "Route     : ${it.from} → ${it.to}\n" +
                "Price     : ₹${it.price}\n" +
                "Duration  : ${it.durationMinutes} min\n"
            )
        }
}
```

---

# 🖥️ Sample Output

```text
===== Cheapest First =====

Airline   : Airline B
Flight    : BB202
Route     : BLR → DEL
Price     : ₹3900.0
Duration  : 150 min

Airline   : Airline A
Flight    : AA101
Route     : BLR → DEL
Price     : ₹4500.0
Duration  : 120 min


===== Fastest First =====

Airline   : Airline A
Flight    : AA101
Route     : BLR → DEL
Price     : ₹4500.0
Duration  : 120 min

Airline   : Airline B
Flight    : BB202
Route     : BLR → DEL
Price     : ₹3900.0
Duration  : 150 min
```

---

## 🧠 Interview Flow

```text
                 User
                  │
                  │ Search BLR → DEL
                  ▼
        FlightAggregationService
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
   AirlineASource    AirlineBSource
          │                │
          │    Flight[]    │
          └───────┬────────┘
                  ▼
           Combined Flights
                  │
                  ▼
          RankingStrategy
          ┌───────┴────────┐
          │                │
          ▼                ▼
    CheapestFirst     FastestFirst
          │                │
          └───────┬────────┘
                  ▼
             Final Results
```

### ⭐ Key Point to Remember

> **Flight Aggregation = Multiple Data Sources + One Common Model + Strategy for Ranking**

```text
Multiple Airlines
       ↓
FlightDataSource
       ↓
Fetch from ALL sources
       ↓
Combine results
       ↓
RankingStrategy
       ↓
Final ranked flight list
```

### Patterns Used

* **Repository / Data Source** → abstracts each airline's data source.
* **Adapter-style abstraction** → different airline APIs can expose data through the same `FlightDataSource` interface.
* **Strategy Pattern** → ranking can change without modifying the aggregation service.
* **Service** → coordinates the complete aggregation flow.

```
