# 💰 LLD TOPIC: EXPENSE MANAGEMENT SYSTEM

### (Splitwise, Tricount, Settle Up, Splitser, Venmo, PayPal, Revolut)

---

## 🔄 SYSTEM COMMUNICATION FLOW

Before writing code, understand **who communicates with whom**.

```text
                         👤 User
                           │
                           │ creates expense
                           ▼
                  ┌─────────────────┐
                  │  ExpenseService  │
                  └────────┬────────┘
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
     ┌─────────────────┐        ┌─────────────────┐
     │  SplitStrategy   │        │ ExpenseRepository│
     └────────┬────────┘        └────────┬────────┘
              │                          │
       ┌──────┼────────┐                 │
       │      │        │                 │
       ▼      ▼        ▼                 ▼
    Equal  Percentage  Exact         Stores Expense
    Split    Split     Split              │
                                           │
                                           ▼
                                    ┌──────────────┐
                                    │   Expenses   │
                                    └──────┬───────┘
                                           │
                                           │ get all expenses
                                           ▼
                                  ┌─────────────────┐
                                  │  ExpenseService  │
                                  │                 │
                                  │ Calculate       │
                                  │ Balances        │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  👥 User Balances
````

### 🧠 Simple Explanation

Think about a real-life dinner with friends.

**1. User creates an expense**

> Alice paid ₹900 for dinner.

↓

**2. ExpenseService handles the business action**

> "How should this ₹900 be divided?"

↓

**3. SplitStrategy decides the rule**

It could be:

* Equal split
* Percentage split
* Exact split

↓

**4. ExpenseService creates the Expense**

The calculated splits are stored inside the `Expense` object.

↓

**5. Repository stores the Expense**

The Service doesn't care whether the data is stored in memory, database, API, etc.

↓

**6. User asks for balances**

ExpenseService reads all expenses from the Repository and calculates:

> Who should receive money and who owes money?

---

## ⭐ MAIN RESPONSIBILITY OF EACH COMPONENT

| Component          | Responsibility                    |
| ------------------ | --------------------------------- |
| **User**           | Participates in expenses          |
| **Expense**        | Represents one expense            |
| **Split**          | Represents how much one user owes |
| **Repository**     | Stores and retrieves expenses     |
| **SplitStrategy**  | Decides how an expense is divided |
| **ExpenseService** | Contains the business logic       |
| **main()**         | Demonstrates the complete flow    |

---

## 🔑 IMPORTANT BUSINESS FLOW

```text
Add Expense
     ↓
Choose Split Strategy
     ↓
Calculate Individual Shares
     ↓
Create Expense
     ↓
Save Expense in Repository
```

For balances:

```text
Get Balances
     ↓
Read All Expenses
     ↓
Credit Person Who Paid
     ↓
Debit Each Participant
     ↓
Calculate Net Balance
     ↓
Return Final Balances
```

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

### 💡 Why these models?

```text
User
 ↓
Who is involved?

Expense
 ↓
What was purchased/paid?

Split
 ↓
How much does each participant owe?
```

The domain models represent the **real-world nouns** of the Expense Management System.

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

### 💡 Why Repository?

The Service should **not directly manage where the data is stored**.

Today:

```text
ExpenseService
      ↓
InMemoryExpenseRepository
```

Tomorrow:

```text
ExpenseService
      ↓
DatabaseExpenseRepository
```

The Service doesn't need to change.

---

# 3️⃣ STRATEGY INTERFACE

> The thing that varies here = HOW an expense is split between users.

This is the textbook example of the Strategy pattern.

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

### 🧠 Why Strategy?

The **business rule that changes** is:

> "How should the expense be divided?"

Instead of putting all rules inside `ExpenseService`, we separate them.

```text
                  SplitStrategy
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Equal       Percentage      Exact
       Split          Split         Split
```

This makes it easy to add another splitting rule later.

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

### Real-life example

```text
Dinner = ₹900
People = 3

Alice → ₹300
Bob   → ₹300
Carol → ₹300
```

---

## 📊 Percentage Split Strategy

> Split by percentage

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

### Real-life example

```text
Dinner = ₹1000

Alice → 50% = ₹500
Bob   → 30% = ₹300
Carol → 20% = ₹200
```

---

## 💵 Exact Split Strategy

> Split by exact amounts

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

### Real-life example

```text
Movie tickets = ₹500

Alice → ₹300
Carol → ₹200
```

---

# 4️⃣ SERVICE CLASS ⭐

The **Service is the most important part** of this LLD.

Why?

Because this is where the **business actions and business rules** live.

```text
ExpenseService

    ├── Add Expense
    │      ↓
    │   Choose Strategy
    │      ↓
    │   Calculate Splits
    │      ↓
    │   Create Expense
    │      ↓
    │   Save Expense
    │
    └── Get Balances
           ↓
        Read Expenses
           ↓
        Calculate Net Balance
