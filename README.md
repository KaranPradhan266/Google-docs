# Google Docs Clone (Next.js + Convex + Clerk + CRDT/Yjs)

A collaborative document editor built with Next.js App Router, Tiptap, Convex, and Clerk. The realtime editing model is CRDT-based (Yjs-style updates) with presence, comments, and notifications layered on top.

## Table of contents

- Overview
- Features
- Tech stack
- Architecture
- CRDT + Yjs collaboration model
- Editor and UI composition
- Convex + Clerk: data, auth, and tenancy
- Data model
- Document lifecycle and discovery
- Metrics and observability
- Setup
- Running locally
- Scripts
- Project layout
- Troubleshooting

## Overview

This project is a Google Docs-style editor that supports authenticated, multi-user editing with rich text tools, document templates, organization-aware sharing, and realtime collaboration features like presence and threaded comments. It uses Convex for persistent data and server functions, and Clerk for authentication and organization identity. The collaboration layer is designed around CRDTs (specifically the Yjs mental model) to ensure conflict-free merges under concurrent edits.

## Features

- Rich text editing with Tiptap (tables, tasks, images, links, highlights, text styles)
- Document templates gallery (blank, proposal, resume, letters)
- Document list with search and pagination
- Realtime collaboration with presence avatars and comment threads
- Notifications inbox for comment activity
- Organization-aware access control (owner or org member)
- Server-authenticated collaboration sessions

## Tech stack

- UI: Next.js 15 App Router, React, Tailwind CSS, Radix UI
- Editor: Tiptap + custom extensions
- Realtime model: CRDTs (Yjs-style updates and awareness)
- Backend: Convex (queries, mutations, schema, indexes)
- Auth: Clerk (JWT templates, org membership, middleware)

## Architecture

High-level view:

```
+----------------------+      +-----------------------+
|  Browser (Next.js)   |<---->|  Next.js App Router   |
|  - Tiptap editor     |      |  - Pages & layouts    |
|  - Presence UI       |      |  - API routes         |
+----------+-----------+      +-----------+-----------+
           |                              |
           | JWT + session                | queries/mutations
           v                              v
      +---------+                    +-----------+
      | Clerk   |                    |  Convex   |
      | Auth    |                    |  DB + API |
      +----+----+                    +-----+-----+
           |                               |
           | realtime auth                  |
           v                               |
      +------------------------------+      |
      | CRDT sync provider (Yjs-     |<-----+
      | compatible)                  |
      | - document updates           |
      | - awareness/presence         |
      | - threads/inbox              |
      +------------------------------+
```

Key flows:

- The browser loads Next.js routes, which preload data from Convex.
- Clerk authenticates users and provides org membership data.
- Convex stores document metadata and enforces access control.
- A realtime provider streams CRDT updates so multiple editors converge.
- The document ID from the route is used as the collaboration room ID.

## CRDT + Yjs collaboration model

This project treats collaboration as a CRDT problem. The editor state is modeled as a shared document that can be updated concurrently by multiple clients. Each client emits incremental updates, and all replicas converge without conflicts.

Core ideas (Yjs-style):

- **Shared document**: The editor state is represented as a shared data structure (Y.Doc in Yjs terms).
- **Incremental updates**: Each change produces a small update payload rather than a full document snapshot.
- **Conflict-free merges**: Updates are commutative and associative, so order does not matter.
- **Awareness**: Presence data (cursor/selection, user identity) is broadcast separately from document updates.
- **Snapshots/compaction**: Over time, updates can be merged or snapshotted to keep history manageable.

Yjs internals (short, practical view):

- **State vectors**: Each client tracks a version vector to compute minimal diffs during sync.
- **Binary updates**: Changes are encoded as compact binary updates that can be merged and rebroadcast.
- **Structs + delete sets**: Inserts/formatting are encoded as structs, deletes are tracked separately, and the two reconcile on apply.
- **Awareness protocol**: Presence is ephemeral and not persisted; it uses a lightweight protocol (`y-protocols`) distinct from document updates.

Collaboration dataflow (conceptual):

```
User A edit
   |
   v
[CRDT update] ---> broadcast ---> other clients
   |                                    |
   |                               apply update
   v                                    v
local state                         converged state
```

In this repo, the collaboration extension integrates with the editor and a realtime provider is used to relay Yjs-style updates and awareness across sessions. The provider also supports threads and inbox notifications used by the comments UI.

## Editor and UI composition

The document view is composed of several cooperating pieces:

- **Layout composition**: `src/app/documents/[documentId]/document.tsx` composes `Navbar`, `Toolbar`, and `Editor` inside a `Room` provider.
- **Realtime room wiring**: `src/app/documents/[documentId]/room.tsx` creates the collaboration room and resolves users and documents for mentions and room info.
- **Editor instance**: `src/app/documents/[documentId]/editor.tsx` configures Tiptap with rich-text extensions and connects it to the CRDT provider.
- **Toolbar commands**: `src/app/documents/[documentId]/toolbar.tsx` reads the editor instance from a shared store to run formatting commands.
- **State sharing**: `src/store/use-editor-store.ts` holds the Tiptap editor instance so the toolbar and editor stay in sync.
- **Presence + comments**: `src/app/documents/[documentId]/avatar.tsx` renders collaborator avatars; `threads.tsx` and `inbox.tsx` render comment threads and notifications.

## Convex + Clerk: data, auth, and tenancy

Convex and Clerk are tightly integrated:

