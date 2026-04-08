# Hotel Management System — Complete Project Documentation

## Overview

A JavaFX-based standalone desktop application for managing a luxury hotel's rooms, guests, bookings, checkout billing, room services, and analytics. Built with Maven, FXML + CSS, and strict MVC architecture. Uses Java Serialization for file-based persistence — no database required.

---

## Tech Stack

| Component     | Technology                        |
|---------------|-----------------------------------|
| Language      | Java 23                           |
| UI Framework  | JavaFX 21                         |
| Build Tool    | Maven                             |
| UI Design     | Scene Builder + External CSS      |
| Persistence   | Java Object Serialization (.dat)  |
| Architecture  | MVC (Model-View-Controller)       |

---

## Project Structure

```
hotel-management/
│
├── src/main/java/com/hotel/
│   ├── model/
│   │   ├── Room.java                  (abstract)
│   │   ├── StandardRoom.java
│   │   ├── DeluxeRoom.java
│   │   ├── SuiteRoom.java
│   │   ├── Customer.java
│   │   ├── Booking.java               (+ discount + feedback)
│   │   └── RoomServiceRequest.java    (NEW)
│   │
│   ├── service/
│   │   ├── AppContext.java
│   │   ├── RoomService.java
│   │   ├── CustomerService.java
│   │   ├── BookingService.java
│   │   ├── RoomServiceService.java    (NEW)
│   │   └── PersistenceService.java
│   │
│   ├── controller/
│   │   ├── DashboardController.java
│   │   ├── RoomController.java
│   │   ├── CustomerController.java
│   │   ├── BookingController.java
│   │   └── RoomServiceController.java (NEW)
│   │
│   ├── util/
│   │   └── AlertHelper.java           (NEW — styled alerts)
│   │
│   └── Main.java
│
├── src/main/resources/
│   ├── fxml/
│   │   ├── Dashboard.fxml
│   │   ├── Rooms.fxml
│   │   ├── Customers.fxml
│   │   ├── Bookings.fxml
│   │   └── RoomServices.fxml          (NEW)
│   └── css/
│       └── style.css
│
└── pom.xml
```

---

## OOP Concepts — Where Each Is Used

### 1. Encapsulation (Week 1)
Every model class uses private fields with public getters/setters.
- `Room`, `Customer`, `Booking`, `RoomServiceRequest` — all fields are private
- Service classes expose only defined public methods; internal lists are private
- `AlertHelper` encapsulates all dialog-building logic behind static methods

### 2. Inheritance (Week 1)
```
Room  (abstract)
 ├── StandardRoom
 ├── DeluxeRoom
 └── SuiteRoom
```
All three concrete room types extend the abstract `Room` class and inherit `roomNumber`, `pricePerNight`, getters/setters.

### 3. Polymorphism (Week 1)
`calculateTariff(int nights)` is abstract in `Room` and overridden differently in each subclass:
- `StandardRoom` → `price × nights` (base rate)
- `DeluxeRoom`   → `price × nights × 1.1` (10% premium)
- `SuiteRoom`    → `price × nights × 1.25` (25% premium)

Called polymorphically in `BookingService.bookRoom()`:
```java
double cost = room.calculateTariff((int) nights);
```

### 4. Abstraction (Week 1)
`Room` is declared `abstract` — it cannot be instantiated directly. `calculateTariff()` and `getRoomType()` are abstract methods that enforce implementation in all subclasses.

### 5. Enums (Week 1 / Week 7)
`RoomServiceRequest.ServiceType` and `ServiceStatus` use Java enums with fields:
```java
public enum ServiceType {
    FOOD_DELIVERY("Food Delivery", 250),
    SPA_WELLNESS("Spa & Wellness", 800), ...
}
```

---

## Collections Used (Week 8)

| Collection       | Location                | Purpose                              |
|------------------|-------------------------|--------------------------------------|
| `ArrayList<Room>`| `RoomService`           | All rooms in memory                  |
| `ArrayList<Booking>` | `BookingService`    | All bookings in memory               |
| `ArrayList<RoomServiceRequest>` | `RoomServiceService` | All service requests    |
| `HashMap<String, Customer>` | `CustomerService` | Fast O(1) lookup by ID         |
| `Map<String, Double>` | `BookingController`| Discount code → percentage map      |
| `stream() + Collectors` | All services    | Filtering, grouping, summing         |

---

## Generics (Week 7)

Used throughout with Java Collections:
```java
ArrayList<Room>          rooms
HashMap<String, Customer> customers
ArrayList<Booking>       bookings
TableView<Booking>       bookingTable
ObservableList<Room>     roomList
Map<String, Double>      DISCOUNT_CODES
```
Also in service return types:
```java
public List<Booking> filterByStatus(String status) { ... }
public List<Room> searchRooms(String query) { ... }
```

