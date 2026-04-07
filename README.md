# Hotel Management System – Product Definition

## Overview
JavaFX-based standalone desktop application for managing hotel rooms, customers, bookings, checkout, and billing using file-based persistence (serialization). Built with Maven, FXML + CSS, and MVC architecture.

---

## Objectives
- Implement OOP concepts (Encapsulation, Inheritance, Polymorphism, Abstraction)
- Build modular JavaFX GUI using FXML + Scene Builder + CSS
- Use file handling (Serialization) for data persistence
- Follow MVC architecture for clean separation of concerns
- Demonstrate all lab concepts (Week 1–10) in a single integrated application

---

## Architecture
**Pattern: MVC**

- **Model** → AbstractRoom, StandardRoom, DeluxeRoom, SuiteRoom, Customer, Booking
- **Service** → RoomService, CustomerService, BookingService, PersistenceService
- **Controller** → DashboardController, RoomController, CustomerController, BookingController
- **UI** → JavaFX (FXML + CSS + Scene Builder)

---

## Core Functionalities (5 Marks)

### Room Management
- Add room (`room_number`, `room_type`, `price_per_night`)
- View all rooms in a TableView
- Filter and display only available rooms

### Customer Management
- Add customer (`customer_id`, `name`, `contact`)
- View all customers in a TableView
- Remove customer (only if no active booking)

### Booking System
- Book a room (select customer + room + check-in/check-out dates via DatePicker)
- Prevent double booking (room must be available)
- Auto cost calculation (`nights × price_per_night`)
- `book_room()` is `synchronized` to prevent race conditions

### Checkout
- Checkout a booked room
- Display bill summary on checkout via Alert dialog
- Update room availability automatically
- Maintain booking history

---

## Additional Functionalities (5 Marks)

### 1. File Persistence (Week 5 + 6)
- All data saved to a single `.dat` file using Java Serialization (`ObjectOutputStream` / `ObjectInputStream`)
- **Primary save mechanism: Shutdown Hook**
  - `Runtime.getRuntime().addShutdownHook()` triggers save automatically when the app window is closed
  - On next launch, data is loaded from the `.dat` file automatically
  - This directly handles the demo scenario: close app → reopen → data intact
- **Secondary save: Background Auto-Save Thread (Bonus)**
  - Daemon thread runs every 60 seconds as a safety net
  - Does NOT replace the shutdown hook — just an extra backup
  - Uses `Thread.sleep(60000)` in a loop; marked as daemon so it doesn't block JVM exit
- Manual save button also provided in the UI

### 2. GUI Design with Styles and Layouts (Week 9)
- Tab-based interface: Dashboard | Rooms | Customers | Bookings
- Dark theme applied globally via external CSS file
- Card-style components on Dashboard
- GridPane for all input forms
- VBox / HBox for navigation and button rows
- TableView for all data listings
- Alerts (success / error) for all user actions
- Input fields cleared after successful operations

### 3. Maven Integration
- Standard Maven project structure
- JavaFX dependencies managed via `pom.xml`
- `javafx-maven-plugin` for running and packaging
- Clean separation of `src/main/java` and `src/main/resources`

### 4. Billing Management (Week 2 + 6)
- Total cost calculated per booking: `nights × price_per_night`
- Bill displayed as Alert dialog on checkout with:
  - Room Number and Type
  - Check-in / Check-out dates
  - Number of nights
  - Price per night
  - **Total cost**
- Booking history table shows all past bills
- Dashboard shows **Total Revenue** (sum of all completed bookings)

### 5. Scene Builder + Various Components (Week 9)
- All FXML files designed using Scene Builder
- UI components used:
  - `Label`, `TextField`, `Button` (basic)
  - `ComboBox` (room type selection)
  - `TableView` + `TableColumn` (data display)
  - `DatePicker` (check-in / check-out dates)
  - `TabPane` + `Tab` (navigation)
  - `Alert` (confirmations and errors)

---

## Data Persistence – Detailed Behavior

### Save Strategy
| Trigger | Mechanism | Purpose |
|---|---|---|
| App window closed | Shutdown Hook | Primary – guaranteed save on exit |
| Every 60 seconds | Daemon Thread | Backup – protects against crashes |
| Manual button click | Direct call | User-triggered save |
| App startup | Auto-load | Restores previous state |

### Shutdown Hook Implementation Note
```
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    PersistenceService.saveAll(rooms, customers, bookings);
}));
```
Registered once in `Main.java` at startup.

### Demo Scenario
- Run app → add rooms/customers/bookings
- Close the window (X button)
- Shutdown hook fires → data saved to `hotel_data.dat`
- Reopen app → data loaded → all records visible

### Persistence Guarantee
- Data survives application restart and system reboot
- Single file stores: `ArrayList<Room>`, `ArrayList<Customer>`, `ArrayList<Booking>`

---

## OOP Concepts Integration

| Concept | Implementation |
|---|---|
| Encapsulation | Private fields + getters/setters in all model classes |
| Inheritance | `AbstractRoom` → `StandardRoom`, `DeluxeRoom`, `SuiteRoom` |
| Polymorphism | `calculateTariff(int nights)` overridden per room type |
| Abstraction | `abstract class AbstractRoom` with abstract `calculateTariff()` |
| Collections | `ArrayList`, `HashMap` in service classes |
| Generics | `ArrayList<T>`, optional `Pair<T,U>` for booking records |

---

## Threading Design

### Booking Synchronization (Week 4)
- `book_room()` method in `BookingService` is `synchronized`
- Prevents race conditions if multiple actions are triggered simultaneously
- Ensures no double booking at the data level
- **Keep this simple** — synchronized method only, no complex background booking thread