- **Clerk** provides authentication, session claims, and organization membership.
- **Convex** stores document metadata and performs queries/mutations with auth checks.
- **Next.js** uses Clerk server utilities to mint a Convex JWT (`template: "convex"`) for secure preloading.

Where this happens in code:

- `src/components/convex-client-provider.tsx` wires Clerk auth into Convex React client.
- `src/middleware.ts` enforces authentication at the edge.
- `src/app/documents/[documentId]/page.tsx` preloads document data using a Clerk-issued Convex token.
- `convex/documents.ts` enforces ownership/org checks before reads and writes.

Access control logic (simplified):

```
request -> Clerk session -> Convex query/mutation
   |                            |
   |                    check owner or org
   v                            v
allow/deny                   result
```

Realtime session authorization (used before joining a collaborative room):

```
Editor -> POST /api/liveblocks-auth
   |           |
   |           v
   |     Clerk session + Convex doc check
   |           |
   v           v
token ok   401/403
   |
   v
join room + start CRDT sync
```

## Data model

Convex schema (see `convex/schema.ts`):

- **documents**
  - `title`: string
  - `initialContent`: optional string
  - `ownerId`: string (Clerk user id)
  - `organizationId`: optional string
  - indexes:
    - `by_owner_id`
    - `by_organization_id`
    - `search_title` (full-text search on title)

Document list queries use the search index for fast title filtering and support pagination (`convex/documents.ts`).

Note on content storage:

- Convex stores document metadata and optional template content.
- The live collaborative document body is managed by the CRDT layer; persistence strategy can be extended if you want to sync snapshots into Convex.

## Document lifecycle and discovery

Creation and discovery flow:

```
Template click or \"Blank\" -> Convex mutation -> documentId
           |                                   |
           v                                   v
   navigate to /documents/:id        preload document metadata
           |                                   |
           v                                   v
      join CRDT room                 render editor + toolbar
```

Search and pagination details:

- `src/app/(home)/page.tsx` uses `usePaginatedQuery` to load document pages.
- `src/app/(home)/search-input.tsx` and `src/hooks/use-search-param.ts` keep search terms in the URL.
- `convex/documents.ts` routes search to the `search_title` index for fast filtering.
- `src/constants/template.ts` and `src/app/(home)/template-gallery.tsx` define and render the template gallery.

## Metrics and observability

This repo does not ship a full metrics pipeline, but it is ready for instrumentation. Suggested metrics by layer:

- **Editor/CRDT**
  - Update payload size (bytes) and frequency
  - Collaboration latency (edit -> remote apply)
  - Awareness/presence fan-out (active collaborators per doc)
- **Product**
  - Documents created per day
  - Avg. collaborators per document
  - Comment threads created/resolved
- **Backend (Convex)**
  - Query/mutation latency and error rate
  - Search latency and pagination depth
  - Authorization failure counts
- **Auth (Clerk)**
  - Sign-in success rate
  - Org membership changes

Where to observe:

- **Convex dashboard** provides function logs and performance stats.
- **Clerk dashboard** provides authentication activity and user/org analytics.
- **Client metrics** can be shipped to your preferred analytics or APM tool.

## Setup

### Prerequisites

- Node.js 18+ (or the version supported by your Next.js setup)
- A Clerk application
- A Convex project
- A realtime provider key for CRDT sync

### 1) Install dependencies

```
npm install
```

### 2) Configure Clerk

1. Create a Clerk application.
2. Create a JWT template named `convex`.
3. Update `convex/auth.config.ts` with your Clerk instance domain.
4. Add your keys to `.env.local` (see below).

### 3) Configure Convex

1. Run `npx convex dev` and follow the prompts to create or link a project.
2. Convex will generate `.env.local` entries like `CONVEX_DEPLOYMENT` and `NEXT_PUBLIC_CONVEX_URL`.
3. Keep those entries; they are required by the client and server.

### 4) Configure realtime collaboration

Set the secret key for the CRDT provider in `.env.local`. The Next.js auth route uses it to authorize collaboration sessions.

### Environment variables

Create `.env.local` in the project root with:

```
# Convex
CONVEX_DEPLOYMENT=your-convex-deployment
NEXT_PUBLIC_CONVEX_URL=https://your-convex-url

# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...

# Realtime collaboration (CRDT provider)
LIVEBLOCKS_SECRET_KEY=sk_liveblocks_...
```

Notes:

- The Clerk JWT template must be named `convex` to match `getToken({ template: "convex" })`.
- The auth config in `convex/auth.config.ts` must match your Clerk domain.

## Running locally

In one terminal, start Convex:

```
npx convex dev
```

In another terminal, start Next.js:

```
npm run dev
```

Then open `http://localhost:3000`.

## Scripts

- `npm run dev` - start Next.js dev server
- `npm run build` - build production bundle
- `npm run start` - start production server
- `npm run lint` - lint with Next.js rules

## Project layout

- `src/app/(home)` - landing page, templates, document list
- `src/app/documents/[documentId]` - editor, toolbar, room, comments
- `src/app/api/liveblocks-auth/route.ts` - realtime session authorization
- `src/components/convex-client-provider.tsx` - Clerk + Convex client wiring
- `convex/` - schema, queries, and mutations
- `public/` - template thumbnails and assets

## Troubleshooting

- **Unauthorized errors**: verify Clerk keys and JWT template name `convex`.
- **No documents loading**: confirm `NEXT_PUBLIC_CONVEX_URL` and Convex dev server are running.
- **Collaboration not syncing**: check `LIVEBLOCKS_SECRET_KEY` and the auth route response.
- **Org sharing not working**: ensure users belong to the same Clerk organization.