---

## Threading Design (Week 3 + Week 4)

### Thread 1 — Auto-Save Daemon (Week 3)
```java
Thread autoSave = new Thread(() -> {
    while (true) {
        Thread.sleep(60_000);
        persistenceService.saveAllData(...);
    }
});
autoSave.setDaemon(true);    // Won't block JVM exit
autoSave.start();
```
Runs every 60 seconds in the background as a safety net save.

### Thread 2 — Shutdown Hook (Week 5)
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    persistenceService.saveAllData(...);
}));
```
**Primary save mechanism.** Fires automatically when the app window is closed. Guarantees data is not lost on exit.

### Thread 3 — Checkout Notification Thread (Week 3)
```java
Thread notifThread = new Thread(() -> {
    Thread.sleep(2000);   // Wait for UI to load
    List<Booking> dueToday = bookingService.getAllBookings()
        .stream()
        .filter(b -> b.getCheckOut().isEqual(LocalDate.now()))
        .toList();
    if (!dueToday.isEmpty()) {
        Platform.runLater(() -> AlertHelper.showAndWait(...));
    }
});
notifThread.setDaemon(true);
notifThread.start();
```
Background thread that alerts staff at startup if any bookings are due for checkout today. Uses `Platform.runLater()` to update JavaFX UI safely from a non-UI thread.

### Thread 4 — Synchronization (Week 4)
```java
public synchronized boolean bookRoom(...) { ... }
```
`bookRoom()` in `BookingService` is `synchronized` — prevents race conditions if two actions try to book the same room simultaneously.

### Summary Table

| Thread Name              | Type    | Purpose                                  |
|--------------------------|---------|------------------------------------------|
| AutoSaveThread           | Daemon  | Save data every 60 seconds               |
| Shutdown Hook Thread     | JVM     | Final save when app window closes        |
| CheckoutNotificationThread | Daemon | Alert on startup if checkouts are due  |
| `synchronized bookRoom()`| Lock    | Prevent double-booking race condition    |

---

## File Persistence (Week 5 + Week 6)

### Mechanism
All data is serialized into a single binary file `hotel_data.dat` using Java's `ObjectOutputStream` / `ObjectInputStream`.

```java
// Save
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("hotel_data.dat"));
oos.writeObject(hotelData);   // HotelData wraps all 4 lists/maps

// Load
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("hotel_data.dat"));
HotelData data = (HotelData) ois.readObject();
```

### What Is Serialized
All model classes implement `Serializable` with explicit `serialVersionUID`:
- `Room` (and subclasses)
- `Customer`
- `Booking` (including feedback + discount fields)
- `RoomServiceRequest`

### Save Triggers

| Trigger               | Mechanism           | Priority  |
|-----------------------|---------------------|-----------|
| App window closed     | Shutdown Hook       | Primary   |
| Every 60 seconds      | Daemon Thread       | Backup    |
| Manual button click   | Direct call         | On-demand |
| App startup           | Auto-load           | Restore   |

---

## Wrapper Classes & Autoboxing (Week 2)

Used throughout billing and data handling:
```java
// Autoboxing: int → Integer, double → Double
TableColumn<Booking, Double> costCol = ...;
costCol.setCellValueFactory(new PropertyValueFactory<>("totalCost"));

// Integer.parseInt(), Double.parseDouble() for field validation
int roomNumber = Integer.parseInt(roomNumberField.getText().trim());
double price   = Double.parseDouble(priceField.getText().trim());

