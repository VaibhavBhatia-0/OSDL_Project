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
│   │   ├── Booking.java               (discount + feedback fields)
│   │   └── RoomServiceRequest.java
│   │
│   ├── service/
│   │   ├── AppContext.java
│   │   ├── RoomService.java
│   │   ├── CustomerService.java
│   │   ├── BookingService.java
│   │   ├── RoomServiceService.java
│   │   └── PersistenceService.java
│   │
│   ├── controller/
│   │   ├── DashboardController.java
│   │   ├── RoomController.java
│   │   ├── CustomerController.java
│   │   ├── BookingController.java
│   │   ├── RoomServiceController.java
│   │   └── AnalyticsController.java
│   │
│   ├── util/
│   │   └── AlertHelper.java
│   │
│   └── Main.java
│
├── src/main/resources/
│   ├── fxml/
│   │   ├── Dashboard.fxml
│   │   ├── Rooms.fxml
│   │   ├── Customers.fxml
│   │   ├── Bookings.fxml
│   │   ├── RoomServices.fxml
│   │   └── Analytics.fxml
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

### 5. Enums (Week 2 / Week 7)
`RoomServiceRequest.ServiceType` and `ServiceStatus` use Java enums with fields and constructors:
```java
public enum ServiceType {
    FOOD_DELIVERY("Food Delivery", 250),
    SPA_WELLNESS("Spa & Wellness", 800), ...
}
```

---

## Collections Used (Week 8)

| Collection                        | Location             | Purpose                              |
|-----------------------------------|----------------------|--------------------------------------|
| `ArrayList<Room>`                 | `RoomService`        | All rooms in memory                  |
| `ArrayList<Booking>`              | `BookingService`     | All bookings in memory               |
| `ArrayList<RoomServiceRequest>`   | `RoomServiceService` | All service requests                 |
| `HashMap<String, Customer>`       | `CustomerService`    | Fast O(1) lookup by customer ID      |
| `Map<String, Double>`             | `BookingController`  | Discount code → percentage map       |
| `stream() + Collectors`           | All services         | Filtering, grouping, summing         |

---

## Generics (Week 7)

Used throughout with Java Collections:
```java
ArrayList<Room>            rooms
HashMap<String, Customer>  customers
ArrayList<Booking>         bookings
TableView<Booking>         bookingTable
ObservableList<Room>       roomList
Map<String, Double>        DISCOUNT_CODES
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
autoSave.setDaemon(true);
autoSave.start();
```
Runs every 60 seconds in the background. Marked daemon so it does not block JVM exit.

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
    Thread.sleep(2000);
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

| Thread Name                  | Type    | Purpose                                  |
|------------------------------|---------|------------------------------------------|
| AutoSaveThread               | Daemon  | Save data every 60 seconds               |
| Shutdown Hook Thread         | JVM     | Final save when app window closes        |
| CheckoutNotificationThread   | Daemon  | Alert on startup if checkouts are due    |
| `synchronized bookRoom()`    | Lock    | Prevent double-booking race condition    |

---

## File Persistence (Week 5 + Week 6)

### Mechanism
All data is serialized into a single binary file `hotel_data.dat` using Java's `ObjectOutputStream` / `ObjectInputStream`.

