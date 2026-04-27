# Game Server Orchestration Platform Overview

Game server orchestration platform for a multiplayer FPS, built on Kubernetes. A .NET Director API coordinates matchmaking via OpenMatch. Regional Allocators handle game server allocation through Agones, and Quilkin proxies route public UDP. Every region runs game server infrastructure while core services are centralised in the primary region.

## Architecture

![Architecture](architecture.png)

## What it does

Players authenticate via Steam, queue through OpenMatch, and connect to dedicated game server pods managed by Agones.

```
Player              Director API            OpenMatch         Redis        Allocator
  |                      |                      |                |              |
  |-- Login ------------>|                      |                |              |
  |<-- JWT --------------|                      |                |              |
  |                      |                      |                |              |
  |-- SSE Connect ------>|                      |                |              |
  |<-- state: none ------|                      |                |              |
  |                      |                      |                |              |
  |-- Start Queue ------>|-- Create Ticket ---->|                |              |
  |                      |-- Publish: queued ------------------->|              |
  |<-- state: queued ----|<--------------------- Subscribe ------|              |
  |                      |                      |                |              |
  |     (waiting)        |                      |                |              |
  |                      |<-- Match Found ------|                |              |
  |                      |                      |                |              |
  |                      |-- Allocate Server ---------------------------------->|
  |                      |<-- Regional proxy domain + routing token ------------|
  |                      |                      |                |              |
  |                      |-- Publish: matched ------------------>|              |
  |<-- state: matched ---|<--------------------- Push -----------|              |
  |                      |                      |                |              |
  |-- Join Session ----->|                      |                |              |
  |<-- Connection Info --|                      |                |              |
  |                      |                      |                |              |
  |=================== Connect to Game Server =================================>|
```

## Design notes

- **Quilkin UDP proxy with HMAC routing tokens.** UDP cannot be terminated and reissued by a normal L7 load balancer, and exposing GameServer pods directly is not ideal. An 8 byte HMAC token is prefixed on each client UDP packet; Quilkin validates and strips it before forwarding to the bound game server.
- **Region topology.** Every region runs the same cluster template, but the services node pool (Director API, OpenMatch, Redis) only deploys in the primary region as matchmaking is global. Allocators run per region for fast game server allocation.
- **OpenMatch 2 and Agones.** OpenMatch handles matchmaking via a custom MatchFunction (gRPC, .NET); Agones manages GameServer pod lifecycle, scaling, and allocation.
- **SSE over Redis pub/sub.** Push channel for player state changes without polling. One connection per player; new connections evict existing ones. Redis fans out state transitions across Director API replicas.
- **Server to server auth via HMAC.** Game servers authenticate to the Director via HMAC challenge response, receiving a JWT for session calls. Director to Allocator calls across regions are HMAC signed with per region keys.
- **Cloudflare Zero Trust Access for the operations dashboard.** Identity verified at the edge via signed JWT, validated by the Director. Email allowlist gates access.
- **Telemetry via Grafana Alloy.** DaemonSet on every node collects OpenTelemetry traces, metrics, and logs and ships to Grafana Cloud.

## Tech stack

| Layer | |
|---|---|
| API / Allocator / MatchFunction | .NET 10, ASP.NET Core, gRPC |
| Dashboard | React, TypeScript, Vite, TanStack Query |
| Database | Postgres (Aurora) |
| Messaging | Redis (matchmaking tickets and SSE pub/sub) |
| Matchmaking | OpenMatch 2 |
| Orchestration | Kubernetes (DigitalOcean DOKS), Agones |
| UDP proxy | Quilkin |
| Edge | Cloudflare (DNS, Pages, Zero Trust Access, WAF) |
| Auth | Steam Web API, JWT |
| IaC / CI | Terraform, GitHub Actions, GHCR, AWS S3 (tf state) |
| Observability | OpenTelemetry, Grafana Alloy, Grafana Cloud |

## Repository note

Implementation repo is private, this repo is an architectural overview only. 