// String.format() with wrapper types in billing
String.format("₹%.2f", b.getTotalCost());  // Double unboxing
```

---

## Core Functionality

### Room Management
- Add rooms as Standard / Deluxe / Suite with a price per night
- Each type uses polymorphic `calculateTariff()` with its own pricing rule
- Update price, delete room (blocked if active booking exists)
- Search by room number or type; filter by Available / Booked status
- TableView shows: Room No. | Type | Price/Night | Status (live)

### Customer Management
- Add customers with ID, name, contact, email
- **Real-time field validation**: email checked for `@domain.tld` pattern; phone checked for 10-digit format
- Visual hint labels (`✓ Valid` / `✗ Invalid`) update as you type
- Update, delete (blocked if active booking), clear form
- Customer table shows a **Bookings count** column
- **View Booking History**: select a customer → see all their bookings, total spent, and any feedback left

### Booking System
- Select customer ID and room number manually; pick dates with DatePicker
- **Check Availability**: instantly verify if a room is free for chosen dates with estimated cost shown
- **Promo / Discount Codes**: enter a code at booking to get a percentage discount:
  - `WELCOME10` → 10% off
  - `DELUXE15`  → 15% off
  - `VIP20`     → 20% off
  - `SUMMER5`   → 5% off
  - `STAFF50`   → 50% off
  - Live validation hint shows as you type
- Overlap detection prevents double-booking the same room for overlapping dates
- Cost is auto-calculated using the room's `calculateTariff()` method
- `bookRoom()` is `synchronized` for thread safety

### Checkout
- Select booking from table → click Checkout
- **Feedback dialog** pops up before the bill:
  - 1–5 star radio buttons (default: 4 stars)
  - Optional text comment
  - "Skip" option
- Rating stored in Booking and displayed as `★★★★☆` in the table
- Formatted bill shown in a styled TextArea dialog
- Bill includes: customer info, room details, nights, price/night, discount (if any), room service charges, grand total

### Cancellation
- Cancel any active booking with a styled confirmation dialog
- Cancelled bookings shown in the table with "Cancelled" status
- Cannot cancel a completed booking

---

## Room Services Module (NEW)

### Concept
Mirrors real-world hotel services. Guests with an **active booking** can request in-room services that are tracked through a lifecycle.

### Available Services
| Service           | Charge       |
|-------------------|--------------|
| Room Cleaning     | Free         |
| Food Delivery     | ₹250         |
| Laundry           | ₹150         |
| Spa & Wellness    | ₹800         |
| Extra Amenities   | ₹100         |
| Maintenance       | Free         |
| Wake-Up Call      | Free         |
| Airport Transfer  | ₹600         |

### Request Lifecycle
```
PENDING  →  IN_PROGRESS  →  COMPLETED
                          ↘  CANCELLED
```
- Staff can move any request through these statuses
- Cancelled requests do not add to the bill
- Completed service charges are **included in the checkout bill** automatically

### Features
- Select an active booking → select service type → optional special instructions
- Mini stats on the panel: open request count + total service charges accumulated
- Filter requests by status; view all requests for a specific booking
- Table shows: Request ID | Booking | Room | Guest | Service | Charge | Status | Time | Instructions

---

## Bill Export

### Export Selected Bill (`.txt`)
Generates a formatted receipt for one completed booking:
```
┌─────────────────────────────────────────────────┐
│              LUXURY HOTEL — BILL                │
└─────────────────────────────────────────────────┘
 Booking ID   : BK-A3F9C2
 Customer     : Ravi Shankar
 ...
 Room Charges : ₹4,400.00
 Room Services: ₹400.00
 ═════════════════════════════════════════════════
 GRAND TOTAL  : ₹4,800.00
