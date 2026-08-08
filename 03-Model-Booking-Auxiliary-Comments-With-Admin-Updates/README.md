# 📝 LLD TOPIC: RESERVATION WITH AUXILIARY COMMENTS

### *(Navan-style twist)*

---

## 🎯 Overview

This is a variant of a booking system: instead of designing the whole booking flow again, the ask is **"extend a Reservation so it can carry comments/notes"** — e.g. an agent note like:

* `"guest requested late checkout"`
* `"visa docs pending"`

> Same **5-step skeleton** as always. The new piece is the **Comment entity** + how it attaches to a **Reservation**.

---

## 🧩 FRAMEWORK

| Step  | Component            | Purpose                                   |
| ----- | -------------------- | ----------------------------------------- |
| **1** | Enums + Data classes | Model reservations, users, and comments   |
| **2** | Repository           | Store and fetch reservations              |
| **3** | Strategy interface   | Who gets notified when a comment is added |
| **4** | Service class        | Comment business logic + visibility       |
| **5** | `main()`             | Proves everything works                   |

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

enum class ReservationStatus {
    PENDING,
    CONFIRMED,
    CANCELLED
}

// Who is allowed to see a comment - a common real-world requirement:
// internal ops notes vs comments visible to the traveler/customer.
enum class CommentVisibility {
    INTERNAL,
    CUSTOMER_VISIBLE
}

data class User(
    val id: String,
    val name: String
)

// A single comment/note attached to a reservation.
data class Comment(
    val id: String,
    val author: User,
    val text: String,
    val visibility: CommentVisibility,
    val createdAt: Long = System.currentTimeMillis()
)

// The Reservation now HAS a list of comments (composition).
// 'comments' is a MutableList because comments get added over time,
// but the list reference itself is never reassigned.
data class Reservation(
    val id: String,
    val guestName: String,
    val details: String, // e.g. "Flight BLR->DEL, 12 Aug" - kept generic on purpose
    var status: ReservationStatus = ReservationStatus.PENDING,
    val comments: MutableList<Comment> = mutableListOf()
)
```

---

# 2️⃣ REPOSITORY

> Only **ONE repository** needed — comments live inside the `Reservation` object, so we don't need a separate `CommentRepository`.

> Keep it simple unless the interviewer specifically asks for comments to be queried independently.

### Reservation Repository

```kotlin
interface ReservationRepository {

    fun save(reservation: Reservation)

    fun getById(id: String): Reservation?
}
```

### In-Memory Reservation Repository

```kotlin
class InMemoryReservationRepository : ReservationRepository {

    private val reservations = mutableListOf<Reservation>()

    override fun save(reservation: Reservation) {

        reservations.removeAll {
            it.id == reservation.id
        }

        reservations.add(reservation)
    }

    override fun getById(id: String) =
        reservations.find {
            it.id == id
        }
}
```

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = **WHO gets notified when a new comment is added.**

For example:

* Notify just the ops team
* Notify the customer too

### 🔔 Comment Notification Strategy

```kotlin
interface CommentNotificationStrategy {

    fun notify(
        reservation: Reservation,
        comment: Comment
    )
}
```

---

## 🏢 Internal-Only Notification Strategy

> Only notify internal ops — used for `INTERNAL` comments.

```kotlin
class InternalOnlyNotificationStrategy : CommentNotificationStrategy {

    override fun notify(
        reservation: Reservation,
        comment: Comment
    ) {

        println(
            "[Internal Alert] New note on reservation " +
            "${reservation.id}: \"${comment.text}\""
        )
    }
}
```

---

## 👤 Customer Notification Strategy

> Notify the customer as well — used for `CUSTOMER_VISIBLE` comments.

```kotlin
class CustomerNotificationStrategy : CommentNotificationStrategy {

