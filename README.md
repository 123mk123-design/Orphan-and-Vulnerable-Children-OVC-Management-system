# Orphans & Vulnerable Children (OVC) Education Grant System
### Deputy Prime Minister’s Office (DPMO) — Kingdom of Eswatini

A full-stack digital management and verification platform developed for the Deputy Prime Minister’s Office (DPMO) of Eswatini. The system coordinates the identification, multi-stage sequential verification, approval, and funding of education grants for vulnerable children across national chiefdoms and schools.

---

## System Architecture

- **Web Application**: React 18, TypeScript, Vite, Chart.js (Used by DPMO Administrators & School Heads)
- **Mobile Application**: Flutter, Material 3, Provider, Supabase Flutter (Used by Guardians, Community Leaders, Identifiers, and Caseworkers)
- **Backend & Cloud Database**: Supabase (PostgreSQL 15, PostGIS, Row-Level Security, Storage, Realtime, Edge Functions)
- **Automated Messaging**: Multi-stage sequential pipeline with Twilio SMS Gateway & Resend Email triggers

---

## Repository Structure

```text
OVCProject/
├── web/                 # React + TypeScript administrative web dashboard
│   ├── src/             # Frontend source code (routes, components, context, services)
│   ├── package.json     # Node dependencies & scripts (dev, build, lint)
│   └── .env.example     # Web environment configuration template
├── mobile/              # Flutter cross-platform mobile application
│   ├── lib/             # Mobile source code (screens, services, widgets, theme)
│   └── pubspec.yaml     # Flutter package specifications
├── supabase/            # Database schema & backend infrastructure
│   ├── schema.sql       # Canonical PostgreSQL database schema
│   ├── migrations/      # Historical migrations & schema versions
│   └── functions/       # Edge functions (send-sms, invite-school-head, suspend-user)
├── scripts/             # Administrative data seeding and testing utilities
├── manual/              # System screenshots and walkthrough assets
└── USER_MANUAL.md       # Illustrated User Manual
```

---

## Getting Started

### 1. Database Setup (Supabase)
1. Create a project at [supabase.com](https://supabase.com).
2. Open the **SQL Editor** in your Supabase Dashboard.
3. Run the database schema located at `supabase/schema.sql`.
4. Create a private Storage bucket named `ovc-documents`.
5. *(Optional)* Deploy Supabase Edge Functions:
   ```bash
   supabase functions deploy send-sms
   supabase functions deploy invite-school-head
   supabase functions deploy suspend-user
   ```

---

### 2. Web Application Setup
The web portal serves national DPMO administrators and school head teachers.

```bash
# Navigate to web directory
cd web

# Install dependencies
npm install

# Copy environment variables and configure credentials
cp .env.example .env

# Run development server
npm run dev

# Run code linter
npm run lint

# Build production bundle
npm run build
```
The web dashboard runs locally at: `http://localhost:5173/`

---

### 3. Mobile Application Setup
The mobile app serves Guardians/Applicants, Community Leaders, and Social Workers.

```bash
# Navigate to mobile directory
cd mobile

# Fetch Flutter dependencies
flutter pub get

# Run static code analysis
flutter analyze

# Launch mobile application (Emulator or connected device)
flutter run
```

---

### 4. Administrative Seed Scripts
To seed reference data, school head accounts, or community leader accounts:

```bash
# Set your Supabase Service Role Key
$env:SUPABASE_SERVICE_ROLE_KEY="your-service-role-key"  # PowerShell
# export SUPABASE_SERVICE_ROLE_KEY="your-service-role-key" # Bash

# Seed Community Leaders across all chiefdoms
node scripts/seed_community_leaders.mjs

# Seed School Head accounts
node scripts/seed_school_heads.js

# Seed SMS notification templates
node scripts/seed_sms_templates.mjs
```
See `scripts/README.md` for detailed script documentation.

---

## Multi-Stage Verification Pipeline & Roles

| Role | Interface | Primary Responsibility |
| :--- | :--- | :--- |
| **Applicant / Guardian** | Mobile | Submits child profile, poverty indicators, and uploaded affidavits |
| **Community Leader** | Mobile | Verifies child residency within the Chiefdom |
| **School Head Teacher** | Web / Mobile | Confirms school enrollment, fee structure, and academic standing |
| **DPMO Administrator** | Web Portal | Evaluates vulnerability scoring, approves grants, and disburses funds |
| **Social Worker / Caseworker**| Mobile | CCIMS case management, home visits, PSS sessions, and welfare notes |

---

## Documentation
- **User Manual**: See `USER_MANUAL.md` for role guides with system screenshots.
- **Reference Data Setup**: See `REFERENCE_DATA_SETUP.md` for Eswatini regional and chiefdom mappings.
- **Utility Scripts**: See `scripts/README.md` for database seeding and SMS verification.