### Auto-Save Daemon Thread (Week 3)
- Created as a daemon thread (`setDaemon(true)`)
- Runs in background every 60 seconds
- Uses `Thread.sleep(60000)`
- Updates UI counters using `Platform.runLater()` if needed
- Marked daemon so JVM exit is not blocked

### Why No Complex Threading for Booking
- A background booking thread with `Platform.runLater()` introduces timing bugs that are hard to debug during a demo
- The synchronized method alone satisfies Week 4 (Synchronization) requirements cleanly

---

## User Interface Design

### Navigation
Tab-based interface with 4 tabs:
1. **Dashboard** – System overview
2. **Rooms** – Room management
3. **Customers** – Customer management
4. **Bookings** – Booking and checkout

### Dashboard Tab
- Cards showing:
  - Total Rooms
  - Available Rooms
  - Occupied Rooms
  - Total Revenue
- Updates dynamically after every booking and checkout

### Rooms Tab
- Input form (GridPane): Room Number, Room Type (ComboBox), Price per Night
- Add Room button
- TableView: Room Number | Type | Price | Status (Available / Booked)
- Filter button: Show Available Rooms Only

### Customers Tab
- Input form (GridPane): Customer ID, Name, Contact Number
- Add / Remove Customer buttons
- TableView: Customer ID | Name | Contact

### Bookings Tab
- Booking form: Select Customer (ComboBox), Select Room (ComboBox), Check-in Date (DatePicker), Check-out Date (DatePicker)
- Book Room button
- Checkout button (select from active bookings)
- Booking History TableView: Booking ID | Customer | Room | Dates | Cost | Status

### UI Style (CSS)
- Dark theme (CSS applied globally via `scene.getStylesheets()`)
- Consistent spacing and padding
- Color-coded status: green = Available, red = Booked
- Card components on Dashboard using styled `VBox`

---

## FXML Controller Binding

### Rules
- Each `.fxml` file declares its controller: `fx:controller="com.hotel.controller.XController"`
- All UI elements use matching `fx:id` values
- All button actions bound via `onAction="#handleMethodName"`
- `@FXML` fields initialized before use (guaranteed by `FXMLLoader`)
- No `NullPointerException` risk if `fx:id` names match exactly between FXML and controller

### Example Pattern
```
@FXML private TextField roomNumberField;
@FXML private ComboBox<String> roomTypeCombo;
@FXML private TableView<Room> roomTable;

@FXML
private void handleAddRoom() { ... }
```

---

## Lab Concept Mapping

| Week | Concept | Where Used |
|---|---|---|
| Week 1 | OOP | Model classes: inheritance, polymorphism, encapsulation |
| Week 2 | Wrapper Classes, Autoboxing | Billing calculations using `Integer`, `Double` |
| Week 3 | Multithreading | Auto-save daemon thread |
| Week 4 | Synchronization | `synchronized book_room()` in BookingService |
| Week 5 | File Streams | `FileOutputStream` / `FileInputStream` in PersistenceService |
| Week 6 | Serialization | Object serialization for `.dat` file storage |
| Week 7 | Generics | `ArrayList<Room>`, `ArrayList<Booking>`, `Pair<T,U>` |
| Week 8 | Collections | `ArrayList`, `HashMap` for in-memory data management |
| Week 9 | JavaFX | FXML, Scene Builder, event handling, all UI components |
| Week 10 | Integration | Complete system combining all above |

---

## Constraints

### Functional
- No database — strictly file-based persistence
- Standalone desktop JavaFX application only
- Must support: add room, view rooms, add customer, book room, checkout, view history

### Data Validation
- `room_number` → unique, positive integer
- `customer_id` → unique, non-empty string
- `price_per_night` → must be > 0
- `check_out_date` → must be after `check_in_date`
- No empty fields allowed (validated before any operation)

### Business Rules
- Cannot book a room that is already booked
- Cannot remove a customer with an active booking
- Cannot checkout a room that is already available
- Booking cost = `nights × price_per_night` (calculated automatically)

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Java 23 |
| UI Framework | JavaFX 21 |
| Build Tool | Maven (latest stable) |
| UI Design | Scene Builder + CSS |
| Persistence | Java Serialization (.dat file) |
| Architecture | MVC |

---

## Project Structure (Maven)

```
hotel-management/
│
├── src/main/java/com/hotel/
│   ├── model/
│   │   ├── AbstractRoom.java
│   │   ├── StandardRoom.java
│   │   ├── DeluxeRoom.java
│   │   ├── SuiteRoom.java
│   │   ├── Customer.java
│   │   └── Booking.java
│   ├── service/
│   │   ├── RoomService.java
│   │   ├── CustomerService.java
│   │   ├── BookingService.java
│   │   └── PersistenceService.java
│   ├── controller/
│   │   ├── DashboardController.java
│   │   ├── RoomController.java
│   │   ├── CustomerController.java
│   │   └── BookingController.java
│   └── Main.java
│
├── src/main/resources/
│   ├── fxml/
│   │   ├── Dashboard.fxml
│   │   ├── Rooms.fxml
│   │   ├── Customers.fxml
│   │   └── Bookings.fxml
│   └── css/
│       └── style.css
│
└── pom.xml
```

---

## Change Log

### v2.0
- Replaced auto-save timer thread as primary save with **Shutdown Hook** as primary mechanism
- Kept daemon thread as secondary backup only
- Removed complex background booking thread (replaced with simple `synchronized` method)
- Added explicit `DatePicker` usage for booking dates
- Added persistence demo scenario documentation
- Clarified threading design rationale
- Added complete Lab Concept Mapping table
- Updated project structure with all class names
