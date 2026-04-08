# Hotel Management System — Developer README

## What This Project Is

A desktop hotel management application built in Java 23 + JavaFX 21. It runs standalone — no internet, no database. All data is saved to a binary file on your machine. You interact with it through a tabbed GUI window.

---

## How to Run

```
mvn clean javafx:run
```
First run creates `hotel_data.dat` automatically. Delete it to reset all data.

---

## File-by-File Explanation

### `Main.java`
Entry point. Launches JavaFX, loads saved data from disk on startup, sets the scene to `Dashboard.fxml`, registers a shutdown hook to save on close, and starts the auto-save background thread. Think of it as the "power button" of the app.

---

### Model Layer (`model/`)

**`Room.java`** — Abstract base class for all room types. Holds `roomNumber`, `pricePerNight`, `isBooked` flag. Declares two abstract methods: `calculateTariff(int nights)` and `getRoomType()`. Cannot be instantiated directly.

**`StandardRoom.java`** — Extends `Room`. `calculateTariff` returns `price × nights` (no premium).

**`DeluxeRoom.java`** — Extends `Room`. `calculateTariff` returns `price × nights × 1.1` (10% premium).

**`SuiteRoom.java`** — Extends `Room`. `calculateTariff` returns `price × nights × 1.25` (25% premium).

**`Customer.java`** — Plain data class. Holds `customerId`, `name`, `contact`, `email`. Implements `Serializable` so it can be saved to disk.

**`Booking.java`** — Represents a single room reservation. Holds references to a `Customer` and a `Room`, check-in/check-out dates, `totalCost`, `discountPercent`, `status` (Active/Checked Out/Cancelled), feedback `rating` (1–5 stars), and `feedbackComment`. Has an `applyDiscount()` method that reduces cost by a percentage. Implements `Serializable`.

**`RoomServiceRequest.java`** — Represents a request for an in-room service (food, laundry, spa, etc.). Contains a `ServiceType` enum (with a display name and charge amount) and a `ServiceStatus` enum (PENDING → IN_PROGRESS → COMPLETED / CANCELLED). Linked to a `Booking`. Implements `Serializable`.

---

### Service Layer (`service/`)

**`AppContext.java`** — Static class that holds one instance each of all service objects (`roomService`, `customerService`, `bookingService`, `roomServiceService`, `persistenceService`). Acts as a simple service locator — any controller can call `AppContext.bookingService` to get the shared instance.

**`RoomService.java`** — Manages `ArrayList<Room>`. Methods: `addRoom`, `deleteRoom`, `updatePrice`, `getAllRooms`, `searchRooms`, `getRoomByNumber`. Internally maintains the list; no public access to the raw list.

**`CustomerService.java`** — Manages `HashMap<String, Customer>` (key = customerId) for O(1) lookup. Methods: `addCustomer`, `deleteCustomer`, `updateCustomer`, `getCustomerById`, `getAllCustomers`, `searchCustomers`.

**`BookingService.java`** — Core booking logic. Manages `ArrayList<Booking>`. The `bookRoom()` method is `synchronized` to prevent two threads booking the same room at the same time. Contains `isRoomAvailableForDates()` for overlap detection, `checkoutBooking()`, `cancelBooking()`, and various filters.

**`RoomServiceService.java`** — Manages `ArrayList<RoomServiceRequest>`. Methods to create requests, update status (PENDING → IN_PROGRESS → COMPLETED), cancel, and calculate total charges for a given booking (used in the checkout bill).

**`PersistenceService.java`** — Handles all reading and writing to disk. `saveAllData()` wraps all four lists/maps into a `HotelData` wrapper object and writes it with `ObjectOutputStream` to `hotel_data.dat`. `loadAllData()` reads it back with `ObjectInputStream` and repopulates the service objects.

---

### Controller Layer (`controller/`)