```java
// Save
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("hotel_data.dat"));
oos.writeObject(hotelData);

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

### Customer Management
- Add customers with ID, name, contact, email
- **Real-time field validation**: email checked for `@domain.tld` pattern; phone checked for 10-digit format
- Visual hint labels (`✓ Valid` / `✗ Invalid`) update as you type
- Update, delete (blocked if active booking), clear form
- Customer table shows a **Bookings count** column
- **View Booking History**: select a customer → see all bookings, total spent, and any feedback left

### Booking System
- Select customer ID and room number; pick dates with DatePicker
- **Check Availability**: instantly verify if a room is free with estimated cost shown
- **Promo / Discount Codes** with live validation hint:
  - `WELCOME10` → 10% off | `DELUXE15` → 15% off | `VIP20` → 20% off | `SUMMER5` → 5% off | `STAFF50` → 50% off
- Overlap detection prevents double-booking
- Cost auto-calculated using polymorphic `calculateTariff()`
- `bookRoom()` is `synchronized` for thread safety

### Checkout
- **Feedback dialog** before the bill: 1–5 star rating + optional text comment
- Rating stored in Booking and displayed as `★★★★☆` in the table
- Bill includes: customer info, room details, nights, price/night, discount, room service charges, grand total

### Cancellation
- Cancel any active booking with a styled confirmation dialog
- Cannot cancel a completed booking

---

## Room Services Module

### Available Services
| Service           | Charge   | Service           | Charge   |
|-------------------|----------|-------------------|----------|
| Room Cleaning     | Free     | Extra Amenities   | ₹100     |
| Food Delivery     | ₹250     | Maintenance       | Free     |
| Laundry           | ₹150     | Wake-Up Call      | Free     |
| Spa & Wellness    | ₹800     | Airport Transfer  | ₹600     |

### Request Lifecycle
```
PENDING  →  IN_PROGRESS  →  COMPLETED
                          ↘  CANCELLED
