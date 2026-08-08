# 💰 LLD TOPIC: EXPENSE MANAGEMENT SYSTEM

### *(like Splitwise)*

---

> **This is the MOST commonly asked LLD topic** because it has a very clean Strategy pattern use case: **SPLITTING** an expense can be done in different ways (**equal / percentage / exact**).

---

## 🧩 FRAMEWORK

| Step  | Component            | Purpose                              |
| ----- | -------------------- | ------------------------------------ |
| **1** | Enums + Data classes | The data/domain objects              |
| **2** | Repository           | Stores expenses                      |
| **3** | Strategy interface   | How an expense is split              |
| **4** | Service class        | Business logic + balance calculation |
| **5** | `main()`             | Proves everything works              |

---

# 1️⃣ ENUMS + DATA CLASSES

```kotlin
import java.util.UUID

enum class SplitType {
    EQUAL,
    PERCENTAGE,
    EXACT
}

data class User(
    val id: String,
    val name: String
)

// How much ONE user owes for a particular expense
data class Split(
    val user: User,
    val amount: Double
)

data class Expense(
    val id: String,
    val description: String,
    val amount: Double,
    val paidBy: User,
    val splits: List<Split> // who owes how much for this expense
)
```

---

# 2️⃣ REPOSITORY

> Stores expenses and keeps a running balance sheet between users.

### Expense Repository

```kotlin
interface ExpenseRepository {

    fun addExpense(expense: Expense)

    fun getAllExpenses(): List<Expense>
}
```

### In-Memory Repository

```kotlin
class InMemoryExpenseRepository : ExpenseRepository {

    private val expenses = mutableListOf<Expense>()

    override fun addExpense(expense: Expense) {
        expenses.add(expense)
    }

    override fun getAllExpenses(): List<Expense> = expenses
}
```

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = **HOW an expense is split between users.**

This is the textbook example of the **Strategy pattern**.

### 🔌 Split Strategy

```kotlin
interface SplitStrategy {

    // participants = everyone involved in this expense (excluding logic of who paid)
    fun split(
        totalAmount: Double,
        participants: List<User>,
        extraData: Map<String, Double> = emptyMap()
    ): List<Split>
}
```

---

## ⚖️ Equal Split Strategy

> Split equally among everyone

```kotlin
class EqualSplitStrategy : SplitStrategy {

    override fun split(
        totalAmount: Double,
        participants: List<User>,
        extraData: Map<String, Double>
    ): List<Split> {

        val share = totalAmount / participants.size

        return participants.map {
            Split(it, share)
        }
    }
}
```

---

## 📊 Percentage Split Strategy

> Split by percentage, e.g. `{"user1": 50.0, "user2": 50.0}`

```kotlin
class PercentageSplitStrategy : SplitStrategy {

    override fun split(
        totalAmount: Double,
        participants: List<User>,
        extraData: Map<String, Double>
    ): List<Split> {

        return participants.map { user ->

            val percent = extraData[user.id] ?: 0.0

            Split(
                user,
                totalAmount * (percent / 100)
            )
        }
    }
}
```

---

## 💵 Exact Split Strategy

> Split by exact amounts, e.g. `{"user1": 300.0, "user2": 200.0}`

```kotlin
class ExactSplitStrategy : SplitStrategy {

    override fun split(
        totalAmount: Double,
        participants: List<User>,
        extraData: Map<String, Double>
    ): List<Split> {

        return participants.map { user ->

            Split(
                user,
                extraData[user.id] ?: 0.0
            )
        }
    }
}
```

---

# 4️⃣ SERVICE CLASS

```kotlin
class ExpenseService(
    private val repository: ExpenseRepository
) {

    // Add a new expense using whichever split strategy is passed in
    fun addExpense(
        description: String,
        amount: Double,
        paidBy: User,
        participants: List<User>,
        strategy: SplitStrategy,
        extraData: Map<String, Double> = emptyMap()
    ): Expense {

        val splits = strategy.split(
            amount,
            participants,
            extraData
        )

        val expense = Expense(
            id = UUID.randomUUID().toString(),
            description = description,
            amount = amount,
            paidBy = paidBy,
            splits = splits
        )

        repository.addExpense(expense)

        return expense
    }

    // Calculate net balance: who owes whom, in simple form (per user, how much they owe overall)
    fun getBalances(): Map<String, Double> {

        val balances = mutableMapOf<String, Double>()
        // userId -> net amount (+ve = should receive, -ve = owes)

        for (expense in repository.getAllExpenses()) {

            // person who paid gets credited the full amount
            balances[expense.paidBy.id] =
                (balances[expense.paidBy.id] ?: 0.0) + expense.amount

            // each participant in the split gets debited their share
            for (split in expense.splits) {

                balances[split.user.id] =
                    (balances[split.user.id] ?: 0.0) - split.amount
            }
        }

        return balances
    }
}
```

---

# 5️⃣ `main()`

> **Always write a `main()` to PROVE your code compiles and runs.**

```kotlin
fun main() {

    val repo = InMemoryExpenseRepository()
    val service = ExpenseService(repo)

    val alice = User("U1", "Alice")
    val bob = User("U2", "Bob")
    val carol = User("U3", "Carol")

    // Example 1: Equal split - dinner for 3 people, Alice paid ₹900
    service.addExpense(
        description = "Dinner",
        amount = 900.0,
        paidBy = alice,
        participants = listOf(alice, bob, carol),
        strategy = EqualSplitStrategy()
    )

    // Example 2: Exact split - Bob paid ₹500 for movie tickets, split unevenly
    service.addExpense(
        description = "Movie tickets",
        amount = 500.0,
        paidBy = bob,
        participants = listOf(alice, carol),
        strategy = ExactSplitStrategy(),
        extraData = mapOf(
            "U1" to 300.0,
            "U3" to 200.0
        )
    )

    println(
        "Final balances " +
        "(positive = should receive money, negative = owes money):"
    )

    service.getBalances()
        .forEach { (userId, balance) ->
            println(" - $userId: ₹$balance")
        }
}
```

