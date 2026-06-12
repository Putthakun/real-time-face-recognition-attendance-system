<div align="center">

# Real-Time Face Recognition Attendance System

**An end-to-end attendance system: edge face detection, real-time recognition, and a management dashboard**

[![.NET](https://img.shields.io/badge/.NET-10-512BD4?style=flat&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![Vue](https://img.shields.io/badge/Vue-3-4FC08D?style=flat&logo=vuedotjs&logoColor=white)](https://vuejs.org)
[![YOLOv8](https://img.shields.io/badge/YOLOv8-Ultralytics-FF6F00?style=flat)](https://ultralytics.com)
[![InsightFace](https://img.shields.io/badge/InsightFace-buffalo_l-FF6F00?style=flat)](https://github.com/deepinsight/insightface)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-FF6600?style=flat&logo=rabbitmq&logoColor=white)](https://rabbitmq.com)
[![Redis](https://img.shields.io/badge/Redis-cache-DC382D?style=flat&logo=redis&logoColor=white)](https://redis.io)
[![SQL Server](https://img.shields.io/badge/SQL%20Server-Azure%20SQL%20Edge-CC2927?style=flat&logo=microsoftsqlserver&logoColor=white)](https://hub.docker.com/_/microsoft-azure-sql-edge)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?style=flat&logo=docker&logoColor=white)](https://docker.com)

</div>

---

## Overview

This project is a microservice-based attendance system that recognizes employees by face in real time. A camera feed is processed on an edge device, faces are detected and forwarded to a recognition service, matched against enrolled employees, and recorded as attendance — all manageable through a web dashboard.

The system is split into five independently deployable repositories, each with its own README covering setup, configuration, and API details.

---

## Architecture

```
┌────────────────────┐      face_events / face.detected      ┌──────────────────────────┐
│ face-recognition-   │ ───────────────────────────────────▶ │ face-recognition-server   │
│ edge                │        (RabbitMQ, face crop JPEG)     │ (InsightFace matching)    │
│                     │                                        │                          │
│ • OpenCV stream     │                                        │ • Embedding extraction    │
│ • YOLOv8 detection  │                                        │ • Cosine similarity match │
│ • MJPEG /stream     │                                        │ • Cooldown rules          │
└─────────┬───────────┘                                        └─────────┬────────────────┘
          │                                                              │
          │ embedding extraction                          POST /api/transactions
          │ (enrollment)                                  GET  /api/embeddings
          │                                                              │
          ▼                                                              ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│ face-recognition-api  (.NET 10 / ASP.NET Core)                                  │
│                                                                                  │
│ • JWT auth, role-based authorization (Admin / Supervisor / Employee)           │
│ • Employee, Camera, Transaction management (EF Core + SQL Server)              │
│ • Redis "face:vectors" cache sync                                              │
└─────────┬───────────────────────────────────────────────────────────┬──────────┘
          │                                                            │
          ▼                                                            ▼
┌────────────────────┐                                     ┌──────────────────────────┐
│ face-recognition-   │                                     │ face-recognition-infra    │
│ web (Vue 3 SPA)     │                                     │                          │
│                     │                                     │ • Azure SQL Edge          │
│ • Login / dashboard │                                     │ • Redis                  │
│ • Employee & camera │                                     │ • RabbitMQ                │
│   management        │                                     │   (shared Docker network) │
│ • Attendance history │                                     └──────────────────────────┘
└────────────────────┘
```

**Flow summary:**

1. **`face-recognition-edge`** reads a camera stream, detects faces with YOLOv8, and publishes cropped face images to RabbitMQ.
2. **`face-recognition-server`** consumes those events, extracts a face embedding (InsightFace), compares it against a Redis-cached set of known employee embeddings, applies cooldown rules, and reports a match as an attendance transaction.
3. **`face-recognition-api`** is the system of record — it stores employees, cameras, credentials, and transactions in SQL Server, keeps the Redis face-embedding cache in sync, issues JWTs, and exposes the embedding-extraction endpoint used when enrolling a new employee photo.
4. **`face-recognition-web`** is the admin/supervisor dashboard for managing employees and cameras and reviewing attendance history.
5. **`face-recognition-infra`** provides the shared SQL Server, Redis, and RabbitMQ instances via Docker Compose, on a common external network used by all the above services.

---

## Repositories

| Repository | Stack | Role |
|---|---|---|
| [`face-recognition-edge`](../face-recognition-edge) | Python, FastAPI, OpenCV, YOLOv8, aio-pika | Captures video, detects faces, streams live MJPEG, publishes detection events |
| [`face-recognition-server`](../face-recognition-server) | Python, FastAPI, InsightFace, aio-pika, Redis | Extracts embeddings, matches faces, applies cooldowns, records attendance |
| [`face-recognition-api`](../face-recognition-api) | .NET 10, ASP.NET Core, EF Core, SQL Server, Redis, JWT | Core API — employees, cameras, transactions, auth, face-vector cache |
| [`face-recognition-web`](../face-recognition-web) | Vue 3, TypeScript, Vite, Pinia | Admin/supervisor dashboard |
| [`face-recognition-infra`](../face-recognition-infra) | Docker Compose | Shared SQL Server, Redis, RabbitMQ |

---

## Key Design Points

- **Edge/cloud split** — heavy detection runs close to the camera (YOLOv8 on the edge device); only face crops, not full video, cross the network.
- **Shared face-vector cache** — embeddings live in a Redis hash (`face:vectors`) so the recognition server avoids querying SQL Server on every detection, while `face-recognition-api` keeps the cache authoritative.
- **Atomic enrollment** — when an employee photo is uploaded, the API extracts the embedding *before* writing any database records, so a photo with no detectable face never results in a half-created employee.
- **Two-tier cooldowns** — per-camera and per-employee cooldowns prevent duplicate attendance records from repeated detections of the same person.
- **Independent deployability** — each service has its own Dockerfile and can be built/deployed separately, connecting to shared infra over a common Docker network.

---

## Getting Started

1. Start shared infrastructure: [`face-recognition-infra`](../face-recognition-infra) (`make up`)
2. Start the core API: [`face-recognition-api`](../face-recognition-api)
3. Start the recognition server: [`face-recognition-server`](../face-recognition-server)
4. Start the edge service (with a webcam or RTSP source): [`face-recognition-edge`](../face-recognition-edge)
5. Start the dashboard: [`face-recognition-web`](../face-recognition-web)

Each repository's README has detailed setup, configuration, and API documentation.
