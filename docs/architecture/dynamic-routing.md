# Technical Design Document: Dynamic Routing & Re-Routing Service

## 1. Purpose & Scope
The routing service has a **single responsibility**: computing paths and trip plans from an explicit request. A generated plan includes the stop sequence, the geometry of each leg, travel times and distances, and any unassigned stops accompanied by their rejection reasons.

### Out of Scope:
* Vehicle and driver management.
* Tenant configurations.
* Management of collection points and their operational statuses.
* Weight log recording.
* GPS telemetry tracking.
* Business decisions regarding *when* to trigger a plan calculation.
* Notifications and reporting.

*These responsibilities belong entirely to other services; the routing service does not store, manage, or decide them.*

---

## 2. Ecosystem Context
The core application consists of several Python services coordinated by a central orchestrator:

| Service | Responsibility | Relationship with Routing |
| :--- | :--- | :--- |
| **Orchestrator** | Decides when to plan, what data to request, and enforces business rules. Maintains the official route plan. | **The sole client** of the routing service. |
| **Operations** | Manages collection points, weigh-ins, stop statuses, and current load. | **Data source** for stops, passed dynamically within the request payload. |
| **Fleet & Identity** | Manages vehicles, drivers, and tenant configurations. | **Data source** for vehicle capacity and physical constraints. |
| **Telemetry** | Tracks GPS coordinates and last known locations. | **Source** of real-time position data used for re-routing. |
| **Routing (This Service)** | Computes optimized route plans and road geometries. | Responds exclusively to commands issued by the orchestrator. |
| **Reports** | Maintains metrics and historical data. | Consumes emitted events; does not interact with route calculation. |

Communication follows a command pattern from the orchestrator to the routing service, returning responses or events. Because the transport layer (e.g., RabbitMQ or Kafka) is decoupled, the routing interface design remains independent of the broker.

---

## 3. Overview & Objectives
The **Recycling Logistics Routing Service** is a core backend module designed to optimize waste collection routes for recycling trucks. Unlike static delivery routes, this system handles variable daily pickups from departments and enterprises, enforces mandatory intermediate stops (weight stations and material selling points), and supports real-time dynamic re-routing when a truck reaches maximum capacity mid-route.

### Key Objectives:
* **Capacitated Vehicle Routing (CVRP):** Calculate optimal daily collection paths based on real-time collection requests.
* **Mandatory Intermediate Workflows:** Ensure trucks automatically route to a **Weight Station** and subsequently a **Selling Facility** once capacity limits are hit or a batch is complete.
* **Real-Time Dynamic Re-Routing:** Instantly recalculate remaining stops and generate a revised route when a driver triggers a "Truck Full" event.

---

## 4. Design Principles

To ensure the routing module remains scalable, maintainable, and robust against physical logistics failures, the codebase adheres to the following principles:

* **Stateless Business Logic**: Every request contains all the necessary data required for calculation. The service does not need to query external services or maintain session state during a route computation.
* **Determinism:** Given the same input, engine version, and configuration, the service will always produce the exact same route result. (*Note: Enforced by setting a fixed random seed in the optimization resolver*).
* **Transport Independence:** The core domain logic remains completely decoupled from HTTP frameworks, messageing queues, or specific database drivers. External transports interface with the core domain via clean adaptors (Hexagonal / Clean Architecture).
* **Explicit Failure:** If the underlying mapping provider fails or times out, the service returns a explicit, retryable error. It will never return inaccurate, straight-line, or "fallback" estimated routes as if the were valid.

---

## 5. Architecture & Design Choice

* **Approach:** **Modular Monolith** (embedded within the existing Python backend).
* **Rationale:** Keeping the routing logic as an isolated package inside the core backend avoids premature microservice overhead, simplifies database transactions, and speeds up deployment. The package can be easily extracted into a standalone service later if computational load scales significantly.

```
routing-service/
│
├── app/
│   ├── domain/          # Models,capacity constraints, precedence rules
│   ├── application/     # Use cases: plan_route, replan_route, estimate_eta
│   ├── ports/           # Interfaces: MatrixProvider, GeometryProvider, Optimizer
│   ├── adapters/
│   │   ├── maps/        # osrm.py, mapbox.py
│   │   ├── solver/      # ortools_solver.py
│   │   ├── messaging/   # rabbitmq.py or kafka.py
│   │   └── http/        # Internal diagnostic API
│   └── contracts/       # Pydantic input/output schemas
├── tests/
└── Dockerfile
```

---

## 6. Service Responsibilities

