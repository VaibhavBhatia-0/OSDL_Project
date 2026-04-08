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

| Week | Concept | Exact Location in Code |
|------|---------|------------------------|
| **1** | **Encapsulation** | All fields in `Room`, `Customer`, `Booking`, `RoomServiceRequest` are `private` with public getters/setters |
| **1** | **Inheritance** | `StandardRoom`, `DeluxeRoom`, `SuiteRoom` all `extend Room` |
| **1** | **Polymorphism (Runtime)** | `room.calculateTariff(nights)` in `BookingService.bookRoom()` — the correct override runs based on actual room type |
| **1** | **Abstraction** | `Room` is `abstract`; `calculateTariff()` and `getRoomType()` are abstract methods |
| **2** | **Wrapper Classes** | `Integer.parseInt(roomNumberField.getText())` and `Double.parseDouble(priceField.getText())` in all controllers |
| **2** | **Autoboxing** | `TableColumn<Booking, Double>` — `double` primitive autoboxed into `Double` for JavaFX property binding |
| **2** | **Enum with constructor** | `ServiceType` and `ServiceStatus` enums in `RoomServiceRequest.java` — each constant has a display name and charge value |
| **3** | **Thread (extend Thread / Runnable)** | `AutoSaveThread` — `new Thread(() -> { while(true) { Thread.sleep(60_000); save(); } })` in `Main.java` |
| **3** | **Daemon Thread** | `autoSave.setDaemon(true)` in `Main.java`; `notifThread.setDaemon(true)` in `DashboardController.java` |
| **3** | **sleep()** | `Thread.sleep(60_000)` in AutoSaveThread; `Thread.sleep(2000)` in checkout notification thread |
| **3** | **Platform.runLater()** | `CheckoutNotificationThread` in `DashboardController.java` — safely updates JavaFX UI from background thread |
| **4** | **Synchronized method** | `public synchronized boolean bookRoom(...)` in `BookingService.java` |
| **4** | **Shutdown Hook** | `Runtime.getRuntime().addShutdownHook(new Thread(() -> save()))` in `Main.java` |
| **5** | **FileOutputStream / FileInputStream** | `PersistenceService.java` — wraps these around `ObjectOutputStream`/`ObjectInputStream` |
| **5** | **FileWriter** | `BookingController.java` — bill export writes `.txt` files using `FileWriter` |
| **6** | **Serializable** | `Room`, `Customer`, `Booking`, `RoomServiceRequest` all implement `Serializable` with explicit `serialVersionUID` |
| **6** | **ObjectOutputStream / ObjectInputStream** | `PersistenceService.saveAllData()` and `loadAllData()` — writes/reads `HotelData` wrapper object |
| **7** | **Generic classes** | `ArrayList<Room>`, `HashMap<String, Customer>`, `ObservableList<Booking>`, `Map<String, Double>` throughout all services and controllers |
| **7** | **Generic methods** | Service return types like `public List<Booking> filterByStatus(String status)` and `public List<Room> searchRooms(String query)` |
| **8** | **ArrayList** | `RoomService`, `BookingService`, `RoomServiceService` — all store their data in `ArrayList` |
| **8** | **HashMap** | `CustomerService` uses `HashMap<String, Customer>` for O(1) lookup by customer ID |
| **8** | **Iterator / Stream** | `stream().filter().collect()` in all service methods; `Collectors.groupingBy()` in analytics/statistics calculations |
| **8** | **Collections.sort** | Sorting rooms/bookings in various controller display methods |
| **9** | **JavaFX Controls** | `TableView`, `ComboBox`, `DatePicker`, `RadioButton`, `TextArea`, `BarChart`, `PieChart` in all FXMLs |
| **9** | **Event Handling** | `onAction="#handleAddRoom"` etc. in all FXMLs wired to `@FXML` methods in controllers |
| **9** | **FXML + Scene Builder** | All 6 `.fxml` files define the UI; CSS applied via `scene.getStylesheets().add(...)` in `Main.java` |
| **10** | **Integration** | All concepts above working together in one running application |

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