**`DashboardController.java`** — Populates the 8 stat cards (total rooms, revenue, etc.) on load and refresh. Fires the checkout-notification background thread 2 seconds after startup to alert staff of any bookings due today. Has a "Statistics" button that opens a detailed popup with revenue breakdown, rating distribution, and recent comments. Calls `AnalyticsController.refresh()` so charts stay in sync.

**`RoomController.java`** — Handles the Rooms tab. Binds room data to `TableView`. Handles add/update/delete with `AlertHelper` dialogs. Search field filters live. Status filter ComboBox filters by Available/Booked.

**`CustomerController.java`** — Handles the Customers tab. Has real-time text listeners on email and contact fields that show `✓ Valid` or `✗ Invalid` hint labels as you type. "View Booking History" opens a dialog listing all bookings for the selected customer with total spent.

**`BookingController.java`** — The most complex controller. Handles room booking (with date overlap check, promo code validation with live hint, cost preview), checkout (triggers feedback dialog then shows formatted bill), cancellation, and all three export functions (selected bill .txt, all bills .txt, CSV).

**`RoomServiceController.java`** — Handles the Room Services tab. Active bookings ComboBox refreshes on click. Lets staff create service requests and move them through PENDING → IN_PROGRESS → COMPLETED → CANCELLED. Shows live stats (open request count, total accumulated charges).

**`AnalyticsController.java`** — Manages the Analytics tab. Builds 3 JavaFX charts: a grouped BarChart for room availability by type, a PieChart for revenue breakdown, and a status overview BarChart. Uses `Platform.runLater` to force legend colors to match the custom gold/dark palette. Y-axis is integer-only. 4 summary cards show Room Revenue, Service Revenue, Grand Total, and Occupancy Rate %.

---

### Utility (`util/`)

**`AlertHelper.java`** — Replaces all default JavaFX `Alert` dialogs with custom styled ones matching the Noir & Gold theme. Has five kinds: INFO, SUCCESS, WARNING, ERROR, CONFIRM. Each has a colored icon circle, gold/green/red accent, dark background. Usage: `AlertHelper.showAndWait(Kind.SUCCESS, "Title", "Message")` or `AlertHelper.confirm("Title", "Message")` which returns `true`/`false`.

---

### FXML Views (`resources/fxml/`)

**`Dashboard.fxml`** — Two rows of stat cards + a Statistics/Save/Refresh button row. Wrapped in ScrollPane.

**`Rooms.fxml`** — Left panel: form to add/update/delete room. Right panel: searchable, filterable TableView.

**`Customers.fxml`** — Left panel: form with hint labels for email/phone. Right panel: customer TableView with a search bar.

**`Bookings.fxml`** — Left panel (in ScrollPane): booking form with date pickers, promo code field, availability check; below it checkout/cancel/export buttons. Right panel: booking TableView.

**`RoomServices.fxml`** — Left panel: mini stats + new request form (booking selector, service type, instructions) + status management buttons. Right panel: service request log TableView with filter.

**`Analytics.fxml`** — Full-page layout: 4 summary cards on top, then 3 chart areas (availability bar chart, revenue pie chart, status bar chart). All wrapped in ScrollPane.

---