To maintain clean separation of concerns and protect the domain logic, responsibilities are strictly divided between routing service and the outer backend application:

### 6.1 Inside the Routing Service (Pure Domain & Math):

* **Request Validation:** Validate incoming data payloads for structural and mathematical correctness.
* **Matrix Construction:** Build travel time and distance matrices via the mapping adapter (OSRM/Mapbox).
* **Optimization Solving:** Resolve the stop sequence while respecting capacity constraints and mandatory intermediate stops (weight stations and selling facilities).
* **Geometry & ETA Calculation:** Compute route geometries (polylines) for each leg and estimated times of arrival (ETAs) for every stop.
* **Plan Generation:** Return the calculated route plan, metrics, and any unassigned stops accompanied by their rejection/exclusion reasons.

### 6.2 Outside the Routing Service (Application & Orchestration Layer):

* **Data Filtering & Selections:** Decide which specific pickup points are eligible or included for a given route plan.
* **Business Event Interpretation:** Handle real-world business events (e.g., a truck reaching full capacity). The orchestrator detects the event, updates database states, and invokes a re-plan.
* **State Persistence:** Persist operational states, such as stop completions, weight logs, or live truck positions.

### 6.3 Example Workflow: "Truck Full" Scenario
1. The driver triggers the "Truck Full" action on the mobile app.
2. The app notifies the backend orchestrator.
3. The orchestrator updates the route state in the database.
4. The orchestrator requests a re-plan from the routing service, providing the **current GPS position**, **current load**, **remaining pending stops**, and the **facility as a mandatory destination**.
5. The routing engine calculates the optimal path purely based on these inputs, completely agnostic to *why* the re-plan was triggered.

---

## 7. Input and Output Contracts

The service **does not** store the following input entities; they must be provided fully within the request:

* **Vehicle**: Identifier, capacity in kg (optional), and current load in kg.
* **Origin**: Starting latitude and longitude.
* **Stop**: Identifier, location, estimated demand in kg, service time in minutes, and optional time window.
* **Facility**: Identifier, type (`WEIGHT_STATION` or `DISPOSAL`), location, service time, and optional schedule.

### 7.1 Request Payload Example

```yaml
request_id: req_20261010_0001
type: PLAN_ROUTE
vehicle:
  id: veh_01
  capacity_kg: 3000
  current_load_kg: 0
origin:
  lat: 19.43
  lng: -99.13
stops:
- id: stp_01
  lat: 19.44
  lng: -99.14
  demand_kg: 450
  service_time_min: 5
facilities:
- id: fac_scale
  type: WEIGHT_STATION
  lat: 19.40
  lng: -99.17
  service_time_min: 3
- id: fac_plant
  type: DISPOSAL
  lat: 19.38
  lng: -99.15
  service_time_min: 20
constraints:
  weigh_before_disposal: true
options:
  objective: DURATION
  time_limit_s: 10
```

### 7.2 Response Payload Example 

```yaml
request_id: req_20261010_0001
status: OK
engine_version: 1.4.0
sequence:
- order: 1
  type: STOP
  id: stp_01
  eta_s: 540
- order: 2
  type: FACILITY
  id: fac_scale
  eta_s: 1020
- order: 3
  type: FACILITY
  id: fac_plant
  eta_s: 1500
legs:
- from: origin
  to: stp_01
  distance_m: 1800
  duration_s: 540
  geometry: encoded_polyline
unassigned: []
metrics:
  total_duration_s: 2100
  total_distance_m: 9400
  solve_time_ms: 812
```

### 7.3 Status Definition

* `OK`: Solution is complete and optimal/feasible
* `PARTIAL`: Missing stops or the solver stopped before finding the global optimum due to a timeout.
* `INFEASIBLE`: No valid solution satisfies all constraints.

---

## 8. Request Types

* `PLAN_ROUTE`: Complete daily plan for a vehicle.

* `REPLAN_ROUTE`: Plan calculated from the current position, handling pending stops and mandatory destination facilities (e.g., truck full scenario).

* `ESTIMATE_ETA`: Calculates travel times and geometries for a fixed, predetermined sequence without running optimization. Used for ETA recalculation when ordering is already decided.

---

## 9. Algorithm & Constraints

* **Time Matrix**: Nodes include origin, stops, and facilities. The matrix is calculated via the map adapter and cached by provider, time window, and node pair with a short TTL to reflect traffic changes without excessive API calls.

