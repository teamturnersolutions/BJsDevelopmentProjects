# BJsDevelopmentProjects

Local SaaS & POC container examples

## 🚀 Overview

This repository contains experimental POCs built locally and containerized using Docker.

## 🛠 Prerequisites

Install:
* Docker 20+
* Docker Compose

## 📦 Quick Start

```bash
git clone https://github.com/teamturnersolutions/BJsDevelopmentProjects.git
cd BJsDevelopmentProjects

docker compose up --build
```

# InvTrack Pro - Technical Documentation & Roadmap

This document provides a deep dive into the **InvTrack Pro** codebase, explaining its architecture, component organization, and providing a step-by-step guide for migrating from a CSV-based data store to a robust **SQLite** database.

---

## 🏗️ Architecture Overview

InvTrack Pro is built using **Next.js 15** with the **App Router**, utilizing Server Components for rendering and Server Actions for data mutations.

### 📁 Project Structure

```text
├── src/
│   ├── app/                # Next.js App Router (Pages, Layouts, Actions)
│   │   ├── (app)/          # Main application routes (Dashboard, etc.)
│   │   │   ├── audit/      # Equipment audit workflow
│   │   │   ├── checkin/    # Equipment return/check-in workflow
│   │   │   ├── checkout/   # Equipment checkout workflow
│   │   │   └── inventory/  # Gallery view of all equipment
│   │   ├── actions.ts      # Server Actions (Business logic for mutations)
│   │   └── layout.tsx      # Root layout (Navigation, Metadata)
│   ├── components/         # Reusable React components
│   │   ├── ui/             # Radix UI / Shadcn based components (Lower level)
│   │   ├── inventory-card.tsx # Visual card for equipment items
│   │   └── app-header.tsx  # Global navigation header
│   ├── lib/                # Shared utilities and data access
│   │   ├── data.ts         # Data Access Layer (Currently CSV-based)
│   │   ├── types.ts        # TypeScript interfaces for Items/Members
│   │   └── utils.ts        # Formatting and Tailwind helpers
├── public/                 # Static assets (Not present, created during build)
├── inventory.csv           # Primary data file for equipment
├── team-members.csv        # Primary data file for team members
├── Dockerfile              # Multi-stage production build configuration
└── docker-compose.yml      # Orchestration for local dev and deployment
```

---

## 🧩 Component Breakdown

### 1. Data Access Layer (`src/lib/data.ts`)
This is the heart of the application's data handling. It currently uses `fs` to read and write to CSV files.
- `getInventoryItems()`: Loads all equipment.
- `updateInventory()`: Merges updates back into the CSV.
- `getTeamMembers()`: Loads the list of personnel.

### 2. Server Actions (`src/app/actions.ts`)
Encapsulates business logic that runs on the server.
- `checkOutEquipment`: Validates status, updates the item with the member's name/ID, and triggers a revalidation of the UI.
- `checkInEquipment`: Resets the item status to 'Available' and clears assignment data.

### 3. UI Components (`src/components/`)
- **Inventory Card**: Responsive card showing item thumbnail, status badge, and assignment details. It uses conditional styling based on the `status` field.
- **UI Directory**: Contains specialized components like Buttons, Dialogs, and Select menus (standard Shadcn-style architecture).

---

## 🗄️ SQLite Integration Guide

To move from CSV to a relational database like **SQLite**, follow these steps:

### Phase 1: Setup
1.  **Install dependencies**:
    ```bash
    npm install better-sqlite3
    npm install -D @types/better-sqlite3
    ```

### Phase 2: Schema Design
Create a database initialization script (`src/lib/db.ts`):
```typescript
import Database from 'better-sqlite3';

const db = new Database('inventory.db');

// Initialize tables
db.exec(`
  CREATE TABLE IF NOT EXISTS team_members (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL
  );

  CREATE TABLE IF NOT EXISTS inventory (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    status TEXT DEFAULT 'Available',
    checkedOutBy TEXT,
    checkedOutById TEXT,
    checkedOutDate TEXT,
    checkedInDate TEXT,
    FOREIGN KEY(checkedOutById) REFERENCES team_members(id)
  );
`);

export default db;
```

### Phase 3: Data Migration
Update `src/lib/data.ts` to use `db` instead of `fs`.
**Example Refactor for `getInventoryItems`**:
```typescript
import db from './db';

export async function getInventoryItems(): Promise<InventoryItem[]> {
  const stmt = db.prepare('SELECT * FROM inventory');
  return stmt.all() as InventoryItem[];
}
```

### Phase 4: Docker Updates
Update `docker-compose.yml` to persist the database file using a volume:
```yaml
volumes:
  - ./inventory.db:/app/inventory.db
```

---

## 🎯 Summary of Workflow
1.  **Request**: User selects items and identifies themselves.
2.  **Action**: `checkOutEquipment` called with IDs.
3.  **Library**: `updateInventory` writes to storage (CSV/SQL).
4.  **UI**: `revalidatePath` forces Next.js to refresh the data on screen.
