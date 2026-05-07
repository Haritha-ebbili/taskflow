# TaskFlow — Team Task Manager

A full-stack team task management application built with React, Express, PostgreSQL, and Clerk authentication.

---

## Live Demo

> Deploy via Replit to get a `.replit.app` live URL (click Publish in the editor).

---

## Features

- **Authentication** — Sign up / sign in via Clerk (email, Google, GitHub, etc.)
- **Dashboard** — Real-time command center with project stats, assigned tasks, overdue count, and activity feed
- **Projects** — Create, browse, archive, and delete projects; searchable grid view
- **Task Management** — Create tasks with title, description, priority, status, due date, and assignee; one-click status cycling (To Do → In Progress → Done)
- **Team Members** — Add members by email, assign roles (Admin / Member), update or remove members
- **Role-Based Access** — Admins can manage tasks, members, and project settings; members can view and update tasks
- **Activity Feed** — Every action (project created, task updated, member added) is logged and visible on the dashboard

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite, Tailwind CSS v4, shadcn/ui |
| Routing | Wouter |
| State / Data | TanStack Query (React Query) |
| Auth | Clerk (whitelabel, Replit-managed) |
| Backend | Node.js 24, Express 5 |
| Database | PostgreSQL + Drizzle ORM |
| Validation | Zod v4, drizzle-zod |
| API Contract | OpenAPI 3.1 → Orval codegen (React Query hooks + Zod schemas) |
| Build | esbuild (API), Vite (frontend) |
| Monorepo | pnpm workspaces |

---

## Project Structure

```
taskflow/
├── artifacts/
│   ├── api-server/          # Express 5 REST API (port 8080)
│   │   └── src/
│   │       ├── routes/      # users, projects, members, tasks, dashboard
│   │       ├── lib/auth.ts  # Clerk → DB user lookup helper
│   │       └── index.ts     # App entry, Clerk middleware
│   └── task-manager/        # React + Vite frontend (port 18810)
│       └── src/
│           ├── pages/       # home, dashboard, projects, project-detail, task-detail
│           ├── components/  # layout, auth-sync, shadcn/ui components
│           └── App.tsx      # ClerkProvider, routing
├── lib/
│   ├── db/                  # Drizzle schema + migrations
│   │   └── src/schema/      # users, projects, project_members, tasks, activity
│   ├── api-spec/            # OpenAPI 3.1 YAML + Orval codegen config
│   ├── api-client-react/    # Auto-generated React Query hooks (do not edit)
│   └── api-zod/             # Auto-generated Zod schemas (do not edit)
└── pnpm-workspace.yaml      # Workspace catalog + package discovery
```

---

## Database Schema

| Table | Key Columns |
|---|---|
| `users` | `id`, `clerkId`, `email`, `name`, `avatarUrl` |
| `projects` | `id`, `name`, `description`, `ownerId`, `status` |
| `project_members` | `projectId`, `userId`, `role` (admin/member) |
| `tasks` | `id`, `projectId`, `title`, `status`, `priority`, `assigneeId`, `dueDate` |
| `activity` | `id`, `type`, `actorId`, `projectId`, `taskId`, `description` |

---

## API Endpoints

### Auth / Users
| Method | Path | Description |
|---|---|---|
| POST | `/api/users/sync` | Sync Clerk user to local DB on login |
| GET | `/api/users` | Search users by name or email |

### Projects
| Method | Path | Description |
|---|---|---|
| GET | `/api/projects` | List projects for authenticated user |
| POST | `/api/projects` | Create a new project |
| GET | `/api/projects/:id` | Get project details |
| PATCH | `/api/projects/:id` | Update project (admin only) |
| DELETE | `/api/projects/:id` | Delete project (owner only) |

### Members
| Method | Path | Description |
|---|---|---|
| GET | `/api/projects/:id/members` | List project members |
| POST | `/api/projects/:id/members` | Add member by email (admin only) |
| PATCH | `/api/projects/:id/members/:memberId` | Update member role (admin only) |
| DELETE | `/api/projects/:id/members/:memberId` | Remove member (admin only) |

