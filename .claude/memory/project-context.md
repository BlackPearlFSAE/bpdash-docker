---
name: project-context
description: bpdash — personal FSAE telemetry dashboard project; goal is to redesign from scratch in Vite + Express
metadata:
  type: project
---

**Project:** bpdash — a real-time telemetry dashboard for an FSAE electric vehicle. Receives data from vehicle MCU nodes over WebSocket, displays live sensor data, and records sessions to PostgreSQL for post-run analysis.

**Current state:** Working system exists in `frontend/` (React/Vite) and `backend/` (Node.js/Express + Sequelize + ws), composed via Docker. The existing codebase is the reference — the goal is to redesign it from scratch as a learning exercise, not modify it destructively.

**Objective:** Project-based learning. Rebuild the system incrementally in the same stack. User drives the direction; Claude guides on the sideline.

**Key architectural facts to keep in mind:**
- Normalization happens server-side only (`backend/utils/dataProcessor.js`) — frontend receives pre-scaled flat objects
- Two WS roles: device clients push raw telemetry; dashboard clients (`?role=dashboard`) receive normalized broadcasts
- DB writes only happen while a session is active (`activeSession` singleton in `sessionRoutes.js`)
- Data groups follow `<node>.<type>` naming (e.g. `front.mech`, `bmu3.cells`)

**Do not** touch or destructively modify the existing `frontend/` and `backend/` source when building the redesign — they are the reference implementation.

See also: [[user-profile]], [[feedback-communication]]
