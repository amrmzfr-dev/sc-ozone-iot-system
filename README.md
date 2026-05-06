# Ozone IoT System

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat&logo=go&logoColor=white)](https://go.dev)
[![Rust](https://img.shields.io/badge/Rust-000000?style=flat&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![React](https://img.shields.io/badge/React-20232A?style=flat&logo=react&logoColor=61DAFB)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-007ACC?style=flat&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=flat&logo=postgresql&logoColor=white)](https://www.postgresql.org)
[![ESP32](https://img.shields.io/badge/ESP32-E7352C?style=flat&logo=espressif&logoColor=white)](https://www.espressif.com)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-F5822A?style=flat&logo=platformio&logoColor=white)](https://platformio.org)

> Source code is private. This repository documents the system architecture and engineering decisions.

---

## Overview

Full-stack IoT platform for managing commercial ozone treatment machines across multiple outlet locations in Malaysia. In production with real paying users, including deployments at enterprise clients in the Malaysian automotive sector.

The system connects ESP32-powered hardware units to a Go REST API and a React dashboard. It handles remote device management, timed treatment sessions, usage-based billing, and multi-outlet analytics — all from a single operator dashboard.

---

## System Architecture

![System Architecture](./assets/architecture.png)

```
┌──────────────────────────────────────────────────────────────┐
│                  React Dashboard (Vite + React 18)            │
│             TanStack Query · Zustand · Tailwind CSS           │
└────────────────────────────┬─────────────────────────────────┘
                             │ HTTPS / REST + SSE
┌────────────────────────────▼─────────────────────────────────┐
│                  Go REST API  (Gin + GORM)                     │
│               PostgreSQL · AlmaLinux VPS · UTC+8              │
└──────┬──────────────────────────────────────────┬────────────┘
       │ HTTPS REST                                │ HTTPS REST
       │ (event upload · command poll)             │ (handshake · OTA)
┌──────▼──────────┐                    ┌───────────▼───────────┐
│  ESP32 Device A │        ...         │   ESP32 Device N       │
│  FreeRTOS tasks │                    │   FreeRTOS tasks       │
│  Relay · OLED   │                    │   Relay · OLED         │
└─────────────────┘                    └───────────────────────┘
```

Each ESP32 controls a physical ozone machine at a customer outlet. The device captures treatment events locally, uploads them to the backend with exponential retry, and polls for remote commands every 30 seconds.

**Organisational model:**

```
Outlet  (e.g. "Johor Branch")
  └─ Machine  (e.g. MY-001-2024)
       └─ Device  (ESP32, identified by MAC address)
            └─ DeviceEvents  (treatment records)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Firmware MCU | ESP32 DevKit v1 |
| Firmware framework | C++ / PlatformIO (Arduino) |
| Firmware RTOS | FreeRTOS — 3 tasks across 2 cores |
| Firmware storage | EEPROM (counters + credentials) · SPIFFS (event queue) |
| Firmware RTC | DS3231 hardware clock |
| Backend language | Go |
| Backend framework | Gin |
| Backend ORM | GORM |
| Database | PostgreSQL (prod) · SQLite (dev) |
| Backend auth | bcrypt + cookie sessions + Bearer tokens |
| Hosting | VPS · AlmaLinux |
| Frontend build | Vite 5 |
| Frontend framework | React 18 + TypeScript 5 |
| Frontend state | Zustand (auth) · TanStack React Query v5 (server state) |
| Frontend UI | Radix UI + Tailwind CSS + Recharts |

---

## Key Features

**Device lifecycle management**
- New devices register via MAC address handshake and are held in a `pending` state until an admin approves them
- Approved devices receive a Bearer token; no event uploads are accepted before approval
- Admin dashboard provides one-click approve, reject, assign-to-machine, and remote command dispatch

**Timed treatment sessions**
- Three treatment tiers: Basic (5 min), Standard (10 min), Premium (15 min)
- Each tier maps to a dedicated relay, LED mirror, and billing counter
- AC voltage sampled every 30 ms via ZMPT101B sensor — treatment stops automatically if line voltage drops below 100 V

**Remote command execution**
- Backend maintains a per-device command queue
- Supported commands: `RESET_COUNTERS`, `CLEAR_MEMORY`, `CLEAR_QUEUE`, `REBOOT_DEVICE`, `GET_STATUS`, `SYNC_TIME`, `UPDATE_FIRMWARE`, `UPLOAD_LOGS`
- OTA firmware updates delivered over HTTPS with SHA-256 verification

**Analytics and billing**
- Treatment events aggregated into `DailyAnalytics`, `WeeklyAnalytics`, `MonthlyAnalytics` tables
- Dashboard charts break down usage by treatment type, outlet, and machine
- Admin-adjustable counters and invoice records for usage-based billing

**Real-time dashboard**
- Server-Sent Events (SSE) stream pushes device state changes to the dashboard without polling

**Role-based access**
- Three roles: `admin`, `operator`, `viewer`
- Enforced at the backend handler level

---

## Engineering Highlights

**Clock integrity across power loss**

ESP32 devices in field deployments lose power unpredictably. Without a reliable clock, timestamps on treatment events become meaningless for billing. The firmware anchors the RTC to a last-known-good Unix timestamp persisted in NVS before every treatment. On boot, if NTP is unavailable, the device falls back to this anchor rather than reporting epoch 0. Every POST payload includes a `clock_source` field (`ntp`, `rtc`, `nvs_anchor`) so the backend can assess timestamp confidence and flag suspicious records.

**Durable event queue on constrained hardware**

Treatment events are written to a SPIFFS-backed queue immediately on capture — before any network attempt. The network task drains this queue with exponential backoff and per-event idempotency IDs, so no treatment is lost to a transient WiFi outage and no event is double-counted on retry. EEPROM counters are protected with a CRC16 checksum and a 500 ms debounced write to reduce flash wear.

**Reliable data pipeline from hardware to cloud**

The firmware runs three FreeRTOS tasks across two cores. `appTask` (Core 1) owns the relay state machine, button debounce, and treatment countdown entirely independently of network state. `netTask` (Core 0) handles all I/O: WiFi reconnect, handshake, event upload, command polling, OTA checks, and NTP sync. Inter-task communication uses FreeRTOS queues — the app layer never blocks on network. `diagTask` provides a serial CLI and periodic heap/stack watermark reporting.

**PostgreSQL time discipline**

All timestamps stored as `TIMESTAMPTZ`. The backend operates at a fixed UTC+8 offset (Malaysia, no DST) rather than relying on server locale. Analytics aggregation buckets are computed in UTC+8 consistently across daily, weekly, and monthly tables, which avoids the class of midnight-boundary bugs common in systems that mix naive datetimes.

**TLS on a microcontroller**

The ESP32 pins the ISRG Root X1 certificate directly in firmware (valid until 2035). All backend communication is HTTPS. There is no fallback to plain HTTP in production builds.

**Multi-user billing on event data**

Treatment counters exist at two levels: raw `DeviceEvents` (immutable append-only log) and machine-level canonical counters (admin-adjustable). Billing is derived from the canonical counters. The admin dashboard exposes a Machine Adjustments page for correcting counters after hardware swaps or field incidents, with an audit trail.

---

## Project Status

**Production** — deployed on a VPS running AlmaLinux, serving real paying users across multiple outlet locations in Malaysia.
