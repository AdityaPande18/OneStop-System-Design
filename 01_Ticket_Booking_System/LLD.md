# Low-Level Design: Ticket Booking System

### 1. BookingService
This class handles the booking process, including seat selection and payment.

```python
import uuid
import time

class BookingService:
    def __init__(self, seat_inventory_service):
        self.seat_inventory_service = seat_inventory_service
        self.sessions = {}

    def create_session(self, user, show, seats, ttl=300):
        if self.seat_inventory_service.are_seats_available(show, seats):
            session_id = str(uuid.uuid4())
            expiry_time = time.time() + ttl
            self.sessions[session_id] = {
                "user": user,
                "show": show,
                "seats": seats,
                "expiry_time": expiry_time
            }
            return session_id
        else:
            raise SeatsUnavailableException("Selected seats are not available.")

    def create_booking(self, session_id):
        session = self.sessions.get(session_id)
        if not session:
            raise SeatsUnavailableException("Session not found or expired.")
        
        if time.time() > session["expiry_time"]:
            del self.sessions[session_id]
            raise SeatsUnavailableException("Session has expired.")
        
        user = session["user"]
        show = session["show"]
        seats = session["seats"]
        
        self.seat_inventory_service.reserve_seats(show, seats)
        del self.sessions[session_id]
        return Booking(user, show, seats)
```

### 2. SeatInventoryService
This class manages seat availability and reservations.

```python
class SeatInventoryService:
    def __init__(self):
        self.seat_inventory = {}

    def are_seats_available(self, show, seats):
        available_seats = self.seat_inventory.get(show, [])
        return all(seat in available_seats for seat in seats)

    def reserve_seats(self, show, seats):
        available_seats = self.seat_inventory.get(show, [])
        for seat in seats:
            if seat in available_seats:
                available_seats.remove(seat)

    def add_seats(self, show, seats):
        if show not in self.seat_inventory:
            self.seat_inventory[show] = []
        self.seat_inventory[show].extend(seats)
```

### 3. Supporting Classes

#### User
Represents a user in the system.

```python
class User:
    def __init__(self, user_id, name):
        self.user_id = user_id
        self.name = name
```

#### Show
Represents a show for which tickets can be booked.

```python
class Show:
    def __init__(self, show_id, movie_name, show_time):
        self.show_id = show_id
        self.movie_name = movie_name
        self.show_time = show_time
```

#### Seat
Represents a seat in the theater.

```python
class Seat:
    def __init__(self, seat_number, seat_type):
        self.seat_number = seat_number
        self.seat_type = seat_type
```

#### Booking
Represents a booking made by a user.

```python
class Booking:
    def __init__(self, user, show, seats):
        self.user = user
        self.show = show
        self.seats = seats
```

#### SeatType
Enum to represent different types of seats.

```python
from enum import Enum

class SeatType(Enum):
    REGULAR = "Regular"
    PREMIUM = "Premium"
    VIP = "VIP"
```

### Exceptions

#### SeatsUnavailableException
Custom exception for unavailable seats.

```python
class SeatsUnavailableException(Exception):
    def __init__(self, message):
        super().__init__(message)
```
