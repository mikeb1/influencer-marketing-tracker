# Influencer Marketing Tracker

A comprehensive SaaS application for tracking and managing influencer marketing campaigns, built with modern web technologies.

## 🚀 Tech Stack

### Frontend
- **Core**: React 18, TypeScript, Vite
- **UI Framework**: Tailwind CSS, shadcn/ui, Radix UI, Lucide React
- **State Management**: React Query, Zustand
- **Form Handling**: React Hook Form, Zod

### Backend
- **Database & Auth**: Supabase (PostgreSQL)
- **API**: Supabase REST/GraphQL API
- **Storage**: Supabase Storage

## 🏗️ Architecture

### System Overview

The application follows a modern client-server architecture with Supabase handling the backend responsibilities:

```
┌─────────────────┐      ┌─────────────────┐
│                 │      │                 │
│  React Frontend │◄────►│ Supabase Backend│
│                 │      │                 │
└─────────────────┘      └─────────────────┘
        │                         │
        ▼                         ▼
┌─────────────────┐      ┌─────────────────┐
│  Local State    │      │  PostgreSQL DB  │
│  (Zustand)      │      │  (Supabase)     │
└─────────────────┘      └─────────────────┘
```

### Frontend Architecture

The frontend follows a feature-based architecture with shared UI components:

```
src/
├── components/       # Reusable UI components
├── features/         # Feature-specific components and logic
│   ├── auth/         # Authentication related components
│   ├── campaigns/    # Campaign management
│   ├── dashboard/    # Dashboard and analytics
│   ├── influencers/  # Influencer management
│   └── settings/     # User and organization settings
├── hooks/            # Custom React hooks
├── lib/              # Utility functions and services
│   └── supabase/     # Supabase client and helpers
├── types/            # TypeScript type definitions
└── App.tsx           # Main application component
```

## 📊 Database Schema

The application uses multiple schemas in Supabase for better organization:

### `public` Schema

```sql
-- User profiles (extends auth.users)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users,
  full_name TEXT,
  avatar_url TEXT,
  organization_id UUID REFERENCES organizations,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Organizations (for multi-tenant support)
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  logo_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Organization members
CREATE TABLE organization_members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID REFERENCES organizations NOT NULL,
  user_id UUID REFERENCES auth.users NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(organization_id, user_id)
);
```

### `marketing_data` Schema

```sql
-- Influencers
CREATE TABLE marketing_data.influencers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID REFERENCES public.organizations NOT NULL,
  name TEXT NOT NULL,
  email TEXT,
  phone TEXT,
  social_handles JSONB,
  categories TEXT[],
  audience_demographics JSONB,
  notes TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Campaigns
CREATE TABLE marketing_data.campaigns (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID REFERENCES public.organizations NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  start_date DATE,
  end_date DATE,
  budget NUMERIC,
  status TEXT CHECK (status IN ('draft', 'active', 'paused', 'completed')),
  goals JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Campaign-Influencer relationships
CREATE TABLE marketing_data.campaign_influencers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  campaign_id UUID REFERENCES marketing_data.campaigns ON DELETE CASCADE NOT NULL,
  influencer_id UUID REFERENCES marketing_data.influencers ON DELETE CASCADE NOT NULL,
  status TEXT CHECK (status IN ('invited', 'negotiating', 'confirmed', 'active', 'completed', 'declined')),
  compensation JSONB,
  deliverables JSONB,
  tracking_links JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(campaign_id, influencer_id)
);

-- Content
CREATE TABLE marketing_data.content (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  campaign_influencer_id UUID REFERENCES marketing_data.campaign_influencers ON DELETE CASCADE NOT NULL,
  platform TEXT NOT NULL,
  content_type TEXT NOT NULL,
  url TEXT,
  scheduled_date TIMESTAMP WITH TIME ZONE,
  published_date TIMESTAMP WITH TIME ZONE,
  status TEXT CHECK (status IN ('planned', 'scheduled', 'published', 'archived')),
  metrics JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## 🔐 Authentication Flow

The application uses Supabase Auth for authentication with the following flow:

1. User signs up/logs in via Supabase Auth
2. A trigger creates a corresponding entry in the `public.profiles` table
3. Row Level Security (RLS) policies ensure users can only access their organization's data
4. JWT tokens are used for authentication with the Supabase API

### RLS Policies

```sql
-- Example RLS policy for campaigns
CREATE POLICY campaign_organization_policy
ON marketing_data.campaigns
FOR ALL
USING (
  organization_id IN (
    SELECT organization_id FROM public.organization_members
    WHERE user_id = auth.uid()
  )
);
```

## 🎯 Key Features

### 1. Influencer Management
- Influencer profiles with contact info and social handles
- Performance metrics and historical data
- Categorization and tagging

### 2. Campaign Management
- Campaign creation and planning
- Budget allocation and tracking
- Timeline and milestone management

### 3. Content Tracking
- Content calendar and scheduling
- Performance metrics (views, engagement, etc.)
- Content approval workflow

### 4. Analytics Dashboard
- Campaign performance metrics
- ROI calculation
- Influencer performance comparison

### 5. Reporting
- Customizable reports
- Export functionality (CSV, PDF)
- Scheduled report delivery

## 🖥️ UI Components

### Dashboard Layout

```tsx
// src/components/layout/DashboardLayout.tsx
import { Outlet } from 'react-router-dom';
import { Sidebar } from './Sidebar';
import { Header } from './Header';

