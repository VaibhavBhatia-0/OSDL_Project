# Hotel Management System – Product Definition

## Overview
JavaFX-based standalone desktop application for managing hotel rooms, customers, bookings, checkout, and billing using file-based persistence.

---

## Objectives
- Implement OOP concepts (Encapsulation, Inheritance, Polymorphism)
- Build modular JavaFX GUI (FXML + CSS)
- Use file handling (Serialization) for persistence
- Follow MVC architecture

---

## Architecture
Pattern: MVC

- Model → Room, Customer, Booking
- Service → RoomService, CustomerService, BookingService
- UI → JavaFX (FXML + Controllers)

---

## Core Functionalities (5 Marks)

### Room Management
- Add room (room_number, room_type, price_per_night)
- View all rooms (TableView)
- Filter available rooms

### Customer Management
- Add customer (customer_id, name, contact)
- View customers
- Remove customer (only if not booked)

### Booking System
- Book room (customer + room + dates)
- Prevent double booking
- Auto cost calculation (nights × price)

### Checkout
- Checkout room
- Update availability
- Maintain booking history

---

## Additional Functionalities (5 Marks)

### File Persistence
- Save all data → `.dat` file (Serialization)
- Load data on startup/manual trigger
- Single file for rooms + customers + bookings

### GUI Enhancements
- Tab-based UI (Dashboard, Rooms, Customers, Bookings)
- Alerts (success/error)
- Input validation messages
- Styled UI using CSS

### Billing
- Total cost per booking
- Display booking history with cost
- Total revenue calculation

### Scene Builder (FXML)
- Layouts → VBox, HBox, GridPane
- Controller binding via @FXML

### Maven Integration
- Dependency management (JavaFX plugins)
- Standard project structure

---

## UX & Functional Enhancements

### Booking Lifecycle Clarity
- Room status transitions:
  - Available → Booked → Available
- Display status in UI (TableView column)
- Auto-update status on booking and checkout

### Bill Display on Checkout
- On checkout, display:
  - number_of_nights
  - price_per_night
  - total_cost
- Show using Alert dialog
- Uses calculateTariff(int nights)

### Status Feedback System
- Clear user messages:
  - "Room booked successfully"
  - "Room checked out successfully"
  - Error messages for invalid operations

---

## Data Persistence Behavior

- System saves all data in a `.dat` file using serialization
- Data is automatically loaded on application startup
- Supports:
  - Manual save option
  - Auto-save (background thread)

### Persistence Guarantee
- Data remains intact after:
  - Application restart
  - System reboot

---

## Dashboard Functionality

- Displays:
  - total_rooms
  - available_rooms
  - occupied_rooms
  - total_revenue
- Updates dynamically after booking and checkout

---

## User Interface Design

### Navigation
- Tab-based interface:
  - Dashboard
  - Rooms
  - Customers
  - Bookings

### UI Style
- Dark theme (CSS-based)
- Clean layout with proper spacing
- Card-style components for dashboard

### Dashboard
- Displays system overview using cards:
  - Total Rooms
  - Available Rooms
  - Occupied Rooms
  - Total Revenue

### Rooms Screen
- Input form (room_number, type, price)
- TableView showing room details
- Status column (Available / Booked)

### Customers Screen
- Input form (customer_id, name, contact)
- TableView displaying customer details

### Bookings Screen
- Booking form (customer, room, dates)
- Checkout functionality
- Booking history table

### UI Features
- Alerts for success/error messages
- Dynamic updates (booking/checkout)
- Simple, clean, and user-friendly design

---

## FXML Controller Binding (Critical)

- Each `.fxml` file must have a properly defined controller:
  - fx:controller="com.hotel.controller.YourController"

- All UI elements must match controller variables:
  - @FXML private TextField roomNumberField;

- Method bindings must be correct:
  - onAction="#handleAddRoom"

### Rules
- Field names in FXML and controller must match exactly
- All @FXML fields must be initialized before use
- Avoid NullPointerException by:
  - Correct fx:id mapping
  - Proper controller linking
  - Loading FXML using FXMLLoader correctly

---

## Modules

### Dashboard
- total_rooms
- available_rooms
- occupied_rooms
- total_revenue

### Room Module
- add_room()
- get_all_rooms()
- get_available_rooms()

### Customer Module
- add_customer()
- remove_customer()
- find_customer()

### Booking Module
- book_room()
- checkout_room()
- get_booking_history()

---

## Data Storage
- ObjectOutputStream / ObjectInputStream
- Stored:
  - ArrayList<Room>
  - ArrayList<Customer>
  - ArrayList<Booking>

---

## OOP Concepts
- Encapsulation → private fields + getters/setters
- Inheritance → AbstractRoom → StandardRoom / DeluxeRoom / SuiteRoom
- Polymorphism → calculateTariff(int nights)
- Abstraction → abstract Room
- Collections → ArrayList, HashMap
- Generics → optional Pair<T,U>

---

## Lab Concept Integration

### Week 1 – OOP
- Inheritance, Polymorphism, Encapsulation implemented in model classes

### Week 2 – Wrapper Classes
- Integer, Double used in billing and collections
- Autoboxing in calculations

### Week 3 – Multithreading
- Background booking thread
- Thread.sleep() used for simulation
- UI remains responsive

### Week 4 – Synchronization
- synchronized book_room() method
- Prevents race conditions and double booking

### Week 5 – File Handling
- FileInputStream / FileOutputStream
- FileWriter (optional logs)

### Week 6 – Serialization
- Object streams for saving/loading `.dat` file

### Week 7 – Generics
- ArrayList<T>, optional Pair<T,U>

### Week 8 – Collections
- ArrayList, HashMap, Iterator usage

### Week 9 – JavaFX
- FXML + Scene Builder
- Event handling

### Week 10 – Integration
- Complete system combining all concepts

---

## Threading Design

### Booking Thread
- Triggered on booking
- Runs in background
- Simulates delay (1–2 sec)
- Updates UI using Platform.runLater()

### Auto-Save Thread (Bonus)
- Daemon thread
- Runs every 30 seconds
- Saves application state

---

## Synchronization
- synchronized book_room()
- Ensures:
  - No double booking
  - Thread safety
  - Data consistency

---

## Constraints (VERY IMPORTANT)

### Functional Constraints
- No database (strictly file-based)
- Standalone desktop app only
- Must support add, view, booking, checkout

### Data Constraints
- room_number → unique, integer
- customer_id → unique, string
- price_per_night > 0
- check_out_date > check_in_date

### System Constraints
- No duplicate room booking
- Cannot remove customer if room allocated
- Cannot checkout already available room
- Input must be validated (no empty fields)

### UI Constraints
- Must use JavaFX
- Must include at least 3–4 tabs
- Must use proper layouts (GridPane, VBox, etc.)

---

## Tech Stack
- Java 23
- JavaFX 21
- Maven (latest stable)
- CSS (JavaFX styling)

---

## Project Structure (Maven)

hotel-management/

│

├── src/main/java/com/hotel/
│   ├── model/
│   ├── service/
│   ├── controller/
│   └── Main.java
│
├── src/main/resources/
│   ├── fxml/
│   └── css/
│
├── pom.xml

---

## Change Log

### v1.6
- Updated to Java 23
- Added FXML controller binding guidelines
- Improved robustness for JavaFX integration
