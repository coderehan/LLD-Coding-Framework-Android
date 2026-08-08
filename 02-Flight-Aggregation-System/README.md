# ✈️ LLD TOPIC: FLIGHT AGGREGATION SYSTEM

### *(Skyscanner, Google Flights, Kayak, Momondo, Expedia, Wego, Cleartrip, MakeMyTrip, ixigo, EaseMyTrip)*

---

## 🎯 Goal

Pull flight data from **MULTIPLE airline sources** and show combined, ranked results to the user.

> **This topic is slightly different from others:** instead of **ONE repository**, we have **MULTIPLE data sources** (one per airline).

We use the **Repository pattern per-source**, and a **Strategy pattern** to **RANK** the combined results.

---

# 🧩 FRAMEWORK

| Step  | Component                | Purpose                                  |
| ----- | ------------------------ | ---------------------------------------- |
| **1** | Enums + Data classes     | The data/domain objects                  |
| **2** | Repository / Data Source | One per airline / data source            |
| **3** | Strategy interface       | How we rank/sort results                 |
| **4** | Service class            | Aggregates all sources + applies ranking |
| **5** | `main()`                 | Proves everything works                  |

---

## 1️⃣ ENUMS + DATA CLASSES

```kotlin
data class Flight(
    val flightNumber: String,
    val airline: String,
    val from: String,
    val to: String,
    val price: Double,
    val durationMinutes: Int
)
```

---

## 2️⃣ REPOSITORY

### *(one per airline / data source)*

> Each airline has its **OWN** way of giving us data.

We wrap each one behind the **SAME interface** → this is the **Adapter idea**.

### ✈️ Flight Data Source Interface

```kotlin
interface FlightDataSource {
    fun fetchFlights(
        from: String,
        to: String
    ): List<Flight>
}
```

### 🛫 Airline A Source

> Fake "Airline A" source

```kotlin
class AirlineASource : FlightDataSource {

    override fun fetchFlights(
        from: String,
        to: String
    ): List<Flight> {

        // Imagine this calls Airline A's real API. We hardcode for demo.
        return listOf(
            Flight(
                "AA101",
                "Airline A",
                from,
                to,
                price = 4500.0,
                durationMinutes = 120
            )
        )
    }
}
```

### 🛬 Airline B Source

> Fake "Airline B" source

```kotlin
class AirlineBSource : FlightDataSource {

    override fun fetchFlights(
        from: String,
        to: String
    ): List<Flight> {

        return listOf(
            Flight(
                "BB202",
                "Airline B",
                from,
                to,
                price = 3900.0,
                durationMinutes = 150
            )
        )
    }
}
```

---

## 3️⃣ STRATEGY INTERFACE

> The thing that **"varies"** here = **HOW we rank/sort the combined results.**

### 🔌 Ranking Strategy

```kotlin
interface RankingStrategy {
    fun rank(flights: List<Flight>): List<Flight>
}
```

### 💰 Cheapest First Strategy

```kotlin
class CheapestFirstStrategy : RankingStrategy {

    override fun rank(
        flights: List<Flight>
    ) = flights.sortedBy { it.price }
}
```

### ⚡ Fastest First Strategy

```kotlin
class FastestFirstStrategy : RankingStrategy {

    override fun rank(
        flights: List<Flight>
    ) = flights.sortedBy { it.durationMinutes }
}
```

---

## 4️⃣ SERVICE CLASS

> This is the **"Aggregator"** — it collects from **ALL sources** and applies ranking.

```kotlin
class FlightAggregationService(
    private val sources: List<FlightDataSource>
) {

    fun searchFlights(
        from: String,
        to: String,
        rankingStrategy: RankingStrategy
    ): List<Flight> {

        // Step A: collect flights from EVERY source
        val allFlights = mutableListOf<Flight>()

        for (source in sources) {
            allFlights.addAll(
                source.fetchFlights(from, to)
            )
        }

        // Step B: apply the ranking strategy chosen by the caller
        return rankingStrategy.rank(allFlights)
    }
}
```

---

## 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

    // Register all the airline sources we support
    val sources = listOf(
        AirlineASource(),
        AirlineBSource()
    )

    val aggregationService = FlightAggregationService(sources)

    println("Cheapest first:")

    aggregationService
        .searchFlights(
            "BLR",
            "DEL",
            CheapestFirstStrategy()
        )
        .forEach {
            println(
                " - ${it.airline} ${it.flightNumber}: " +
                "₹${it.price}, ${it.durationMinutes} min"
            )
        }

    println("\nFastest first:")

    aggregationService
        .searchFlights(
            "BLR",
            "DEL",
            FastestFirstStrategy()
        )
        .forEach {
            println(
                " - ${it.airline} ${it.flightNumber}: " +
                "₹${it.price}, ${it.durationMinutes} min"
            )
        }
}
```

