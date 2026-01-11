# Yıkamacım Var

> **A scalable, on-demand car wash booking platform connecting vehicle owners with premium service providers through a seamless mobile experience.**

---

## Important Disclaimer
**This project is a live commercial product developed under Vectorium Technology.**
The full source code is **proprietary and protected by NDA**. The repository contents have been redacted or limited to protect intellectual property. The documentation below serves as a **technical showcase and architectural overview** to demonstrate engineering capabilities, architectural decisions, and modern full-stack development practices.

---

## Project Overview
**Yıkamacım Var** (Turk: "I Have a Car Wash") is a B2C/B2B mobile application that digitizes the traditional car wash industry. It allows users to discover nearby car wash stations, view service menus, book appointments in real-time, and make secure online payments.

The system is built to handle complex scheduling logic, location-based services, and high-concurrency booking requests, ensuring a smooth experience for both end-users and service providers.

## Tech Stack

### Core Architecture
| Domain | Technologies |
| :--- | :--- |
| **Mobile Frontend** | **React Native** (Expo SDK 54), React 19 |
| **Backend API** | **Python**, **Django 5.2**, Django REST Framework (DRF) |
| **Database** | **PostgreSQL** / MySQL, SQLite (Dev) |
| **Async Tasks & Caching** | **Celery**, **Redis**, Django Celery Beat |
| **DevOps & Infra** | **Docker**, Docker Compose, Gunicorn |

### Detailed Libraries & Tools

#### Frontend (Mobile)
*   **Navigation:** `React Navigation v7` (Native Stack & Bottom Tabs) for fluid, native-like navigation.
*   **State & Data Fetching:** **SWR** (Stale-While-Revalidate) for powerful server state management, caching, and optimistic updates.
*   **UI/UX:** `lucide-react-native` (Icons), `expo-blur`, `expo-linear-gradient` for a modern, glassmorphic aesthetic.
*   **Forms & Inputs:** `react-native-confirmation-code-field` (OTP), `react-native-calendars` (Complex Scheduling).

#### Backend (API)
*   **Authentication:** `SimpleJWT` (Stateless generic JWT auth), custom SMS OTP integration via **Verimor**.
*   **Payments:** **Iyzipay** (Iyzico) & **PayTR** generic integration modules for secure 3D processing.
*   **Admin & Osp:** `django-admin-interface`, `django-import-export` for advanced dashboard controls.
*   **Utilities:** `Pillow` (Image Processing), `python-slugify`, `django-filter`.

---

## Key Features

### 1. Secure Authentication Flow
*   Combined **JWT** (Access/Refresh tokens) with **SMS OTP** verification (`Verimor integration`).
*   Secure storage of tokens using generic secure stores (abstracted in generic Auth Services).

### 2. Smart Appointment Scheduling
*   Real-time slot availability checking preventing double-bookings.
*   Support for multiple service types and duration calculations.
*   Visual calendar interface for selecting dates and times using `react-native-calendars`.

### 3. Location & Service Discovery
*   Geolocation-based search to find nearest open facilities.
*   Filtering by service types (Interior, Exterior, Detailing).

### 4. Robust Payment Architecture
*   Integration with **Iyzico** and **PayTR**.
*   Support for secure 3D callbacks and webhook handling.
*   Payment state tracking (Pending, Success, Failed, Refunded).

### 5. Background Processing
*   **Celery + Redis** handles background tasks such as:
    *   Sending delay-tolerant notifications (Push/SMS).
    *   Cleaning up expired pending appointments.
    *   Generating daily transaction reports for business owners.

---

## Architecture & Engineering

### Frontend: Service-Repository Pattern
The React Native app avoids logic coupling in UI components by using a dedicated **Service Layer**.
*   **`src/services/`**: Contains raw API calls (`AuthService`, `AppointmentsService`, `PaymentService`).
*   **`src/hooks/`**: Custom hooks (likely wrapping SWR) that consume services and expose data/loading states to components.
*   **`src/screens/`**: Pure UI components that subscribe to hooks.