### CSS (`resources/css/style.css`)
Defines the Noir & Gold theme: black background (#0c0c0c), gold accent (#C9A84C), Georgia serif fonts, card styles, button variants (default gold, secondary outline, danger red), TableView row highlighting, chart plot area theming (dark backgrounds, forced bar/pie colors), ScrollPane transparent styling.

---

## Week-by-Week Concept Map

---

### Week 1 — Object-Oriented Programming (Classes, Encapsulation, Inheritance, Polymorphism, Abstraction)

**Classes and Objects**
Every entity in the system is a class. `Room`, `Customer`, `Booking`, `RoomServiceRequest` are all model classes instantiated as objects at runtime. For example, when a receptionist adds a room, a `new StandardRoom(101, 1500)` object is created and stored.

**Encapsulation**
All fields in every model class are `private`. Access is only through public getters/setters.
- `Room.java` — `private int roomNumber`, `private double pricePerNight`, `private boolean booked` — accessed via `getRoomNumber()`, `getPricePerNight()`, `isBooked()` etc.
- Same pattern in `Customer.java`, `Booking.java`, `RoomServiceRequest.java`
- Service classes (`RoomService`, `BookingService`, etc.) expose only defined public methods — the internal `ArrayList` or `HashMap` is never directly accessible.

**Inheritance**
```
Room  (abstract)
 ├── StandardRoom
 ├── DeluxeRoom
 └── SuiteRoom
```
`StandardRoom`, `DeluxeRoom`, and `SuiteRoom` all `extend Room`. They inherit `roomNumber`, `pricePerNight`, `isBooked`, and all getters/setters without rewriting them. Each subclass only adds what is unique to it.

**Polymorphism (Runtime / Method Overriding)**
`calculateTariff(int nights)` is declared abstract in `Room` and overridden in each subclass:
- `StandardRoom.calculateTariff(n)` → `pricePerNight * n`
- `DeluxeRoom.calculateTariff(n)` → `pricePerNight * n * 1.1`
- `SuiteRoom.calculateTariff(n)` → `pricePerNight * n * 1.25`

In `BookingService.bookRoom()`, the call is: `double cost = room.calculateTariff((int) nights);`
Here `room` is a `Room` reference — Java decides at runtime which override to call based on the actual object type. This is runtime polymorphism.

**Method Overloading**
`AlertHelper.java` overloads its dialog methods — `show(Kind, title, msg)` (non-blocking) and `showAndWait(Kind, title, msg)` (blocking) — same name, different behaviour.

**Abstraction**
`Room` is declared `abstract` — you cannot do `new Room()`. It exists only to define the contract. The two abstract methods `calculateTariff()` and `getRoomType()` force every subclass to provide its own implementation. The abstract class hides the "how" and only exposes the "what".

**`this` and `super` keywords**
- `this` is used in all model constructors: `this.roomNumber = roomNumber;`
- `super` is used in subclass constructors: `super(roomNumber, pricePerNight);` in `StandardRoom`, `DeluxeRoom`, `SuiteRoom` to call the `Room` constructor.

---

### Week 2 — Wrapper Classes, Autoboxing, Unboxing, Enumerations

**Wrapper Classes**
Raw text from input fields is always a `String`. Converting to numeric types uses wrapper class static methods:
- `Integer.parseInt(roomNumberField.getText().trim())` — in `RoomController.java`
- `Double.parseDouble(priceField.getText().trim())` — in `RoomController.java`
- `Integer.parseInt(contactField.getText())` — in `CustomerController.java`
These are used in every controller that reads numeric input from a `TextField`.

**Autoboxing**
When storing primitives in generic collections or JavaFX properties, Java silently boxes them:
- `TableColumn<Booking, Double> costCol` — the `double totalCost` field is autoboxed to `Double` for the JavaFX `PropertyValueFactory` to work.
- `Map<String, Double> DISCOUNT_CODES` in `BookingController.java` — storing `0.10`, `0.15`, etc. as `Double` objects autoboxed from `double` literals.

**Unboxing**
When pulling values back out of collections for arithmetic: `double cost = DISCOUNT_CODES.get(code);` — the `Double` object is automatically unboxed to `double` for the multiplication.

**Enumeration (enum)**
`RoomServiceRequest.java` contains two enums:
```java
public enum ServiceType {
    FOOD_DELIVERY("Food Delivery", 250),
    SPA_WELLNESS("Spa & Wellness", 800),
    LAUNDRY("Laundry", 150), ...
}
public enum ServiceStatus { PENDING, IN_PROGRESS, COMPLETED, CANCELLED }
```
**Enum with constructor and methods** — `ServiceType` has a constructor that takes a display name and a charge amount. It has `getDisplayName()` and `getCharge()` methods. This demonstrates enum constructors and enum methods exactly as required by Week 2.

---

### Week 3 — Multithreading Basics (Thread creation, lifecycle, sleep, daemon)

**Thread Creation using `Thread` class (Runnable lambda)**
Two threads are created in `Main.java` and `DashboardController.java` using `new Thread(() -> { ... })` with a lambda (Runnable):

**Thread 1 — AutoSaveThread (`Main.java`)**
```java
Thread autoSave = new Thread(() -> {
    while (true) {
        Thread.sleep(60_000);          // sleep()
        persistenceService.saveAllData(...);
    }
});
autoSave.setDaemon(true);             // daemon thread
autoSave.setName("AutoSaveThread");
autoSave.start();
```
This is a Runnable-based thread. `sleep(60_000)` pauses execution for 60 seconds. Marked daemon so it does not prevent JVM exit.

**Thread 2 — CheckoutNotificationThread (`DashboardController.java`)**
```java
Thread notifThread = new Thread(() -> {
    Thread.sleep(2000);               // sleep() — wait for UI to load
    List<Booking> dueToday = ...stream().filter(...).toList();
    if (!dueToday.isEmpty()) {
        Platform.runLater(() -> AlertHelper.showAndWait(...));
    }
});
notifThread.setDaemon(true);
notifThread.start();
```
Wakes up 2 seconds after app launch, checks if any bookings are due for checkout today, and alerts staff. Uses `Platform.runLater()` — mandatory when updating JavaFX UI from a non-UI thread.

**Daemon Thread**
Both threads are marked `.setDaemon(true)`. A daemon thread automatically dies when the main application thread ends — it never blocks the JVM from shutting down.

**Thread Lifecycle**
Both threads go through: NEW → RUNNABLE (`.start()`) → TIMED_WAITING (`.sleep()`) → RUNNABLE → TERMINATED (when app closes).

---

### Week 4 — Synchronization (synchronized methods, inter-thread safety, shutdown hook)

**Synchronized Method**
`BookingService.java`:
```java
public synchronized boolean bookRoom(Customer customer, Room room,
                                      LocalDate checkIn, LocalDate checkOut) { ... }
```
The `synchronized` keyword ensures that if two operations (e.g., two booking requests arriving simultaneously) try to call `bookRoom()` at the same time, one must wait until the other finishes. This prevents the race condition where two bookings could be created for the same room on the same dates.

**Shutdown Hook (inter-thread communication with JVM)**
`Main.java`:
```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    persistenceService.saveAllData(...);
}));
```
A shutdown hook is a special thread the JVM runs when it is about to exit (window closed, Ctrl+C, etc.). This is the primary save mechanism — guarantees data is never lost on close.

**Why synchronization is needed here**
The auto-save daemon thread and the UI thread both access the same `ArrayList<Booking>` and `HashMap<Customer>`. Without `synchronized`, a thread could read a half-written booking. The `synchronized` on `bookRoom()` creates a lock on the `BookingService` instance, serialising writes.

---

### Week 5 — I/O Streams (Byte Streams, Character Streams, FileInputStream, FileOutputStream, FileReader, FileWriter)

**FileOutputStream / FileInputStream (Byte Streams)**
`PersistenceService.java`:
```java
// Save (byte stream wrapping object stream)
FileOutputStream fos = new FileOutputStream("hotel_data.dat");
ObjectOutputStream oos = new ObjectOutputStream(fos);
oos.writeObject(hotelData);

// Load
FileInputStream fis = new FileInputStream("hotel_data.dat");
ObjectInputStream ois = new ObjectInputStream(fis);
HotelData data = (HotelData) ois.readObject();
```
`FileOutputStream` and `FileInputStream` are byte streams — they write raw binary data. The `.dat` file is not human-readable.

**FileWriter (Character Stream)**
`BookingController.java` — bill export:
```java
FileWriter fw = new FileWriter(file);
fw.write("LUXURY HOTEL — BILL\n");
fw.write("Booking ID : " + booking.getBookingId() + "\n");
// ...
fw.close();
```
`FileWriter` is a character stream — it writes text directly. Used for `.txt` bill exports and `.csv` exports. Character streams handle encoding automatically, unlike byte streams.

---

### Week 6 — Random Access File, Serialization, Deserialization

**Serialization**
All model classes implement `java.io.Serializable`:
```java
public class Booking implements Serializable {
    private static final long serialVersionUID = 4L;
    // all fields...
}
```
Same for `Room`, `Customer`, `RoomServiceRequest`. The `serialVersionUID` is explicit — if the class structure changes, you bump this number to avoid `InvalidClassException` when loading old files.

**ObjectOutputStream (Serialize to file)**
`PersistenceService.saveAllData()`:
```java
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("hotel_data.dat"));
oos.writeObject(hotelData);  // HotelData wraps all 4 lists/maps
oos.close();
```
The entire application state — every room, customer, booking, and service request — is serialized into one binary file.

**ObjectInputStream (Deserialize from file)**
`PersistenceService.loadAllData()`:
```java
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("hotel_data.dat"));
HotelData data = (HotelData) ois.readObject();
ois.close();
```
On app startup, the saved state is deserialized and all service objects are repopulated — the app continues exactly where it left off.

---

### Week 7 — Generics (Generic classes, bounded types, generic methods)

**Generic Classes with Type Parameters**
```java
ArrayList<Room>              // in RoomService.java
ArrayList<Booking>           // in BookingService.java
ArrayList<RoomServiceRequest> // in RoomServiceService.java
HashMap<String, Customer>    // in CustomerService.java
Map<String, Double>          // DISCOUNT_CODES in BookingController.java
ObservableList<Booking>      // in BookingController.java (JavaFX live table binding)
```
The `<T>` type parameter enforces compile-time type safety — you cannot accidentally add a `Customer` into an `ArrayList<Room>`.

**Generic Methods (return types)**
```java
public List<Booking> filterByStatus(String status) { ... }
public List<Booking> getBookingsForCustomer(String customerId) { ... }
public List<Room> searchRooms(String query) { ... }
```
These methods return a generic `List<T>` — the caller gets a typed list without any casting.

**Bounded Types (in JavaFX)**
`TableColumn<Booking, Double>`, `TableColumn<Room, Integer>` — the second type parameter is bounded to the property type, ensuring the column only works with that specific type.

---

### Week 8 — Collections Framework (List, ArrayList, HashMap, Iterator, Stream)

**ArrayList**
- `RoomService.java` → `ArrayList<Room> rooms` — stores all hotel rooms
- `BookingService.java` → `ArrayList<Booking> bookings` — stores all reservations
- `RoomServiceService.java` → `ArrayList<RoomServiceRequest> requests` — stores all service requests

Dynamic sizing — rooms/bookings are added and removed at runtime without pre-declaring array size.

**HashMap**
`CustomerService.java` → `HashMap<String, Customer> customers` — key is `customerId` (String), value is `Customer` object. O(1) lookup: `customers.get("CUST-001")` finds the customer instantly regardless of how many customers exist.

**Iterator**
Used via the enhanced for-loop and Stream API internally when traversing collections — e.g., filtering bookings, searching rooms, computing revenue totals.

**Stream API + Collectors (advanced collection operations)**
```java
// Filter active bookings
bookings.stream()
        .filter(b -> b.getStatus().equals("Active"))
        .collect(Collectors.toList());

// Total revenue
bookings.stream()
        .filter(b -> b.getStatus().equals("Checked Out"))
        .mapToDouble(Booking::getTotalCost)
        .sum();

// Group by room type for analytics
bookings.stream()
        .collect(Collectors.groupingBy(b -> b.getRoom().getRoomType(),
                 Collectors.summingDouble(Booking::getTotalCost)));
```
Used across `BookingService.java`, `DashboardController.java`, and `AnalyticsController.java`.

**Collections utility**
`Collections.sort()` used in controller methods to sort room/booking lists before displaying them in the TableView.

---

### Week 9 — JavaFX GUI Programming (Stage, Scene, Controls, Event Handling, FXML)

**JavaFX Architecture (Stage → Scene → Node)**
`Main.java`:
```java
Parent root = FXMLLoader.load(getClass().getResource("/fxml/Dashboard.fxml"));
Scene scene = new Scene(root, 1200, 720);
scene.getStylesheets().add(getClass().getResource("/css/style.css").toExternalForm());
stage.setTitle("Luxury Hotel Management System");
stage.setScene(scene);
stage.show();
```
`Stage` = the window. `Scene` = the content area. `root` = the top-level layout node loaded from FXML.

**Controls used across all FXMLs**

| Control | Where Used |
|---------|-----------|
| `Label` | Stat cards, form labels, hint labels (✓/✗ validation) |
| `TextField` | All input forms (room number, customer ID, promo code, etc.) |
| `Button` | Every action (Add, Update, Delete, Checkout, Export, etc.) |
| `TableView` | Room list, customer list, booking list, service request log |
| `ComboBox` | Room type selector, status filter, active booking selector |
| `DatePicker` | Check-in and check-out date selection in Bookings tab |
| `RadioButton` | Feedback star rating (1–5) in checkout dialog |
| `TextArea` | Bill display in checkout, special instructions in room service |
| `TabPane` | Main navigation (Dashboard / Rooms / Customers / Bookings / Room Services / Analytics) |
| `BarChart` | Occupancy & status charts in Analytics tab |
| `PieChart` | Revenue breakdown in Analytics tab |
| `ScrollPane` | Wraps every tab root so content never gets cut off |

**Event Handling**
In FXML: `<Button text="Add Room" onAction="#handleAddRoom"/>` — the `#handleAddRoom` wires the button click to the `@FXML void handleAddRoom()` method in the controller.

Real-time listeners (not button clicks):
```java
emailField.textProperty().addListener((obs, oldVal, newVal) -> {
    // validate and update hint label live as user types
});
```
Used in `CustomerController.java` for email and phone validation.

**Scene Builder + External CSS**
All layouts defined in `.fxml` files (edited in Scene Builder). CSS applied globally via `scene.getStylesheets()` — all styling lives in `style.css`, keeping Java code clean.

---

### Week 10 — Integration (Complete Application)

All 9 weeks' concepts work together seamlessly:
- OOP models (Week 1) are serialized to disk (Week 6) via byte streams (Week 5)
- Collections (Week 8) using generics (Week 7) hold all runtime data
- Wrapper types (Week 2) handle field parsing and enum-based service types
- Background threads (Week 3) auto-save data; synchronized methods (Week 4) prevent race conditions
- JavaFX GUI (Week 9) binds live to the collections and triggers all operations
- The result is a single `.jar`-runnable desktop application with persistent data, a professional Noir & Gold theme, real-time validation, analytics charts, room services, multi-format bill export, and zero external dependencies.

---

## Key Business Rules (Quick Reference)

- Room cannot be booked if dates overlap with an existing active booking
- Customer/Room cannot be deleted if they have an active booking
- Check-in cannot be in the past; check-out must be after check-in
- Checkout blocked for completed/cancelled bookings
- Room service requests blocked for checked-out/cancelled bookings
- Cancelled service requests are excluded from the bill

---

## Promo Codes

| Code       | Discount |
|------------|----------|
| WELCOME10  | 10%      |
| DELUXE15   | 15%      |
| VIP20      | 20%      |
| SUMMER5    | 5%       |
| STAFF50    | 50%      |
