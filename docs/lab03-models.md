# Lab 03 – Use Cases, Classes and Data Flow

Project: Mobile Parking Finder
Emre Aracı

All diagrams are written in Mermaid, so they can be edited as text and GitHub draws them automatically.

## 1. Use case diagram

Actors:
- **Driver** – the person using the app
- **OpenStreetMap** – external source of parking locations
- **Occupancy simulator** – updates free space counts (test data)

For reservations the user has to log in, so I added a simple login with a token. Searching works without login.

```mermaid
flowchart LR
    Driver([Driver])
    OSM([OpenStreetMap])
    Sim([Occupancy simulator])

    subgraph System [Mobile Parking Finder]
        UC1(("UC1 Find nearest parking"))
        UC2(("UC2 Search area on map"))
        UC3(("UC3 View facility details"))
        UC4(("UC4 Log in"))
        UC5(("UC5 Reserve a space"))
        UC6(("UC6 Cancel reservation"))
        UC7(("UC7 Check reservation status"))
        UC8(("UC8 Import parking locations"))
        UC9(("UC9 Update free spaces"))
    end

    Driver --> UC1
    Driver --> UC2
    Driver --> UC3
    Driver --> UC5
    Driver --> UC6
    Driver --> UC7
    UC5 -. include .-> UC4
    UC6 -. include .-> UC4
    UC7 -. include .-> UC4
    OSM --> UC8
    Sim --> UC9
```

## 2. Class diagrams

### Mobile app

```mermaid
classDiagram
    class MapScreen {
        +onSearchTapped()
        +showResults(results)
        +showDataAge(minutes)
    }
    class ReservationScreen {
        +onReserveTapped(facilityId)
        +showStatus(status)
        +disableButtonsWhenOffline()
    }
    class LocationService {
        +getCurrentPosition() Position
        +distanceTo(position) double
    }
    class ApiClient {
        -token String
        +searchNearest(lat, lon) SearchResult
        +reserve(facilityId, requestId) Reservation
        +getStatusByRequestId(requestId) Reservation
    }
    class LocalCache {
        +saveLastResult(result)
        +getLastResult() SearchResult
        +savePendingRequest(requestId)
        +getPendingRequests() List
    }
    class ConnectivityMonitor {
        +isOnline() bool
        +onReconnect(callback)
    }

    MapScreen --> LocationService
    MapScreen --> ApiClient
    MapScreen --> LocalCache
    ReservationScreen --> ApiClient
    ReservationScreen --> LocalCache
    ReservationScreen --> ConnectivityMonitor
```

### Web service

```mermaid
classDiagram
    class AuthFilter {
        +checkToken(request) bool
    }
    class ParkingController {
        +GET /parking/nearest
        +GET /parking/area
        +GET /parking/id
    }
    class ReservationController {
        +POST /reservations
        +DELETE /reservations/id
        +GET /reservations/by-request/requestId
    }
    class RequestValidator {
        +validatePosition(lat, lon)
        +validateRequestId(requestId)
    }
    class ParkingService {
        +findNearest(lat, lon, limit) List
        +findInArea(box) List
    }
    class ReservationService {
        +reserve(userId, facilityId, requestId) Reservation
        +cancel(reservationId)
        +findByRequestId(requestId) Reservation
        +expireOldReservations()
    }
    class Facility {
        id
        name
        location
        capacity
        freeSpaces
        updatedAt
    }
    class Reservation {
        id
        requestId
        userId
        facilityId
        status
        createdAt
        expiresAt
    }
    class OsmImportJob {
        +importFromOverpass()
        +validateData()
    }

    AuthFilter --> ReservationController
    ParkingController --> RequestValidator
    ReservationController --> RequestValidator
    ParkingController --> ParkingService
    ReservationController --> ReservationService
    ParkingService --> Facility
    ReservationService --> Reservation
    ReservationService --> Facility
    OsmImportJob --> Facility
```

## 3. Data flow – normal read (find nearest parking)

```mermaid
sequenceDiagram
    participant U as Driver
    participant App as Mobile app
    participant API as Web service
    participant DB as Database

    U->>App: tap "Nearest parking"
    App->>App: get GPS position
    App->>API: GET /parking/nearest?lat&lon&time
    API->>API: validate position
    API->>DB: nearest query (free spaces > 0)
    DB-->>API: 10 facilities
    API-->>App: results + computedAt + position used
    App->>App: save to cache
    App-->>U: show list with "updated just now"
```

