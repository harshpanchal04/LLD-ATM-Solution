# 🏧 ATM Machine — Low Level Design (LLD)

![Java](https://img.shields.io/badge/Java-17%2B-ED8B00?logo=openjdk&logoColor=white)
![Design Pattern](https://img.shields.io/badge/Pattern-State%20%7C%20Factory%20%7C%20Singleton-blue)
![Architecture](https://img.shields.io/badge/Architecture-Low%20Level%20Design-green)
![License](https://img.shields.io/badge/License-MIT-yellow)

> A production-style Java implementation of an **ATM system** built with the **State design pattern** at its core, complemented by Factory, Singleton, and Greedy algorithmic strategies. Designed as a clean reference for **low-level design (LLD) interviews** and **object-oriented design** practice.

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [State Machine & Transitions](#-state-machine--transitions)
- [Design Patterns](#-design-patterns-deep-dive)
- [Class Structure](#-class-structure)
- [Key Features](#-key-features)
- [SOLID Principles](#-solid-principles)
- [OOP Concepts](#-oop-concepts)
- [How to Run](#-how-to-run)
- [Sample Output](#-sample-output)
- [Project Structure](#-project-structure)

---

## 🏗 Architecture Overview

The system models a real-world ATM through a **finite state machine** where user actions drive transitions between well-defined states. The `ATMMachineContext` acts as the central orchestrator, delegating behavior to the current `ATMState` and coordinating between domain objects (`Card`, `Account`, `ATMInventory`).

```
┌─────────────────────────────────────────────────────────────────┐
│                      ATMMachineContext                          │
│                      (State Machine)                           │
│                                                                 │
│  ┌─────────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │  IdleState   │──▶│HasCard   │──▶│SelectOp  │──▶│Transact  │ │
│  │             │   │  State   │   │  State   │   │  State   │ │
│  └─────────────┘   └──────────┘   └──────────┘   └──────────┘ │
│         ▲                                              │       │
│         └──────────────────────────────────────────────┘       │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐                │
│  │   Card   │  │ Account  │  │ ATMInventory  │                │
│  └──────────┘  └──────────┘  └───────────────┘                │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │  creates states
  ┌──────┴──────────┐
  │ ATMStateFactory  │  (Singleton)
  └─────────────────┘
```

---

## 🔄 State Machine & Transitions

The ATM operates as a **four-state finite automaton**. Each state encapsulates its own transition logic via the `next()` method, keeping state management decoupled from business logic.

### State Transition Diagram

```
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │   ╔═══════════╗   card      ╔═══════════════╗               │
    │   ║           ║  inserted   ║               ║               │
    │   ║   IDLE    ║────────────▶║  HAS CARD     ║               │
    │   ║           ║             ║               ║               │
    │   ╚═══════════╝             ╚═══════╤═══════╝               │
    │        ▲                            │                       │
    │        │                       PIN  │                       │
    │        │                   verified │                       │
    │        │                            ▼                       │
    │        │                    ╔═══════════════╗               │
    │   card │                    ║    SELECT     ║◀──────┐       │
    │  eject │                    ║  OPERATION    ║       │       │
    │        │                    ╚═══════╤═══════╝       │       │
    │        │                            │               │       │
    │        │                  operation │          next  │       │
    │        │                   selected │      operation │       │
    │        │                            ▼               │       │
    │        │                    ╔═══════════════╗       │       │
    │        │                    ║  TRANSACTION  ║───────┘       │
    │        └────────────────────║               ║               │
    │          session ends       ╚═══════════════╝               │
    │                                                              │
    └──────────────────────────────────────────────────────────────┘
```

### Transition Table

| Current State        | Trigger              | Next State           | Guard Condition                |
|:---------------------|:---------------------|:---------------------|:-------------------------------|
| `IdleState`          | Card inserted        | `HasCardState`       | `currentCard != null`          |
| `HasCardState`       | PIN verified         | `SelectOperationState`| `currentAccount != null`      |
| `HasCardState`       | Card ejected         | `IdleState`          | `currentCard == null`          |
| `SelectOperationState`| Operation chosen    | `TransactionState`   | `selectedOperation != null`    |
| `TransactionState`   | Transaction complete | `SelectOperationState`| Card still present (loop)     |
| `TransactionState`   | Card ejected         | `IdleState`          | `currentCard == null`          |

---

## 🧩 Design Patterns Deep-Dive

### 1. State Pattern *(Core Architecture)*

The **State pattern** eliminates complex `if-else` chains by encapsulating each ATM state as a separate class. The context object delegates behavior to the current state, enabling clean transitions and adherence to the Open/Closed Principle.

| Component        | Role               | Class                       |
|:-----------------|:-------------------|:----------------------------|
| **State**        | Interface           | `ATMState`                  |
| **Context**      | State Machine Host  | `ATMMachineContext`         |
| **ConcreteState** | Idle               | `IdleState`                 |
| **ConcreteState** | Card Inserted      | `HasCardState`              |
| **ConcreteState** | Operation Menu     | `SelectOperationState`      |
| **ConcreteState** | Executing Txn      | `TransactionState`          |

```java
// Each state decides its own successor — no centralized switch/case
public class IdleState implements ATMState {
    @Override
    public ATMState next(ATMMachineContext context) {
        if (context.getCurrentCard() != null) {
            return context.getStateFactory().createHasCardState();
        }
        return this;  // remain idle
    }
}
```

### 2. Factory Pattern

`ATMStateFactory` centralizes all state object creation, providing a single place to control instantiation. This decouples state consumers from concrete state constructors.

```java
public class ATMStateFactory {
    public ATMState createIdleState()            { return new IdleState();            }
    public ATMState createHasCardState()         { return new HasCardState();         }
    public ATMState createSelectOperationState() { return new SelectOperationState(); }
    public ATMState createTransactionState()     { return new TransactionState();     }
}
```

### 3. Singleton Pattern

The `ATMStateFactory` is implemented as a **lazy-initialized Singleton**, ensuring a single factory instance across the entire ATM lifecycle.

```java
public class ATMStateFactory {
    private static ATMStateFactory instance = null;
    private ATMStateFactory() {}  // private constructor

    public static ATMStateFactory getInstance() {
        if (instance == null) {
            instance = new ATMStateFactory();
        }
        return instance;
    }
}
```

### 4. Greedy Algorithm — Cash Dispensing

`ATMInventory.dispenseCash()` applies a **greedy strategy**: iterate denominations from largest (`$100`) to smallest (`$1`), dispensing as many bills as possible at each step. If the exact amount cannot be composed, it performs a **full rollback** restoring all bills to inventory.

```java
// Greedy: largest-denomination-first, with transactional rollback
for (CashType cashType : CashType.values()) {   // $100 → $50 → $20 → $10 → $5 → $1
    int count = Math.min(remaining / cashType.value, available);
    if (count > 0) {
        dispensed.put(cashType, count);
        remaining -= count * cashType.value;
    }
}
if (remaining > 0) { /* rollback all dispensed bills */ }
```

---

## 📐 Class Structure

```
┌──────────────────────────────────────────────────────────────┐
│                       «interface»                            │
│                        ATMState                              │
│──────────────────────────────────────────────────────────────│
│ + getStateName() : String                                    │
│ + next(context: ATMMachineContext) : ATMState                │
└──────────────┬───────────────────────────────────────────────┘
               │ implements
    ┌──────────┼──────────┬──────────────────┐
    │          │          │                  │
    ▼          ▼          ▼                  ▼
┌────────┐ ┌────────┐ ┌──────────────┐ ┌──────────────┐
│Idle    │ │HasCard │ │SelectOp      │ │Transaction   │
│State   │ │State   │ │State         │ │State         │
└────────┘ └────────┘ └──────────────┘ └──────────────┘

┌──────────────────────────────────────────────────────────────┐
│                   ATMMachineContext                           │
│──────────────────────────────────────────────────────────────│
│ - currentState    : ATMState                                 │
│ - currentCard     : Card                                     │
│ - currentAccount  : Account                                  │
│ - atmInventory    : ATMInventory                             │
│ - accounts        : Map<String, Account>                     │
│ - stateFactory    : ATMStateFactory                          │
│ - selectedOp      : TransactionType                          │
│──────────────────────────────────────────────────────────────│
│ + insertCard(card)           + performTransaction(amount)    │
│ + enterPin(pin)              + returnCard()                  │
│ + selectOperation(type)      + cancelTransaction()           │
│ + advanceState()             - performWithdrawal(amount)     │
│ + addAccount(account)        - checkBalance()                │
│ + getAccount(number)         - resetATM()                    │
└──────────────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐
│     Card     │  │   Account    │  │      ATMInventory        │
│──────────────│  │──────────────│  │──────────────────────────│
│ - cardNumber │  │ - accountNum │  │ - cashInventory:         │
│ - pin        │  │ - balance    │  │   Map<CashType, Integer> │
│ - accountNum │  │──────────────│  │──────────────────────────│
│──────────────│  │ + withdraw() │  │ + dispenseCash(amount)   │
│ + validatePin│  │ + deposit()  │  │ + hasSufficientCash()    │
│ + getCardNum │  │ + getBalance │  │ + addCash(type, count)   │
│ + getAcctNum │  │ + getAcctNum │  │ + getTotalCash()         │
└──────────────┘  └──────────────┘  └──────────────────────────┘

┌──────────────────────────┐  ┌──────────────────────────────┐
│   «enum» CashType        │  │   «enum» TransactionType     │
│──────────────────────────│  │──────────────────────────────│
│ BILL_100(100)            │  │ WITHDRAW_CASH                │
│ BILL_50 (50)             │  │ CHECK_BALANCE                │
│ BILL_20 (20)             │  └──────────────────────────────┘
│ BILL_10 (10)             │
│ BILL_5  (5)              │  ┌──────────────────────────────┐
│ BILL_1  (1)              │  │  ATMStateFactory «singleton» │
│──────────────────────────│  │──────────────────────────────│
│ + value : int            │  │ - instance : ATMStateFactory │
└──────────────────────────┘  │ - ATMStateFactory()          │
                              │ + getInstance()              │
                              │ + createIdleState()          │
                              │ + createHasCardState()       │
                              │ + createSelectOpState()      │
                              │ + createTransactionState()   │
                              └──────────────────────────────┘
```

---

## ✨ Key Features

### Two-Phase Withdrawal with Rollback

The withdrawal flow uses a **debit-first, dispense-second** strategy with automatic rollback — mimicking real-world ATM transactional safety:

```
Step 1 ──▶ Debit account          (Account.withdraw)
Step 2 ──▶ Check ATM inventory    (ATMInventory.hasSufficientCash)
Step 3 ──▶ Dispense bills         (ATMInventory.dispenseCash)

         ┌──── Failure at Step 2 or 3 ────┐
         │  Account.deposit(amount)        │  ◀── automatic rollback
         │  Throw exception to caller      │
         └─────────────────────────────────┘
```

### Cash Inventory System

Six denominations managed via a `HashMap<CashType, Integer>` with default stock:

| Denomination | Initial Count | Total Value |
|:-------------|:-------------:|:-----------:|
| `$100`       | 10            | $1,000      |
| `$50`        | 10            | $500        |
| `$20`        | 20            | $400        |
| `$10`        | 30            | $300        |
| `$5`         | 20            | $100        |
| `$1`         | 50            | $50         |
| **Total**    | **140 bills** | **$2,350**  |

### Multi-Transaction Sessions

After a transaction completes, the state machine loops back to `SelectOperationState` rather than ejecting the card — allowing the user to perform multiple operations (withdraw, check balance) in a single session before explicitly returning the card.

### PIN Validation

PIN verification is **encapsulated within the `Card` class** itself (`validatePin()`), ensuring that sensitive authentication logic stays co-located with the credential it protects — no external class ever reads the raw PIN.

---

## 📏 SOLID Principles

| Principle | Application | Evidence |
|:----------|:------------|:---------|
| **S** — Single Responsibility | Each state class handles only its own transition logic | `IdleState` manages idle→hasCard; `TransactionState` manages txn→selectOp |
| **O** — Open/Closed | New states (e.g., `MaintenanceState`) can be added without modifying existing state classes | Add a new `ATMState` implementation + factory method |
| **L** — Liskov Substitution | All four state implementations are interchangeable via the `ATMState` interface | `ATMMachineContext.currentState` works with any `ATMState` |
| **I** — Interface Segregation | `ATMState` defines only two methods (`getStateName`, `next`) — no bloated interfaces | Concrete states aren't forced to implement irrelevant methods |
| **D** — Dependency Inversion | `ATMMachineContext` depends on the `ATMState` abstraction, not concrete state classes | State creation is delegated to `ATMStateFactory` |

> **Note:** `ATMMachineContext` uses `instanceof` checks (e.g., `currentState instanceof IdleState`) to guard public operations. This introduces minor coupling to concrete types — a trade-off for explicit precondition enforcement versus a pure state-driven dispatch approach.

---

## 🧬 OOP Concepts

| Concept | Implementation |
|:--------|:---------------|
| **Encapsulation** | `Card.pin` and `Account.balance` are `private` — exposed only through controlled methods (`validatePin()`, `withdraw()`, `deposit()`) |
| **Polymorphism** | `ATMMachineContext.advanceState()` calls `currentState.next(this)` — dynamic dispatch selects the correct transition logic at runtime |
| **Abstraction** | The `ATMState` interface hides state-specific behavior behind a uniform contract; callers never know which concrete state is active |
| **Composition over Inheritance** | `ATMMachineContext` *composes* `ATMState`, `Card`, `Account`, and `ATMInventory` rather than inheriting from them — favoring flexible object graphs over rigid class hierarchies |

---

## 🚀 How to Run

### Prerequisites

- **Java JDK 8+** (tested with JDK 17)

### Compile & Execute

```bash
# Navigate to the project directory
cd LLD-ATM-Solution

# Compile all source files
javac *.java

# Run the demo
java ATMDemo
```

### What the Demo Does

The `ATMDemo` class simulates a complete ATM session:

1. Initializes the ATM with two accounts (`$1,000` and `$500` balances)
2. Inserts a card and authenticates with PIN `1234`
3. Withdraws `$100` — dispenses bills using the greedy algorithm
4. Checks remaining balance
5. Returns the card, resetting the machine to `IdleState`

---

## 💻 Sample Output

```
ATM initialized in: IdleState
=== Starting ATM Demo ===
Card inserted
ATM is in Has Card State - Please enter your PIN
Current state: HasCardState
PIN authenticated successfully
ATM is in Select Operation State - Please select an operation
1. Withdraw Cash
2. Check Balance
Current state: SelectOperationState
Selected operation: WITHDRAW_CASH
ATM is in Transaction State
Current state: TransactionState
Transaction successful. Please collect your cash:
1 x $100
ATM is in Select Operation State - Please select an operation
1. Withdraw Cash
2. Check Balance
Current state: SelectOperationState
Selected operation: CHECK_BALANCE
ATM is in Transaction State
Current state: TransactionState
Your current balance is: $400.0
ATM is in Select Operation State - Please select an operation
1. Withdraw Cash
2. Check Balance
Current state: SelectOperationState
Card returned to customer
ATM is in Idle State - Please insert your card
=== ATM Demo Completed ===
```

---

## 📁 Project Structure

```
LLD-ATM-Solution/
│
├── ATMState.java               ← State interface (2 methods)
├── IdleState.java              ← Waiting for card insertion
├── HasCardState.java           ← Card present, awaiting PIN
├── SelectOperationState.java   ← PIN verified, choosing operation
├── TransactionState.java       ← Executing withdrawal or balance check
│
├── ATMMachineContext.java      ← State machine context & business logic
├── ATMStateFactory.java        ← Singleton factory for state creation
│
├── Card.java                   ← Card entity with PIN validation
├── Account.java                ← Bank account with withdraw/deposit
├── ATMInventory.java           ← Cash inventory with greedy dispensing
│
├── CashType.java               ← Enum: 6 bill denominations
├── TransactionType.java        ← Enum: WITHDRAW_CASH, CHECK_BALANCE
│
├── ATMDemo.java                ← Entry point — end-to-end demo
└── README.md
```

**13 Java source files** · **~530 lines of code** · **Zero external dependencies**

---

<p align="center">
  <i>Built as a reference implementation for Low Level Design interviews.</i><br>
  <i>Star ⭐ this repo if you found it helpful!</i>
</p>
