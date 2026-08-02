# Real-Time WebSocket Chat — Dockerized Deployment with Nginx & CI/CD

## Project Overview
This project is a real-time WebSocket chat application (FastAPI backend + static frontend), deployed using Docker, Docker Compose, and an Nginx reverse proxy, with automated deployment via GitHub Actions CI/CD.

The original repository contained a deliberately misconfigured deployment setup. All infrastructure and deployment issues were identified, debugged, fixed, deployed to a live cloud server, and automated with a CI/CD pipeline.

---

## Live Application

**Public IP:** [http://13.203.78.31/](http://13.203.78.31/)

---

## Documentation Index

All detailed documentation for this project is available in the `Documentation/` folder:

| Document | Description |
|---|---|
| [Debug.md](Documentation/Debug.md) | Full debugging log — every issue found in Dockerfile, docker-compose.yml, and nginx.conf, with exact error messages, root cause analysis, fixes applied, and verification steps |
| [GithubActionSetup.md](Documentation/GithubActionSetup.md) | Step-by-step CI/CD setup guide — SSH key generation, GitHub Secrets configuration, workflow file creation, and the deploy job definition |
| [GitHub Actions test run.pdf](Documentation/GitHub%20Actions%20test%20run.pdf) | Live end-to-end simulation of the CI/CD pipeline — screenshots showing the bugged production state, the code push, the pipeline running, successful deployment, and the working application verified across multiple browser tabs |
| [applicationArchitecture.drawio](Documentation/applicationArchitecture.drawio) | Editable architecture diagram (open in [draw.io](https://app.diagrams.net)) showing Browser → Public IP → Nginx (Docker) → Backend (Docker) inside the Docker network |
| [cicd-flow.drawio](Documentation/cicd-flow.drawio) | Editable CI/CD pipeline flow diagram (open in [draw.io](https://app.diagrams.net)) showing Developer → GitHub → GitHub Actions → SSH → Server → Live Deployment |

---

## Summary

| Area | Status |
|---|---|
| Docker containers fixed (Dockerfile, docker-compose.yml) | ✅ Done — see `Debug.md` |
| Nginx reverse proxy & WebSocket proxying fixed | ✅ Done — see `Debug.md` |
| Multi-user real-time chat verified | ✅ Verified across multiple browser tabs |
| Cloud deployment (AWS EC2) | ✅ Live at `13.203.78.31` |
| CI/CD pipeline (GitHub Actions) | ✅ Working — auto-deploys on every push to `main` |
| Architecture diagrams | ✅ Application architecture + CI/CD flow (`.drawio` files) |
| End-to-end simulation performed | ✅ See `GitHub Actions test run.pdf` |

---

## Quick Start

```bash
git clone https://github.com/akshay228437/Real-Time-WebSocket-Application-.git
cd Real-Time-WebSocket-Application-
docker-compose up -d --build
```

Access the application at `http://<public-ip>` once containers are up.

For CI/CD, every push to `main` automatically redeploys the application to the production server — see `Documentation/GithubActionSetup.md` for full setup details.
