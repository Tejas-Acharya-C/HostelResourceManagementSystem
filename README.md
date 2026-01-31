# Hostel Resource Management System

A Java-based console application designed to manage shared hostel resources (e.g., washing machines, study rooms, game consoles). This project utilizes **JDBC** for database connectivity and follows the **DAO (Data Access Object) Design Pattern** to separate business logic from database operations.

## 1. Project Overview

This system allows for the efficient management and booking of resources that support parallel usage (capacity > 1).

### Key Features
* **User Management:** Register students and wardens with specific priority scores.
* **Resource Management:** Create resources with specific capacities (allowing parallel usage).
* **Smart Booking System:** Automatically checks resource availability against capacity before confirming reservations.
* **Transaction Management:** Ensures data integrity (ACID properties) during booking operations.

---

## 2. System Architecture

The application is structured into four distinct layers:

* **Model Layer (`model` package):** Plain Old Java Objects (POJOs) representing database entities.
* **DAO Layer (`dao` package):** Handles low-level SQL queries and database communication.
* **Service Layer (`service` package):** Contains business logic and handles transaction management.
* **Presentation Layer (`main` package):** The Command Line Interface (CLI) for user interaction.

---

## 3. Database Configuration

### Connection Details
* **File:** `src/dao/DBConnection.java`
* **Database Name:** `hostel_resource_db`
* **Driver:** MySQL JDBC Driver

### Schema Requirements
Based on the SQL queries in the DAO files, the database requires the following tables:

| Table Name | Columns | Description |
| :--- | :--- | :--- |
| **users** | `user_id` (PK), `name`, `role`, `priority_score`, `penalty_points` | Stores user profiles and permissions. |
| **resources** | `resource_id` (PK), `resource_name`, `capacity` | Stores resources and their maximum concurrent capacity. |
| **bookings** | `booking_id` (PK), `user_id` (FK), `resource_id` (FK), `start_time`, `end_time`, `status` | Stores reservation logs and status. |

---

## 4. Class Reference

### A. Model Layer
* **`User`**: Represents a system user. Includes a `priorityScore` (50 for Wardens, 10 for Students) used for future conflict resolution logic.
* **`Resource`**: Represents a physical asset. Key field is `capacity`, determining how many users can book it simultaneously.
* **`Booking`**: Represents a transaction. Uses `LocalDateTime` for precise scheduling and includes a status field (e.g., "CONFIRMED").

### B. DAO Layer (Data Access)
* **`DBConnection`**: Uses the Singleton pattern approach to provide a static `getConnection()` method via `DriverManager`.
* **`UserDAO`**: Handles insertion of new users into the database.
* **`ResourceDAO`**:
    * `createResource()`: Adds new resources.
    * `getCapacity(int resourceId)`: **Critical.** Fetches the max capacity for the booking logic.
* **`BookingDAO`**:
    * `countActiveBookings()`: **Core Logic.** Counts how many confirmed bookings overlap with a requested time slot.
    * `createBooking()`: Inserts the booking record.
    * `readConflicts()`: (Legacy/Debug) Returns a list of conflicting booking objects.

### C. Service Layer
* **`BookingService`**: Acts as the transaction manager.
    * **Method:** `bookResource(userId, resourceId, start, end)`
    * **Logic:**
        1.  Disables auto-commit.
        2.  Fetches Resource Capacity.
        3.  Counts Active Bookings for the requested duration.
        4.  **Decision:** If `Active Count < Capacity`, commit booking. Otherwise, reject.
        5.  Commits transaction and closes connection.

### D. Main Layer
* **`MainApp`**:
    * Provides a `Scanner`-based menu loop.
    * Handles input parsing (e.g., converting "hours from now" into `LocalDateTime`).
    * Routes user choices to the appropriate DAO or Service.

---

## 5. Core Business Logic: Capacity Check

The system allows multiple users to book the same resource *at the same time* as long as the resource's capacity is not exceeded.

### The Algorithm
**Location:** `BookingService.java` & `BookingDAO.java`

1.  **Input:** Resource ID, Start Time ($T_{start}$), End Time ($T_{end}$).
2.  **Query:** Count rows in `bookings` where:
    * `resource_id` matches.
    * `status` is 'CONFIRMED'.
    * **Time Overlap Logic:** `start_time <` $T_{end}$ **AND** `end_time >` $T_{start}$
3.  **Validation:**

$$
\text{If } (CurrentActiveBookings < MaxCapacity) \rightarrow \text{Approve}
$$
$$
\text{Else } \rightarrow \text{Reject}
$$

---

## 6. Setup & Execution Guide

### Prerequisites
1.  **Database Setup:** Ensure MySQL is running. Create the database `hostel_resource_db` and the tables listed in Section 3.
2.  **Configuration:** Open `src/dao/DBConnection.java` and update the `USER` and `PASSWORD` constants to match your local MySQL installation.

### Build & Run

**1. Compilation**
```bash
javac -d bin src/model/*.java src/dao/*.java src/service/*.java src/main/*.java
