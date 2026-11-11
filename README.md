# Kanban Dashboard: CI/CD with Jenkins, Docker and AWS EC2

A React + TypeScript (Vite) kanban task manager. The production build is static
files served by nginx inside a Docker container, deployed automatically by Jenkins
on every push to GitHub.

## Pipeline flow

GitHub push -> webhook -> Jenkins (on EC2) -> docker build -> Docker Hub -> deploy on EC2 -> health check -> rollback on failure

## Application

| Item | Value |
|---|---|
| Install | `npm install` |
| Dev server | `npm run dev` (port 5173) |
| Production build | `npm run build` (output in `dist/`) |
| Build preview | `npm run preview` (port 4173) |
| Start command in production | none, nginx serves the static files |
| Environment variables | none required |

## Docker

- Multi-stage Dockerfile: `node:22-alpine` builds the app, `nginxinc/nginx-unprivileged:alpine` serves only the built `dist/` files.
- Runs as the non-root user `nginx`, exposes only port 8080.
- `HEALTHCHECK` requests the home page every 30 seconds.
- `.dockerignore` keeps `node_modules`, `.git` and `.env` out of the image.

Run it locally:

    docker build -t kanban:test .
    docker run -d --name kanban-test -p 80:8080 kanban:test

## Registry and versioning

Images are pushed to Docker Hub as `antonypeter02/second-project:<jenkins-build-number>`.
Every build gets a unique tag, so any version can be identified and rolled back to.

## AWS infrastructure

One Ubuntu EC2 instance (t3.medium, 2 vCPU, 4 GB) runs both Jenkins (port 8080) and the app container (port 80).

| Port | Source | Purpose |
|---|---|---|
| 22 | my IP | SSH |
| 8080 | anywhere | Jenkins UI and GitHub webhook |
| 80 | anywhere | the application |

## Jenkins pipeline (Jenkinsfile)

| Stage | What it does |
|---|---|
| Checkout | Fetches the repo from GitHub |
| Build image | `docker build`, tagged with the build number |
| Push to registry | Logs in with stored credentials, pushes the tag |
| Deploy | Replaces the running container, applying memory and CPU limits |
| Health check | Passes only if Docker reports healthy and the site answers over HTTP |
| Post actions | Rolls back to the previous image on failure, always logs out of the registry |

Docker Hub credentials are stored in the Jenkins credentials store (ID `dockerhub-creds`) and never appear in code or logs.

## Automatic trigger

A GitHub webhook (`/github-webhook/`, push events) starts the pipeline on every push, with no manual click.

## Health check, rollback and logging

- A deployment counts as successful only if the container is `healthy` and `curl` gets a response.
- If the check fails, the previous image is restarted and the build is marked failed.
- Evidence: Jenkins console output, `docker ps`, `docker logs`, `docker inspect`.

## Resource limits

The container runs with `--memory=128m --cpus=0.25`. It only serves static files and
uses a few MB at idle, so the limits leave plenty of headroom while stopping it from
starving Jenkins on the same 4 GB host.

## Application URL

http://13.203.204.164

## Screenshots

See the `screenshots/` folder.

## Docker Hub

https://hub.docker.com/r/antonypeter02/second-project

## GitHub repo

https://github.com/Antony604/Second-Project
