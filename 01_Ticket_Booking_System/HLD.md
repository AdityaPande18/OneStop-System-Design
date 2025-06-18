# High-Level Design (HLD) for Ticket Booking System

## Table of Contents
1. **Introduction**
2. **System Requirements**
3. **Architecture Overview**
4. **Component Design**
5. **Data Flow**
6. **Database Design**
7. **Non-Functional Requirements**
8. **Conclusion**

---

## 1. Introduction
The Ticket Booking System is designed to allow users to book tickets for events, movies, or travel. The system ensures scalability, reliability, and user-friendly interactions.

---

## 2. System Requirements

### Functional Requirements
- User registration and authentication.
- Search for available tickets.
- Book, cancel, and modify tickets.
- Payment processing.
- Notifications for booking confirmation.

### Non-Functional Requirements
- High availability and fault tolerance.
- Scalability to handle peak loads.
- Secure payment processing.
- Low latency for user interactions.

---

## 3. Architecture Overview
The system follows a microservices-based architecture with the following components:
- **Frontend**: Web and mobile interfaces.
- **Backend**: RESTful APIs for core functionalities.
- **Database**: Relational and NoSQL databases for structured and unstructured data.
- **Third-party Integrations**: Payment gateways, notification services.

---

## 4. Component Design

### 4.1 User Service
- Handles user registration, login, and profile management.

### 4.2 Ticket Service
- Manages ticket inventory, availability, and booking.

### 4.3 Payment Service
- Processes payments and handles refunds.

### 4.4 Notification Service
- Sends email/SMS notifications for booking updates.

---

## 5. Data Flow
1. User searches for tickets → Ticket Service fetches availability.
2. User selects tickets → Booking request sent to Ticket Service.
3. Payment processed → Payment Service validates and confirms.
4. Notification sent → Notification Service informs the user.

---

## 6. Database Design

### Tables

#### Users Table
- **UserID**: Unique identifier for each user (Primary Key).
- **UserName**: Name of the user.
- **Email**: Email address of the user.
- **PasswordHash**: Encrypted password for authentication.
- **PhoneNumber**: Contact number of the user.
- **CreatedAt**: Timestamp when the user account was created.

#### Tickets Table
- **TicketID**: Unique identifier for each ticket (Primary Key).
- **ShowID**: Identifier for the associated show (Foreign Key).
- **Price**: Price of the ticket.
- **SeatNumber**: Seat assigned to the ticket.
- **Status**: Status of the ticket (e.g., Available, Booked, Cancelled).

#### Bookings Table
- **BookingID**: Unique identifier for each booking (Primary Key).
- **UserID**: Identifier for the user who made the booking (Foreign Key).
- **ShowID**: Identifier for the booked show (Foreign Key).
- **BookingDate**: Date and time when the booking was made.
- **TotalAmount**: Total amount paid for the booking.
- **BookingStatus**: Status of the booking (e.g., Confirmed, Cancelled).

#### Payments Table
- **PaymentID**: Unique identifier for each payment (Primary Key).
- **BookingID**: Identifier for the associated booking (Foreign Key).
- **PaymentDate**: Date and time of the payment.
- **Amount**: Amount paid.
- **PaymentMethod**: Method of payment (e.g., Credit Card, PayPal).
- **PaymentStatus**: Status of the payment (e.g., Success, Failed).

#### Show Table
- **ShowID**: Unique identifier for each show (Primary Key).
- **ShowName**: Name of the show or event.
- **ShowType**: Type of the show (e.g., Movie, Concert, Play).
- **StartTime**: Start time of the show.
- **EndTime**: End time of the show.
- **Venue**: Location where the show is held.
- **AvailableSeats**: Number of seats available for booking.
- **TotalSeats**: Total number of seats for the show.

---
## Key System Flows: Booking Seats (Critical Path)

### Challenge: Prevent Double-Booking in Concurrent Scenarios
In a high-traffic environment, multiple users may attempt to book the same seat simultaneously, leading to potential double-booking issues. Ensuring data consistency and preventing such conflicts is critical.

### Solution: Optimistic Locking with Retry Mechanism
To handle concurrent seat booking, the system employs optimistic locking. This approach ensures that updates to the seat availability are only applied if the data has not been modified by another transaction since it was last read.

### Flow of the Process
1. **User Initiates Booking**:
    - The user selects a seat and initiates the booking process.

2. **Fetch Seat Availability**:
    - The Ticket Service queries the database for the selected seat's current status.

3. **Apply Optimistic Lock**:
    - The system retrieves the seat record along with a version number or timestamp.

4. **Attempt to Update Seat Status**:
    - The system attempts to update the seat's status to "Booked" by checking the version number or timestamp.
    - If the version number matches, the update is successful.
    - If the version number has changed (indicating another transaction modified the record), the update fails.

5. **Handle Update Failure**:
    - If the update fails, the system retries the process or informs the user that the seat is no longer available.

6. **Confirm Booking**:
    - Once the seat status is successfully updated, the booking is confirmed, and the Payment Service processes the payment.

7. **Send Notification**:
    - The Notification Service sends a confirmation message to the user.

### Benefits
- Prevents double-booking by ensuring data consistency.
- Handles high-concurrency scenarios effectively.
- Provides a seamless user experience with retry mechanisms.

--- 

## 7. Non-Functional Requirements
- **Scalability**: Auto-scaling for high traffic.
- **Security**: Data encryption and secure APIs.
- **Performance**: Response time under 200ms.
- **Reliability**: 99.9% uptime SLA.

---

## 8. Conclusion
The Ticket Booking System is designed to provide a seamless and secure experience for users while ensuring scalability and reliability for business needs.