* **Optimization Model**: Capacitated Vehicle Routing Problem (CVRP) with two additional constraints:

    * **Capacity Management**: Accumulated load cannot exceed vehicle capacity. When anticipated load exceeds capacity, the model routes through the weight station and disposal plant before continuing. The disposal plant resets current load.

    * **Precedence**: For any disposal plant drop-off, the weight station check must happen beforehand. Modelled natively using pickup-and-delivery precedence constraints in OR-Tools.

* **Solver**: Google OR-Tools Routing as the primary engine. Initial solutions use parallel cheapest insertion, improved with guided local search. Time limits are parameterized per request.

* **Demand**: The service relies strictly on demand data supplied in the request payload; it does not estimate or correct demand using historical data (handled upstream by the orchestrator).

--- 

## 10. Edge Cases & Error Handling

| Scenario | Behiavor |
| :--- | :--- |
| Request with zero stops | Returns `OK` with an empty sequence. Valid response, not an error  |
| Stop unreachable on road network | Included in `unassigned` with reason. Status: `PARTIAL`. |
| Single stop exceeds total capacity | Reported in `unassigned`. Status: `PARTIAL`. |
| No solution satisfies constraints | Status: `INFEASIBLE` with the failing constraints set. |
| Map provider unavailable | HTTP `503 Service Unavailable` (retryable). Straight-line fallback routes are strictly forbidden. |
| Solver timeout | Status: `PARTIAL` with the best solution found so far. |
| Duplicate request | `request_id` is idempotent. If calculation exists in cache/log, returns cached result; otherwise, recalculates. |

---

## 11. Persistence, Caching, and Observability

* **No Business Database**: The service does not persist stops, vehicles, or weighing logs.

* **Calculation Log (Optional)**: Stores request payload, response, engine version, and metrics for auditing/evaluation. Depends on business traceability requirements.

* **Matrix Cache**: Redis with a short TTL.

* **Metrics**: Solver latency, proportion of `PARTIAL` responses, map provider error rates, and solved instance sizes.

* **Logs**: Every input/output is logged with `request_id` to correlate with backend orchestration traces.

---

## 12. Communication Interfaces

* **Primary Input**: Asynchronous commands (`PLAN_ROUTE`, `REPLAN_ROUTE`, `ESTIMATE_ETA`) received via message queues. Responses are published back to designated reply queues or event streams.

* **HTTP Interface**: Internal FastAPI endpoints sharing the same contracts for diagnostics, testing, and synchronous orchestration calls.

* **Published Events (Optional)**: `ROUTE_CALCULATED` events to allow reporting services to log calculations asynchronously.

## 13. Technology Stack

* **Language**: Python 3.12
* **API Framework**: FastAPI (internal diagnostic HTTP interface)
* **Validation & Schemas**: Pydantic v2
* **Optimization Engine**: Google OR-Tools
* **Routing Adapters**: Self-hosted OSRM or Mapbox Matrix API behind a common adapter interface
* **Caching & Storage**: Redis (matrix cache & optional calculation logs)
* **Messaging**: RabbitMQ or Kafka (pending architectural decision)
* **Testing**: pytest (unit and integration tests with small problem instances)
* **Observability**: OpenTelemetry (traces and metrics)
* **Packaging**: Docker


---

## 14. Pending Decisions

| Topic | Options | Evaluation Criteria |
| :---  | :---    | :---                |
| **Messaging Broker** | RabbitMQ vs. Kafka | Event volume, reprocessing needs, team familiarity.|
| **Calculation Logging** | Yes / No | Business traceability and auditing requirements. |
| **Map Engine** | Self-hosted OSRM vs. Mapbox | Cost, control, live traffic integration. |
| **Multiple Trips** | Recharge nodes vs. Chained trips | Model complexity and solve-time trade-offs. |
| **Demand Estimation** | External / Internal | Who computes demand and with what historical data. |

---

## 15. Glossary

* **CVRP (Capacitated Vehicle Routing Problem)**: A variation of the Vehicle Routing Problem where vehicles have limited carrying capacity and must service a set of customers.

* **ETA (Estimated Time of Arrival)**: The predicted time a vehicle is expected to arrive at a specific stop or facility.

* **Orchestrator**: The core backend component responsible for managing workflow state, calling services, and applying business rules.

* **Polyline**: An encoded string representing a sequence of latitude/longitude coordinates used to draw route lines on map interfaces.

* **Stateless**: A design approach where a service processes requests using only the data provided within the request payload, without relying on internal database state or session storage.


* **Unassigned Stops**: Pickup locations that could not be included in the calculated route due to capacity limits, solver timeouts, or network unreachability.

---