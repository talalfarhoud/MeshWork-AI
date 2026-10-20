# MeshWork AI

## Overview

MeshWork AI is an agentic planning application for software teams. Users submit a natural-language prompt describing a software design or architecture, and an AI agent constructs a corresponding graph of nodes and edges on a shared collaborative canvas. Team members can observe the AI build in real time and collaboratively refine the diagram. Once the design is complete, a separate background AI task converts the current graph into a downloadable Markdown technical specification.


## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Available Scripts](#available-scripts)
- [Project Structure](#project-structure)
- [Data and Storage](#data-and-storage)
- [Notes](#notes)


## Key Features

- **AI-assisted diagram generation**: Converts natural-language prompts into nodes and edges on a canvas.
- **Real-time collaboration**: Synchronized canvas state and presence so multiple users can work together on the same diagram.
- **Interactive canvas nodes**: Supports inline label editing, node resizing, and per-node color selection with changes synchronized across connected clients.
- **AI specification generation**: Produces a detailed Markdown technical specification from the current graph.
- **Multi-spec storage**: Stores specification metadata in the database and writes specification content as Markdown files on disk.
- **Downloadable specifications**: Allows generated specifications to be downloaded from the application.
- **Authentication-protected access**: Restricts access to authenticated users and issues real-time collaboration tokens only for authenticated sessions.
- **Automatic canvas persistence**: Periodically saves the canvas state to disk after periods of inactivity.
- **Project and room management**: Supports creating and navigating between projects, with each project mapped to its own collaborative room.
- **Shareable workspace context**: Provides a way to copy the current workspace URL for sharing with collaborators.


## Architecture

MeshWork AI is split between a web application, a real-time collaboration layer, a background job runner, and a hybrid persistence model. The following is based strictly off of the existing project structure and verified integrations. 

### High-Level Flow

1. **Client (Next.js App Router)** - The UI is rendered with Next.js and React. The canvas is built with React Flow, with collaboration logic layered on top via Liveblocks.
2. **Authentication (Clerk)** - Requests are protected via Clerk middleware. Authentication state is used to gate access to projects and to authorize the issuance of real-time collaboration tokens.
3. **Real-time Collaboration (Liveblocks)** - Each project maps to a collaborative room. Liveblocks handles presence, cursor positions, and shared canvas state. The Liveblocks Node.js SDK is used where server-side room operations are required.
4. **API Layer (Next.js Routes)** - Project creation, specification retrieval, downloads, and other server-side operations are exposed through Next.js API routes under `app/api/`.
5. **Background Jobs (Trigger.dev)** - Long-running AI work (diagram generation and specification generation) is offloaded to Trigger.dev tasks under `trigger/`. This separates model calls from the web application's request lifecycle.
6. **Persistence (Prisma + PostgreSQL + Filesystem)** - Relational data (such as project and specification metadata) is stored with Prisma in PostgreSQL. Generated documents and canvas snapshots are stored as files under `data/` (`data/canvas/` for JSON snapshots and `data/specs/` for Markdown specifications).


### Component Breakdown

| Area | Responsibility | Notes |
| :-- | :-- | :-- |
| `app/` | Routes, layouts, API endpoints, and auth pages | Next.js App Router. Route protection is applied via Clerk middleware. |
| `components/` | Editor UI, canvas components, and shared UI primitives | Contains canvas-specific components as well as `shadcn/ui` primitives. |
| `hooks/` | Client-side behavior | Includes behaviors such as auto-saving and keyboard shortcuts. |
| `lib/` | Integrations and shared utilities | Service clients and utilities for integrations (e.g. Prisma, Liveblocks, and AI) alongside shared logic. |
| `trigger/` | Background task definitions | Contains tasks for AI diagram generation and AI specification generation. |
| `prisma/` | Schema and migrations | Source of truth for relational models. |
| `data/` | File-based storage | Written to at runtime for canvas snapshots and generated specifications. Not a source of schema definitions. |
| `types/` | Shared TypeScript type definitions | Cross-cutting type contracts. |


### Tradeoffs

| Decision | Benefits | Tradeoffs |
| :-- | :-- | :-- |
| **Hybrid storage (DB + filesystem)** | Keeps relational metadata queryable in PostgreSQL while storing large generated Markdown and canvas JSON as files. This keeps rows smaller and allows for direct file downloads. | The filesystem becomes a required, writable runtime dependency. On some serverless hosts, cross-instance persistence can be difficult. Backups and migrations must account for both the database and files within `data/`. |
| **Real-time via Liveblocks** | Simplifies multiplayer synchronization (presence, cursors, and shared state), allowing the project to focus on the canvas and agent behavior. | Introduces a third-party dependency for real-time functionality. Requires network connectivity and correct token issuance to Liveblocks. |
| **Background jobs via Trigger.dev** | AI generation can be long-running without being constrained by HTTP request timeouts. Provides a dedicated space for orchestration outside of the web server. | Requires an additional service to run locally (`npx trigger.dev@latest dev`) and in deployment, with its own configuration and credentials. For AI features, two processes must be running in local development. |
| **Canvas as a collaborative room per project** | Naturally isolates workspaces, simplifies sharing, and allows for scoped presence and state per project. | Room lifecycle must be managed alongside projects. Authorization must be strictly enforced when issuing collaboration tokens. |
| **AI offloaded to external models (Google Gemini)** | Leverages capable models without the need to self-host them. | Dependent on third-party API availability, rate limits, and terms of service. Prompts and graph data are sent to the provider. While the model can be configured via environment variables, there is integration-level provider coupling. |
| **Next.js App Router + TypeScript** | Provides a unified codebase for UI and API routes with strong, end-to-end typing. | Tied to the conventions and versions currently in use (Next.js 16, React 19), which may differ from older versions of the framework. |


## Tech Stack

- [Next.js](https://nextjs.org/) and [React](https://react.dev/) with [TypeScript](https://www.typescriptlang.org/)
- [Liveblocks](https://liveblocks.io/) for real-time collaboration and presence
- [Clerk](https://clerk.com/) for authentication and session management
- [Trigger.dev](https://trigger.dev/) for background task orchestration
- [Prisma ORM](https://www.prisma.io/) with [PostgreSQL](https://www.postgresql.org/)
- [Tailwind CSS](https://tailwindcss.com/) and [shadcn/ui](https://ui.shadcn.com/)
- [Google Gemini](https://deepmind.google/models/gemini/) via the AI SDK


## Prerequisites

Before running the project locally, ensure you have the following installed: 

- [Git](https://git-scm.com/)
- [Node.js](https://nodejs.org/) (compatible with the Node version used by the project)
- [npm](https://www.npmjs.com/)


## Getting Started

1. **Clone the repository**
   ```bash
   git clone https://github.com/adrianhajdin/ghost-ai.git
   cd ghost-ai
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Configure environment variables**
   
   Create a `.env.local` file in the project root. Based off of the existing project configuration, the following environment variables are expected: 
   
   ```env
   # Clerk
   NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
   CLERK_SECRET_KEY=
   NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
   NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up

   # Liveblocks
   LIVEBLOCKS_SECRET_KEY=

   # Trigger.dev
   TRIGGER_SECRET_KEY=
   NEXT_PUBLIC_TRIGGER_PUBLIC_API_KEY=

   # Database
   DATABASE_URL=

   # Google Gemini
   GOOGLE_GENERATIVE_AI_API_KEY=
   # Optional: override the default Gemini model (default: gemini-2.0-flash)
   GEMINI_MODEL=
   # Optional: override model used specifically for spec generation
   GEMINI_SPEC_MODEL=

   # Application
   APP_URL=http://localhost:3000
   ```

   Populate each variable with valid credentials for your local environment. Do not commit actual secrets to the repository.

4. **Run the development server**
   ```bash
   npm run dev
   ```

   The application will be available at [http://localhost:3000](http://localhost:3000).

5. **Start the background task worker (required for AI features)**
   
   In a separate terminal from the project root, run: 
   ```bash
   npx trigger.dev@latest dev
   ```

   The Trigger.dev worker must be running for AI diagram generation and AI specification generation to execute.


## Available Scripts

| Script | Description |
| --- | --- |
| `npm run dev` | Start the Next.js development server |
| `npm run build` | Build the application for production |
| `npm run start` | Start the production server |
| `npm run lint` | Run ESLint |
| `npm run prisma:generate` | Regenerate the Prisma Client |
| `npm run prisma:migrate` | Create and apply a new database migration |
| `npm run prisma:deploy` | Apply pending migrations in production |
| `npm run prisma:studio` | Open Prisma Studio GUI |


## Project Structure

```text
.
├── app/                    # Next.js routes, API endpoints, and pages
├── components/             # UI and editor components
├── data/                   # Auto-saved canvas files and generated specifications
├── docs/                 # Project documentation
├── hooks/                  # Custom React hooks
├── lib/                  # Shared utilities and service integrations
├── prisma/                 # Prisma schema and migrations
├── trigger/                # Trigger.dev background task definitions
└── types/                  # Shared TypeScript type definitions
```


## Data and Storage

- **Canvas state**: Auto-saved as JSON at `data/canvas/{projectId}.json` during editing.
- **Generated specifications**: Stored as Markdown files at `data/specs/{projectId}/{specId}.md`, with metadata persisted in the database.


## Notes

- Environment files (`.env*`), build artifacts (`.next/`, `build/`, `out/`), dependencies (`node_modules/`), generated Prisma client files, and Clerk local configuration are excluded from version control per the project's `.gitignore`.
- The AI features require both the web application and the Trigger.dev worker to be running.


--- 

**Unresolved questions for approval:** 

1. **Licensing and attribution**: The original README contained substantial attribution and promotional material for JavaScript Mastery (JSM). No license file was observed during inspection. Should any attribution be retained for a public repository? If so, in what form and is it authorized? 
2. **Third-party asset rights**: The `public/readme/` directory contains imagery, and the original referenced GitHub-hosted assets. All of these have been removed. Confirmation is needed that this removal is sufficient and that none of these assets are required to be redistributed.
3. **Repository license**: No license has been specified. Should one be added to this repository? If yes, which license should be used?
4. **Trademark considerations**: Product names (Clerk, Liveblocks, Trigger.dev, Prisma, shadcn/ui, Google Gemini, etc.) are referenced strictly for factual description. Is this level of naming acceptable, or should it be adjusted to avoid any impression of endorsement? 
5. **Repository name and clone URL**: The clone URL still references `ghost-ai`. If the repository is being renamed to `meshwork-ai`, should this be updated accordingly?