```

### Export All Bills Report (`.txt`)
One file with every completed checkout, formatted individually, with a grand total at the bottom.

### Export CSV
Full booking history exported as a `.csv` file including discount %, feedback rating, and comments.

---

## Custom Alert System (`AlertHelper`)

All system `Alert` dialogs have been replaced with a custom `AlertHelper` class that produces styled dialogs matching the Noir & Gold theme.

### Alert Types
| Kind    | Accent Color | Icon | Use Case                     |
|---------|-------------|------|------------------------------|
| INFO    | Gold #C9A84C | i    | General information          |
| SUCCESS | Green #22c55e| ✓    | Action completed successfully|
| WARNING | Amber #f59e0b| !    | Non-blocking warnings        |
| ERROR   | Red #ef4444  | ✕    | Validation failures, errors  |
| CONFIRM | Gold #C9A84C | ?    | Confirmation before action   |

### Usage
```java
AlertHelper.showAndWait(AlertHelper.Kind.SUCCESS, "Booking Confirmed", "BK-XXXXX created.");
AlertHelper.showAndWait(AlertHelper.Kind.ERROR,   "Validation Error",  "Email is invalid.");
boolean ok = AlertHelper.confirm("Delete Room", "This cannot be undone.");
```

---

## Dashboard

### Stat Cards (Live)
- Total Rooms, Available, Occupied, Total Revenue
- Total Customers, Active Bookings, Completed Checkouts, Cancelled Bookings
- Average Guest Rating (from feedback)
- Top Revenue Room Type

### Live BarChart (NEW)
JavaFX `BarChart` updated on every refresh:
- Series 1: Available vs Occupied rooms (green / red bars)
- Series 2: Revenue by room type (₹ / 100 for scale)

### Statistics Panel
Click "Statistics" for a detailed popup:
- Revenue breakdown by room type
- Room service total charges + grand combined revenue
- Feedback distribution (5★ to 1★ counts)
- Average rating with star display
- Recent guest comments (up to 5)

---

## Input Validation Summary

| Field          | Rule                                       | Feedback              |
|----------------|--------------------------------------------|-----------------------|
| Email          | Must match `user@domain.tld` regex         | Live hint label       |
| Contact        | 10 digits, optional `+91` prefix           | Live hint label       |
| Room Number    | Positive integer, must be unique           | Error on submit       |
| Price          | Must be > 0                                | Error on submit       |
| Check-Out Date | Must be strictly after Check-In            | Error on submit       |
| Check-In Date  | Cannot be in the past                      | Error on submit       |
| Customer ID    | Must exist before booking                  | Error on submit       |
| Promo Code     | Must match known codes or be blank         | Live hint label       |

---

## Lab Concept Mapping

| Week | Concept                  | Where Demonstrated                                                     |
|------|--------------------------|------------------------------------------------------------------------|
| 1    | OOP                      | `Room` (abstract), `StandardRoom`/`DeluxeRoom`/`SuiteRoom` (inheritance + polymorphism), all model classes (encapsulation) |
| 2    | Wrapper Classes, Autoboxing | `Integer.parseInt()` and `Double.parseDouble()` for field parsing; autoboxing in `TableColumn<Booking, Double>` and billing calculations |
| 3    | Multithreading            | `AutoSaveThread` (daemon, 60s interval), `CheckoutNotificationThread` (daemon, startup alert with `Platform.runLater()`) |
| 4    | Synchronization           | `synchronized bookRoom()` in `BookingService` prevents double-booking race condition |
| 5    | File Streams              | `FileOutputStream` / `FileInputStream` in `PersistenceService` wrapping the object streams |
| 6    | Serialization             | All model classes implement `Serializable` with `serialVersionUID`; full state saved to `hotel_data.dat` via `ObjectOutputStream` |
| 7    | Generics                  | `ArrayList<Room>`, `List<Booking>`, `Map<String, Customer>`, `Map<String, Double>` (discount codes), `ObservableList<T>` in all controllers |
| 8    | Collections Framework     | `ArrayList` in `BookingService` and `RoomService`; `HashMap` in `CustomerService`; `stream()` with `Collectors.groupingBy()`, `filter()`, `mapToDouble()` throughout |
| 9    | JavaFX + Scene Builder    | FXML for all views, CSS for theming, `TableView`, `ComboBox`, `DatePicker`, `TabPane`, `BarChart`, `Dialog`, `RadioButton`, `TextArea` all used |
| 10   | Integration               | All above concepts combined in a single working, themed, persistent desktop application |

---

## Business Rules Enforced

- A room cannot be booked if it already has an active (non-cancelled, non-checked-out) booking that overlaps the requested dates
- A customer cannot be deleted if they have an active booking
- A room cannot be deleted if it has an active booking
- Check-in date cannot be in the past
- Check-out must be strictly after check-in
- Checkout is blocked for already-completed or cancelled bookings
- Cancellation is blocked for already-completed bookings
- Room service requests cannot be placed for checked-out or cancelled bookings

---

## Available Promo Codes

| Code       | Discount |
|------------|----------|
| WELCOME10  | 10%      |
| DELUXE15   | 15%      |
| VIP20      | 20%      |
| SUMMER5    | 5%       |
| STAFF50    | 50%      |

---

## Files Changed in This Version

| File                        | Status  | Change Summary                                        |
|-----------------------------|---------|-------------------------------------------------------|
| `Booking.java`              | Updated | Added `discountPercent`, `applyDiscount()`, feedback  |
| `RoomServiceRequest.java`   | NEW     | Full model for room service requests                  |
| `RoomServiceService.java`   | NEW     | Service layer for room service CRUD + charge totals   |
| `AppContext.java`           | Updated | Added `roomServiceService`                            |
| `PersistenceService.java`   | Updated | Now serializes/deserializes `RoomServiceRequest` list |
| `BookingService.java`       | Updated | Added `isRoomAvailableForDates()`                     |
| `RoomServiceController.java`| NEW     | Full controller for room services tab                 |
| `DashboardController.java`  | Updated | BarChart, checkout-due thread, statistics panel       |
| `BookingController.java`    | Updated | Discount codes, `AlertHelper` throughout, bill export |
| `CustomerController.java`   | Updated | Real-time validation, `AlertHelper`, history viewer   |
| `RoomController.java`       | Updated | `AlertHelper` throughout                              |
| `AlertHelper.java`          | NEW     | Themed dialog utility replacing all system Alerts     |
| `Main.java`                 | Updated | Updated persistence call signatures                   |
| `Dashboard.fxml`            | Updated | BarChart node, Room Services tab, extra stat cards    |
| `Bookings.fxml`             | Updated | Discount code field + export buttons                  |
| `Customers.fxml`            | Updated | Hint labels, View History button                      |
| `RoomServices.fxml`         | NEW     | Full room services UI                                 |
