# Scrimmage Team

**NB**: The details in this file (Tech Stack, Repo Structure, etc) are specific to the particular project the Scrimmage Team is working on in this repo. 

## Getting Started

Read `scrimmage-team.md` in this repo and follow it. If you have not already been assigned a role in the team, then you are the scrimmage master (servant leader).

## Tech Stack

- **Frontend:** React (TypeScript)
- **Backend:** Python
- **Database:** DynamoDB
- **Infrastructure:** AWS CDK (TypeScript)
- **Cloud:** AWS

## Repo Structure

Monorepo layout:

```
/frontend    — React app
/backend     — Python services
/infra       — CDK stacks
```

## Conventions

- All infrastructure must be defined as CDK code in `/infra` — no manual AWS console changes.
- Backend code goes in `/backend` with tests alongside.
- Frontend code goes in `/frontend` with tests alongside.
- The product owner will be briefed by the user at runtime — do not assume what the product is.
- The **Product Backlog** lives in `backlog.db` (SQLite, WAL mode), managed via `backlog_db.py`. Import `get_backlog_db` to access it. The DB supports concurrent read/write by multiple agents.