```

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

    // Calculate net balance:
    // positive = should receive
    // negative = owes
    fun getBalances(): Map<String, Double> {

        val balances = mutableMapOf<String, Double>()

        // userId -> net amount
        // +ve = should receive
        // -ve = owes

        for (expense in repository.getAllExpenses()) {

            // Person who paid gets credited the full amount
            balances[expense.paidBy.id] =
                (balances[expense.paidBy.id] ?: 0.0) + expense.amount

            // Each participant gets debited their share
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

# 🧠 SERVICE ACTIONS TO REMEMBER

For this LLD, don't try to memorize the whole Service code.

Remember:

```text
💰 EXPENSE SERVICE

1. ADD EXPENSE
       ↓
   Select Split Strategy
       ↓
   Calculate Splits
       ↓
   Create Expense
       ↓
   Save in Repository

2. GET BALANCES
       ↓
   Read All Expenses
       ↓
   Credit Person Who Paid
       ↓
   Debit Participants
       ↓
   Calculate Net Balance
```

### 🎯 Memory Shortcut

> **ADD → SPLIT → SAVE**

> **GET → CREDIT → DEBIT → BALANCE**

---

# 5️⃣ `main()`

> Always write a `main()` to PROVE your code compiles and runs.

```kotlin
fun main() {

    val repo = InMemoryExpenseRepository()
    val service = ExpenseService(repo)

    val alice = User("U1", "Alice")
    val bob = User("U2", "Bob")
    val carol = User("U3", "Carol")

    // Example 1: Equal split
    // Alice paid ₹900 for dinner for Alice, Bob and Carol.
    service.addExpense(
        description = "Dinner",
        amount = 900.0,
        paidBy = alice,
        participants = listOf(alice, bob, carol),
        strategy = EqualSplitStrategy()
    )

    // Example 2: Exact split
    // Bob paid ₹500 for movie tickets.
    // Alice owes ₹300 and Carol owes ₹200.
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

---

# 🖥️ SAMPLE OUTPUT

```text
Final balances (positive = should receive money, negative = owes money):
 - U1: ₹0.0
 - U2: ₹0.0
 - U3: ₹-200.0
```

### 🧠 Let's understand the output

#### Dinner — ₹900

Alice paid the entire ₹900.

```text
Alice → +₹900 paid
Alice → -₹300 share
Bob   → -₹300 share
Carol → -₹300 share
```

Result:

```text
Alice → +₹600
Bob   → -₹300
Carol → -₹300
```

#### Movie tickets — ₹500

Bob paid ₹500.

```text
Bob   → +₹500 paid
Alice → -₹300 share
Carol → -₹200 share
```

After combining both expenses:

```text
Alice → +₹600 - ₹300 = +₹300
Bob   → -₹300 + ₹500 = +₹200
Carol → -₹300 - ₹200 = -₹500
```

So the expected final balances for the complete example are:

```text
Alice → +₹300
Bob   → +₹200
Carol → -₹500
```

> **Positive (+)** → user should receive money
> **Negative (-)** → user owes money

---

# 🎯 FINAL LLD FLOW

```text
                    👤 USERS
                       │
                       ▼
                ┌──────────────┐
                │ ExpenseService│
                └──────┬───────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
      ┌───────────────┐  ┌─────────────────┐
      │ SplitStrategy │  │ExpenseRepository│
      └───────┬───────┘  └────────┬────────┘
              │                   │
       ┌──────┼──────┐            │
       ▼      ▼      ▼            ▼
     Equal  Percent  Exact      Expenses
      Split   Split   Split         │
                                    │
                                    ▼
                              Get Balances
                                    │
                                    ▼
                              👥 BALANCES
```

### 🔥 Interview Memory Trick

```text
WHAT EXISTS?
     ↓
User, Expense, Split
     ↓
DOMAIN

WHERE IS DATA?
     ↓
ExpenseRepository

WHAT CHANGES?
     ↓
How expense is split
     ↓
SplitStrategy

WHAT ACTIONS?
     ↓
Add Expense
Get Balances
     ↓
ExpenseService ⭐

HOW DO I PROVE IT?
     ↓
main()
```

### 💡 The most important thing to remember

> **Expense Management = ADD → SPLIT → BALANCE**

And the Strategy pattern exists because:

> **The way we split an expense can change.**

```text
Equal
Percentage
Exact
```

That is the core business logic of this LLD.

```

**One correction to your current README:** the sample output currently implied by the code should be **U1 = ₹300, U2 = ₹200, U3 = -₹500**, not `0, 0, -200`. The code itself produces the former because Alice and Bob are net receivers after the two expenses. :contentReference[oaicite:1]{index=1}
```