No login is needed here, because it only reads data.

## 4. Data flow – data changing action (reserve)

```mermaid
sequenceDiagram
    participant U as Driver
    participant App as Mobile app
    participant API as Web service
    participant DB as Database

    U->>App: tap "Reserve"
    App->>App: check online, create requestId
    App->>App: save requestId as pending
    App-->>U: "Pending..."
    App->>API: POST /reservations (token, facilityId, requestId)
    API->>API: check token (authentication)
    API->>API: validate request
    API->>DB: requestId already used?
    DB-->>API: no
    API->>DB: transaction: if freeSpaces > 0 then freeSpaces - 1, insert reservation
    Note over API,DB: The SERVER decides if the reservation succeeded
    DB-->>API: OK
    API-->>App: status RESERVED
    App->>App: remove from pending
    App-->>U: "Reserved"
```

**Who decides success?** Only the web service, inside the database transaction. The app never decides by itself. It only shows what the server answered.

## 5. Interrupted action and recovery after reconnect

```mermaid
sequenceDiagram
    participant U as Driver
    participant App as Mobile app
    participant API as Web service
    participant DB as Database

    U->>App: tap "Reserve"
    App->>App: save requestId as pending
    App->>API: POST /reservations (requestId)
    API->>DB: reservation saved
    API--xApp: response lost (connection dropped)
    App-->>U: "Status unknown, checking..."

    Note over App: connection comes back
    App->>App: read pending requestIds
    App->>API: GET /reservations/by-request/{requestId}
    API->>DB: find by requestId
    DB-->>API: RESERVED
    API-->>App: RESERVED
    App->>App: remove from pending
    App-->>U: "Reserved"
```

If the server answers NOT_FOUND, the request never arrived, so the app shows "Not reserved, try again". Because the same requestId is used, sending again can't create a second reservation.

## 6. External data boundary

```mermaid
flowchart LR
    subgraph External [Outside our system]
        OSM[OpenStreetMap Overpass API]
    end
    subgraph Ours [Our system]
        Job[OSM import job + validation]
        DB[(Database)]
        API[Web service]
        Sim[Occupancy simulator]
    end
    App[Mobile app]

    OSM -- "parking locations (untrusted)" --> Job
    Job -- "only valid records" --> DB
    Sim -- "free spaces" --> DB
    DB <--> API
    App <-. wireless .-> API
```

OpenStreetMap data comes from outside, so it is checked before saving (missing coordinates, duplicates, places outside the city area). The app never calls OpenStreetMap directly. It only talks to our web service.

## 7. Explaining one path

Reservation path (section 4 and 5): the user taps Reserve and the app creates a request ID and saves it as pending before sending. The server checks the token, validates the request and then does the reservation in one transaction, so two users can't take the last space. If the answer gets lost, the app still has the request ID. When the connection comes back, it asks the server what happened with that ID and shows the real result. This way the user never sees a wrong "Reserved" or a double reservation.

## 8. Requirements mapping

| Requirement (Lab 02) | Use case | Responsible component |
|---|---|---|
| FR1 send location with search | UC1 | LocationService, ApiClient |
| FR2 10 nearest parkings | UC1 | ParkingService |
| FR3 show data age | UC1, UC2 | MapScreen, LocalCache |
| FR4 search again after moving | UC1 | MapScreen, LocationService |
| FR5 unique request ID | UC5, UC6 | ApiClient, RequestValidator |
| FR6 same request ID only once | UC5 | ReservationService |
| FR7 reserve only if space is free | UC5 | ReservationService (DB transaction) |
| FR8 "Reserved" only after confirmation | UC5 | ReservationScreen |
| FR9 check unknown result by ID | UC7 | ReservationScreen, ReservationController |
| FR10 buttons disabled offline | UC5, UC6 | ReservationScreen, ConnectivityMonitor |
| FR11 cache last result | UC1, UC7 | LocalCache |
| NFR5 free spaces never negative | UC5 | ReservationService |
| Login for reservations (new) | UC4 | AuthFilter |

## Connection to SEARCH (week 3)

Our app doesn't need to search for the user, because the user always starts the request and the answer goes back on the same connection. But the recovery in section 5 is similar to the idea in the lecture: if delivery fails, we don't keep retrying the old route. The app asks again when it is reachable.