    override fun notify(
        reservation: Reservation,
        comment: Comment
    ) {

        println(
            "[Internal Alert] New note on reservation " +
            "${reservation.id}: \"${comment.text}\""
        )

        println(
            "[Email to ${reservation.guestName}] " +
            "Update on your reservation: \"${comment.text}\""
        )
    }
}
```

---

# 4️⃣ SERVICE CLASS

> This is where the actual **"add comment" business logic** lives.

```kotlin
class ReservationService(
    private val repository: ReservationRepository
) {

    fun createReservation(
        guestName: String,
        details: String
    ): Reservation {

        val reservation = Reservation(
            id = UUID.randomUUID().toString(),
            guestName = guestName,
            details = details
        )

        repository.save(reservation)

        return reservation
    }

    // Adds a comment to a reservation and notifies the right audience
    // by picking a notification strategy based on the comment's visibility.
    fun addComment(
        reservationId: String,
        author: User,
        text: String,
        visibility: CommentVisibility
    ): Comment {

        val reservation = repository.getById(reservationId)
            ?: throw NoSuchElementException("Reservation not found")

        val comment = Comment(
            id = UUID.randomUUID().toString(),
            author = author,
            text = text,
            visibility = visibility
        )

        reservation.comments.add(comment)

        repository.save(reservation) // persist the updated reservation

        // Pick the right strategy based on visibility - this is our Strategy pattern in action
        val strategy: CommentNotificationStrategy =
            if (visibility == CommentVisibility.CUSTOMER_VISIBLE) {
                CustomerNotificationStrategy()
            } else {
                InternalOnlyNotificationStrategy()
            }

        strategy.notify(reservation, comment)

        return comment
    }

    // Fetch comments a given viewer is allowed to see.
    // e.g. a customer-facing app should only ever call this with isInternalViewer = false.
    fun getVisibleComments(
        reservationId: String,
        isInternalViewer: Boolean
    ): List<Comment> {

        val reservation = repository.getById(reservationId)
            ?: throw NoSuchElementException("Reservation not found")

        return if (isInternalViewer) {

            reservation.comments // internal viewers (ops/agents) see everything

        } else {

            reservation.comments.filter {
                it.visibility == CommentVisibility.CUSTOMER_VISIBLE
            }
        }
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

    val repo = InMemoryReservationRepository()
    val service = ReservationService(repo)

    val agent = User(
        "A1",
        "Support Agent"
    )

    val reservation = service.createReservation(
        "Rehan",
        "Flight BLR -> DEL, 12 Aug"
    )

    // Internal-only note - only ops should see this
    service.addComment(
        reservationId = reservation.id,
        author = agent,
        text = "Guest requested wheelchair assistance - confirm with airline",
        visibility = CommentVisibility.INTERNAL
    )

    // Customer-visible note - guest should be notified too
    service.addComment(
        reservationId = reservation.id,
        author = agent,
        text = "Your seat has been upgraded to window seat 14A",
        visibility = CommentVisibility.CUSTOMER_VISIBLE
    )

    println("\n--- What the CUSTOMER sees ---")

    service.getVisibleComments(
        reservation.id,
        isInternalViewer = false
    ).forEach {
        println(" - ${it.text}")
    }

    println("\n--- What INTERNAL OPS sees (everything) ---")

    service.getVisibleComments(
        reservation.id,
        isInternalViewer = true
    ).forEach {
        println(
            " - [${it.visibility}] ${it.text}"
        )
    }
}
```

---

## 🧠 Interview Focus

> 📝 **Comment as a separate entity** → keeps the reservation model clean while allowing multiple comments.

> 🔗 **Composition** → `Reservation` HAS a list of `Comment` objects.

> 🔔 **Notification Strategy** → notification behavior can vary based on comment visibility.

> 👁️ **Visibility** → internal users see everything, while customers only see `CUSTOMER_VISIBLE` comments.

> 🗄️ **Single Repository** → comments live inside the reservation, so a separate `CommentRepository` is unnecessary unless independent querying is required.

> 🧩 **Service Class** → coordinates comment creation, persistence, notification, and visibility.