```
- Cancelled requests do not add to the bill
- Completed service charges are included in the checkout bill automatically

---

## Bill Export
- **Export Selected Bill (.txt)** — formatted receipt for one completed booking
- **Export All Bills Report (.txt)** — all completed checkouts in one file with grand total
- **Export CSV** — full booking history with discount %, feedback rating, and comments

---

## Analytics Tab (NEW)

A dedicated full-page tab with 3 charts + 4 summary cards:
- **Room Availability by Type** — grouped bar chart (Available vs Occupied) broken down by Standard / Deluxe / Suite
- **Revenue Breakdown** — pie chart with distinct colors (Gold / Blue / Green / Red) per room type + room services
- **Booking Status Overview** — side-by-side bars for booking statuses and room service statuses
- **Summary Cards**: Room Revenue | Service Revenue | Grand Total | Occupancy Rate %

Charts refresh automatically when data changes. Y-axis integer-only (no decimal counts). Legend colors forced to match custom palette via `Platform.runLater`.

---

## Custom Alert System (`AlertHelper`)

All system `Alert` dialogs replaced with styled dialogs matching the Noir & Gold theme.

| Kind    | Accent Color  | Icon | Use Case                     |
|---------|---------------|------|------------------------------|
| INFO    | Gold #C9A84C  | i    | General information          |
| SUCCESS | Green #22c55e | ✓    | Action completed successfully|
| WARNING | Amber #f59e0b | !    | Non-blocking warnings        |
| ERROR   | Red #ef4444   | ✕    | Validation failures, errors  |
| CONFIRM | Gold #C9A84C  | ?    | Confirmation before action   |

---

## Dashboard

### Stat Cards (Live)
Total Rooms | Available | Occupied | Total Revenue | Total Customers | Active Bookings | Completed Checkouts | Cancelled Bookings | Avg. Guest Rating | Top Revenue Room Type

### Statistics Panel
Click "Statistics" for detailed popup: revenue breakdown by room type, room service totals, feedback distribution (5★ to 1★), average rating, recent guest comments.

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

## Scrolling & Responsiveness

All FXML root layouts are wrapped in a `ScrollPane` with `fitToWidth="true"`. Every page automatically shows a vertical scrollbar if the window is too small — no buttons or charts are ever cut off.

---

## Lab Concept Mapping

| Week | Concept                     | Where Demonstrated                                                                                  |
|------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| 1    | OOP                         | `Room` (abstract), `StandardRoom`/`DeluxeRoom`/`SuiteRoom` (inheritance + polymorphism), all model classes (encapsulation) |
| 2    | Wrapper Classes, Autoboxing | `Integer.parseInt()` and `Double.parseDouble()` for field parsing; autoboxing in `TableColumn<Booking, Double>` and billing; enum constructors in `ServiceType` |
| 3    | Multithreading               | `AutoSaveThread` (daemon, 60s interval), `CheckoutNotificationThread` (daemon, startup alert with `Platform.runLater()`) |
| 4    | Synchronization              | `synchronized bookRoom()` in `BookingService` prevents double-booking race condition               |
| 5    | File Streams                 | `FileOutputStream` / `FileInputStream` in `PersistenceService`; `FileWriter` for bill export (.txt) |
| 6    | Serialization                | All model classes implement `Serializable` with `serialVersionUID`; full state saved to `hotel_data.dat` via `ObjectOutputStream` |
| 7    | Generics                     | `ArrayList<Room>`, `List<Booking>`, `Map<String, Customer>`, `Map<String, Double>` (discount codes), `ObservableList<T>` in all controllers |
| 8    | Collections Framework        | `ArrayList` in `BookingService`/`RoomService`; `HashMap` in `CustomerService`; `stream()` with `Collectors.groupingBy()`, `filter()`, `mapToDouble()` throughout |
| 9    | JavaFX + Scene Builder       | FXML for all views, CSS for theming, `TableView`, `ComboBox`, `DatePicker`, `TabPane`, `BarChart`, `PieChart`, `Dialog`, `RadioButton`, `TextArea` all used |
| 10   | Integration                  | All above concepts combined in a single working, themed, persistent desktop application            |

---

## Business Rules Enforced

- A room cannot be booked if it has an active overlapping booking
- A customer/room cannot be deleted if they have an active booking
- Check-in date cannot be in the past; check-out must be strictly after check-in
- Checkout blocked for already-completed or cancelled bookings
- Cancellation blocked for already-completed bookings
- Room service requests cannot be placed for checked-out or cancelled bookings

---

## Files Summary

| File                         | Status  | Role                                                      |
|------------------------------|---------|-----------------------------------------------------------|
| `Booking.java`               | Updated | Added `discountPercent`, `applyDiscount()`, feedback      |
| `RoomServiceRequest.java`    | New     | Full model for room service requests with enums           |
| `RoomServiceService.java`    | New     | Service layer for room service CRUD + charge totals       |
| `AppContext.java`            | Updated | Added `roomServiceService`                                |
| `PersistenceService.java`    | Updated | Serializes/deserializes `RoomServiceRequest` list         |
| `BookingService.java`        | Updated | Added `isRoomAvailableForDates()`, synchronized bookRoom  |
| `RoomServiceController.java` | New     | Full controller for room services tab                     |
| `AnalyticsController.java`   | New     | 3-chart analytics tab (bar, pie, status charts)           |
| `DashboardController.java`   | Updated | Checkout-due thread, statistics panel, calls analytics    |
| `BookingController.java`     | Updated | Discount codes, AlertHelper, bill export (.txt + .csv)    |
| `CustomerController.java`    | Updated | Real-time validation, AlertHelper, history viewer         |
| `RoomController.java`        | Updated | AlertHelper throughout                                    |
| `AlertHelper.java`           | New     | Themed dialog utility replacing all system Alerts         |
| `Main.java`                  | Updated | Updated persistence call signatures, named threads        |
| `Dashboard.fxml`             | Updated | ScrollPane wrap, clean stat cards, Statistics button      |
| `Bookings.fxml`              | Updated | ScrollPane wrap, discount code field, all export buttons  |
| `Customers.fxml`             | Updated | ScrollPane wrap, hint labels, View History button         |
| `Rooms.fxml`                 | Updated | ScrollPane wrap                                           |
| `RoomServices.fxml`          | New     | Full room services UI with ScrollPane                     |
| `Analytics.fxml`             | New     | Dedicated analytics tab with 3 charts + summary cards     |
| `style.css`                  | Updated | Chart theming, bar/pie colors, scroll pane transparency   |