### Backend: Modular Monolith
The Django backend is structured as a **Modular Monolith**, separating concerns into distinct apps:
*   **`core/`**: Shared utilities, base models, and abstract logic.
*   **`services/`**: Business logic for service definitions and pricing.
*   **`appointments/`**: Complex booking logic, slot validation, and concurrency handling.
*   **`users/`**: Extended User model and profile management.
*   **`paytr/` & `verimor_integration/`**: Isolated integration layers for external vendors.

---

## Engineering Showcase (Code Samples)

### 1. Robust Authentication Service (Frontend)
This module handles JWT storage, automatic token refreshing via interceptors, and secure storage implementation.

```javascript
// src/services/AuthService.js

import AsyncStorage from '@react-native-async-storage/async-storage';
import { API_BASE_URL as CONFIG_API_BASE_URL } from '../config';

let API_BASE_URL = CONFIG_API_BASE_URL;
let ACCESS_TOKEN = null;
let REFRESH_TOKEN = null;

// Automatic Token Interceptor & Retry Logic
async function request(path, options = {}) {
  const headers = { ...options.headers };

  if (ACCESS_TOKEN && !headers.Authorization) {
    headers.Authorization = `Bearer ${ACCESS_TOKEN}`;
  }

  let res = await fetch(`${API_BASE_URL}${path}`, { ...options, headers });

  // Handle 401 Unauthorized (Token expired)
  if (res.status === 401 && REFRESH_TOKEN) {
    try {
      const newTokens = await refreshAuthToken(REFRESH_TOKEN);
      if (newTokens && newTokens.access) {
        // Update headers and retry the search
        headers.Authorization = `Bearer ${newTokens.access}`;
        res = await fetch(`${API_BASE_URL}${path}`, { ...options, headers });
      }
    } catch (refreshError) {
      console.error('Token refresh failed', refreshError);
    }
  }

  // ... Response handling and JSON parsing
  return body;
}

export async function refreshAuthToken(refresh) {
  try {
    const res = await fetch(`${API_BASE_URL}/api/token/refresh/`, {
      method: 'POST',
      body: JSON.stringify({ refresh }),
    });
    // ... Update local storage and memory
  } catch (error) {
    console.error('Refresh request error', error);
  }
}
```

### 2. Complex Time Slot Calculation Algorithm (Backend)
This View handles the complex logic of determining available time slots by cross-referencing existing appointments, opening hours, and service duration.

```python
# appointments/views.py

class AvailableTimeSlotsView(APIView):
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        # ... Validator logic
        
        # Real-time slot generation
        while current_slot_time.time() < close_time:
            slot_start_time = current_slot_time.time()
            
            # Calculate service end time
            service_end_datetime = current_slot_time + timedelta(minutes=service_duration)
            slot_end_time = service_end_datetime.time()
            
            # Check availability against booked slots
            is_available = True
            for booked_slot in booked_slots:
                # Logic to detect overlaps
                if (this_start < booked_end and this_end > booked_start):
                    is_available = False
                    break
            
            if is_available:
                available_slots.append({
                    'start_time': slot_start_time.strftime('%H:%M'),
                    'end_time': slot_end_time.strftime('%H:%M'),
                    'is_available': is_available
                })
            
            # Advance to next slot
            current_slot_time = next_slot_start_datetime
            
        return Response(response_data)
```

---

## App Preview

| Home Screen | Appointment Booking | Payment Flow |
| :---: | :---: | :---: |
| ![Home](https://via.placeholder.com/250x500?text=Home+Screen) | ![Booking](https://via.placeholder.com/250x500?text=Booking+Flow) | ![Payment](https://via.placeholder.com/250x500?text=Payment+Screen) |

---

## Team

**Developed by the Engineering Team at Vectorium Technology**

* **Mehmet Furkan Güneş** - Co-Founder & Full Stack Engineer
    * *Focus:* Mobile Architecture (React Native), UI/UX Design, Client-Side Logic.
* **Nihal Kemer** - Co-Founder & Full Stack Engineer
    * *Focus:* System Architecture, API Development (Django), Database Optimization.