export function DashboardLayout() {
  return (
    <div className="flex h-screen overflow-hidden">
      <Sidebar />
      <div className="flex flex-col flex-1 overflow-hidden">
        <Header />
        <main className="flex-1 overflow-y-auto p-6 bg-gray-50">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

### Data Table Component

```tsx
// src/components/ui/data-table/DataTable.tsx
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { useState } from 'react';

interface DataTableProps<T> {
  data: T[];
  columns: {
    header: string;
    accessorKey: keyof T;
    cell?: (item: T) => React.ReactNode;
  }[];
  onRowClick?: (item: T) => void;
}

export function DataTable<T>({ data, columns, onRowClick }: DataTableProps<T>) {
  const [searchTerm, setSearchTerm] = useState('');
  
  // Filter data based on search term
  const filteredData = data.filter((item) =>
    Object.values(item).some(
      (value) =>
        value &&
        value.toString().toLowerCase().includes(searchTerm.toLowerCase())
    )
  );

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <Input
          placeholder="Search..."
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
          className="max-w-sm"
        />
      </div>
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            <TableRow>
              {columns.map((column) => (
                <TableHead key={column.header}>{column.header}</TableHead>
              ))}
            </TableRow>
          </TableHeader>
          <TableBody>
            {filteredData.map((item, i) => (
              <TableRow
                key={i}
                onClick={() => onRowClick?.(item)}
                className={onRowClick ? 'cursor-pointer hover:bg-gray-50' : ''}
              >
                {columns.map((column) => (
                  <TableCell key={column.header}>
                    {column.cell
                      ? column.cell(item)
                      : (item[column.accessorKey] as React.ReactNode)}
                  </TableCell>
                ))}
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>
    </div>
  );
}
```

## 📅 Development Roadmap

### Phase 1: Project Setup and Authentication
- Initialize project with Vite, React, and TypeScript
- Set up Supabase project and database schema
- Implement authentication flow
- Create basic UI components with shadcn/ui

### Phase 2: Core Features
- Implement influencer management
- Build campaign management functionality
- Create content tracking features
- Develop basic analytics dashboard

### Phase 3: Advanced Features
- Implement reporting functionality
- Add data visualization components
- Create export functionality
- Implement notification system

### Phase 4: Optimization and Deployment
- Performance optimization
- Comprehensive testing
- Documentation
- Production deployment

## 🧪 Getting Started

### Prerequisites
- Node.js (v18+)
- npm or yarn
- Supabase account

### Installation

1. Clone the repository
```bash
git clone https://github.com/yourusername/influencer-marketing-tracker.git
cd influencer-marketing-tracker
```

2. Install dependencies
```bash
npm install
# or
yarn install
```

3. Set up environment variables
```bash
cp .env.example .env.local
# Edit .env.local with your Supabase credentials
```

4. Start the development server
```bash
npm run dev
# or
yarn dev
```

## 📚 Resources

- [Supabase Documentation](https://supabase.com/docs)
- [React Documentation](https://reactjs.org/docs/getting-started.html)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [Vite Documentation](https://vitejs.dev/guide/)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
- [shadcn/ui Documentation](https://ui.shadcn.com/docs)
