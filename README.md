# IoT Energy Monitoring Gateway

A fault tolerant IIoT data pipeline that simulates industrial energy meters, streams their readings through a Python gateway, and keeps collecting data even when the time-series database goes offline  using a local store and forward queue that backfills automatically on recovery.

## Why this project exists

Most IoT tutorials demonstrate the happy path: sensor → database → dashboard. Real industrial systems don't get that luxury — networks drop, databases restart, and connections fail without warning. This project is built around one question:

**What happens to your data when the database goes down mid-stream — and does the system recover without human intervention?**

## Architecture

```
                    ENERGY MONITORING SYSTEM
                           ARCHITECTURE

┌─────────────────────────────────────────────┐
│          3 SIMULATED ENERGY METERS           │
│                                               │
│  Meter 1       Meter 2        Meter 3        │
│  230V, 5A      225V, 4A       235V, 6A       │
│  1150W         900W           1410W          │
└──────┬───────────┬────────────┬──────────────┘
       │           │            │
       └───────────┴────────────┘
                    │
              MODBUS TCP
                    │
                    ▼
        ┌───────────────────────┐
        │    PYTHON GATEWAY     │
        │       gateway.py      │
        │                       │
        │ • Read Modbus data    │
        │ • Process readings    │
        │ • Store data          │
        │ • Publish MQTT        │
        │ • Recovery handling   │
        └───────┬───────┬───────┘
                │       │
        ┌───────┘       └────────┐
        │                        │
        ▼                        ▼
┌───────────────┐         ┌───────────────┐
│   TDENGINE    │         │     MQTT      │
│               │         │   Mosquitto   │
│ meter1        │         │               │
│ meter2        │         │ energy/meter1 │
│ meter3        │         │ energy/meter2 │
└───────┬───────┘         │ energy/meter3 │
        │                 └───────────────┘
        │
        │  DATABASE FAILURE
        ▼
┌───────────────────────┐
│        SQLITE         │
│    Offline Queue       │
│  Store data locally    │
└───────────┬───────────┘
            │ TDengine returns
            ▼
┌───────────────────────┐
│   RECOVERY WORKER      │
│ SQLite → TDengine       │
│ Delete after success    │
└───────────┬───────────┘
            ▼
       ┌───────────┐
       │ TDENGINE  │
       └─────┬─────┘
             ▼
      ┌─────────────┐
      │   GRAFANA   │
      │ Meter 1     │
      │ Meter 2     │
      │ Meter 3     │
      │ Total Power │
      └─────────────┘
```

### Data flow

**Normal operation**
```
Meters → Modbus TCP → Python Gateway → TDengine → Grafana
```

**Database failure**
```
Meters → Modbus TCP → Python Gateway → TDengine  → SQLite (queued)
```

**Recovery**
```
SQLite → TDengine comes back → Recovery Worker → TDengine ✅ → Grafana
```

## Tech stack

| Layer | Technology | Purpose |
|---|---|---|
| Field devices | Simulated Modbus TCP meters | 3 virtual energy meters (voltage, current, power) |
| Protocol | Modbus TCP | Industrial-standard meter communication |
| Gateway | Python | Reads Modbus data, processes readings, handles storage and recovery |
| Time-series storage | TDengine | Stores meter readings as time-series data |
| Messaging | MQTT (Mosquitto) | Publishes live readings per meter topic |
| Offline buffer | SQLite | Local durable queue during database outages |
| Visualization | Grafana | Per-meter and total power dashboards |
| Deployment | Docker | Containerized services |

## Features

- Simulates 3 independent energy meters with distinct voltage/current/power profiles
- Modbus TCP communication between simulated meters and the gateway
- Dual-write architecture: TDengine (historian) and MQTT (live pub/sub) are independent, so one going down doesn't block the other
- Store-and-forward offline queue: readings are never dropped during a TDengine outage
- Automatic recovery worker that replays queued readings and deletes them only after a confirmed write
- Grafana dashboards for per-meter and total power visualization
- Fully containerized for reproducible setup

## Getting started

### Prerequisites

- Docker and Docker Compose
- Python 3.x (if running the gateway outside a container)

### Setup

```bash
git clone <your-repo-url>
cd energy-monitoring-gateway
docker-compose up -d
```

This brings up TDengine, Mosquitto, Grafana, and the meter simulator/gateway containers.

### Verifying data is flowing

```bash
docker exec -it tdengine taos
```

At the `taos>` prompt:

```sql
USE energy;
SELECT * FROM meter1 LIMIT 5;
```

You should see rows matching the simulated readings, e.g. `230 5 1150`.

### Testing the failure/recovery path

1. With the meter simulator and gateway running, stop the TDengine container:
   ```bash
   docker stop tdengine
   ```
2. Watch the gateway logs — readings should now be written to the local SQLite queue instead of failing outright.
3. Restart TDengine:
   ```bash
   docker start tdengine
   ```
4. The recovery worker should detect the reconnection, replay queued rows into TDengine, and delete them from SQLite only after each write is confirmed.
5. Confirm in Grafana that the gap in the dashboard has backfilled.

## Sample readings

| Meter | Voltage | Current | Power |
|---|---|---|---|
| Meter 1 | 230V | 5A | 1150W |
| Meter 2 | 225V | 4A | 900W |
| Meter 3 | 235V | 6A | 1410W |
| **Total** | — | — | **3460W** |

## Known limitations / future work

- No authentication or encryption on the simulated Modbus TCP link — acceptable for a simulation, not for a real deployment
- Recovery worker replay ordering and idempotency under a gateway crash mid-replay haven't been stress-tested
- No backpressure or max-size handling on the SQLite queue during a prolonged outage
- No retry backoff on the recovery worker's reconnect attempts
- Planned: run as a systemd service on a Raspberry Pi for a real edge deployment
- Planned: extend the same architecture pattern to other IIoT use cases (e.g. edge inference pipelines feeding the same historian/dashboard stack)

## License

MIT (or your preferred license update this section)