### Tasks
| Method | Path | Description |
|---|---|---|
| GET | `/api/projects/:id/tasks` | List tasks (filterable by status/assignee) |
| POST | `/api/projects/:id/tasks` | Create a task |
| GET | `/api/projects/:id/tasks/:taskId` | Get task details |
| PATCH | `/api/projects/:id/tasks/:taskId` | Update task |
| DELETE | `/api/projects/:id/tasks/:taskId` | Delete task |

### Dashboard
| Method | Path | Description |
|---|---|---|
| GET | `/api/dashboard/summary` | Aggregate stats (totals, overdue, completed this week) |
| GET | `/api/dashboard/my-tasks` | Tasks assigned to the current user |
| GET | `/api/dashboard/overdue-tasks` | All overdue tasks across user's projects |
| GET | `/api/dashboard/activity` | Recent activity feed (last 20 events) |
| GET | `/api/projects/:id/stats` | Per-project completion stats |

---

## Local Development Setup

### Prerequisites

- Node.js 20+
- pnpm (`npm install -g pnpm`)
- PostgreSQL database (or use Replit's built-in DB)

### Steps

```bash
# 1. Install dependencies
pnpm install

# 2. Set environment variables
#    DATABASE_URL=postgres://...
#    CLERK_SECRET_KEY=...
#    CLERK_PUBLISHABLE_KEY=...
#    VITE_CLERK_PUBLISHABLE_KEY=...

# 3. Push DB schema
pnpm --filter @workspace/db run push

# 4. Run codegen (only needed after OpenAPI spec changes)
pnpm --filter @workspace/api-spec run codegen

# 5. Start API server
pnpm --filter @workspace/api-server run dev

# 6. Start frontend (in a new terminal)
pnpm --filter @workspace/task-manager run dev
```

Frontend runs at `http://localhost:18810`, API at `http://localhost:8080`.

---

## Architecture Decisions

1. **Contract-first API**: The OpenAPI 3.1 spec (`lib/api-spec/openapi.yaml`) is the single source of truth. Orval generates both the server-side Zod validators and the client-side React Query hooks — no hand-written API glue.

2. **Clerk whitelabel auth**: Authentication is handled by Clerk provisioned via Replit. The frontend syncs Clerk's user identity to the local `users` table on every login via `AuthSync`.

3. **Role-based guards**: Every protected route calls `getAuthUser(req)` to resolve the Clerk JWT to a local DB user. Route-level checks enforce admin vs. member permissions.

4. **Activity log**: Any meaningful mutation (create project, update task, add member) writes to the `activity` table, powering the dashboard's real-time feed.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `CLERK_SECRET_KEY` | Yes | Clerk backend secret key |
| `CLERK_PUBLISHABLE_KEY` | Yes | Clerk publishable key (server) |
| `VITE_CLERK_PUBLISHABLE_KEY` | Yes | Clerk publishable key (frontend) |
| `VITE_CLERK_PROXY_URL` | Yes (Replit) | Clerk proxy URL injected by Replit |
| `SESSION_SECRET` | Yes | Express session secret |

---

## Design

- Dark theme — background `hsl(222, 47%, 11%)`, primary cyan `hsl(189, 94%, 43%)`
- Geist / Geist Mono fonts
- Monospace font for all badges, labels, and numeric counts (technical/IDE aesthetic)
- All-caps action buttons and nav labels

---

## Scripts

| Command | Description |
|---|---|
| `pnpm run typecheck` | Full TypeScript check across all packages |
| `pnpm run build` | Build all packages |
| `pnpm --filter @workspace/api-spec run codegen` | Regenerate API hooks from OpenAPI spec |
| `pnpm --filter @workspace/db run push` | Push schema changes to DB (dev only) |
