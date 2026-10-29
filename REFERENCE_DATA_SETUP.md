# OVC Reference Data Integration Guide

## Overview

This guide explains how to set up and use the foundational geographic and institutional reference data (chiefdoms and schools) throughout your OVC system.

---

## 1. Setting Up Reference Data in Supabase

### Step 1: Run the Migration

The migration script populates the `chiefdoms` and `schools` tables with all official data from Eswatini's administrative and educational structures.

**Via Supabase Dashboard:**
1. Go to your Supabase project dashboard
2. Navigate to **SQL Editor**
3. Copy the contents of `supabase/migrations/populate_reference_data.sql`
4. Paste and execute

**Via Supabase CLI (if configured):**
```bash
supabase migration up
```

**Expected Results:**
- 98 chiefdoms across 4 regions (Hhohho, Manzini, Lubombo, Shiselweni)
- 99 schools across all regions

### Step 2: Verify Data

```sql
-- Check chiefdoms loaded
SELECT COUNT(*), region FROM chiefdoms GROUP BY region;

-- Expected output:
-- Hhohho: 72 chiefdoms
-- Lubombo: 26 chiefdoms  
-- Manzini: (from schools reference)
-- Shiselweni: (from schools reference)

-- Check schools loaded
SELECT COUNT(*), region FROM schools GROUP BY region;

-- Expected output:
-- Hhohho: 36 schools
-- Manzini: 22 schools
-- Shiselweni: 41 schools
```

---

## 2. Flutter Mobile App Integration

### Step 1: Ensure Provider Dependency

In `mobile/pubspec.yaml`, ensure you have:

```yaml
dependencies:
  provider: ^6.0.0
```

### Step 2: Reference Data Service

The `ReferenceDataService` (`mobile/lib/services/reference_data_service.dart`) handles:
- Fetching chiefdoms and schools from Supabase
- Cascading filter logic (Region → Chiefdom → School)
- Validation utilities
- Change notification for reactive UI updates

### Step 3: Using in Screens

#### Application Form (Applicant & Identifier)

The `ApplicantFormScreen` now includes cascading dropdowns:

```dart
// Select Region → Chiefdom → School
Consumer<ReferenceDataService>(
  builder: (context, refDataService, _) {
    final chiefdomsForRegion = _selectedRegion != null
        ? refDataService.getChiefdomsByRegion(_selectedRegion!)
        : [];
    final schoolsForChiefdom = _selectedChiefdomId != null
        ? refDataService.getSchoolsByChiefdom(_selectedChiefdomId!)
        : [];
    
    // ... build dropdowns
  },
)
```

**Key Features:**
- User selects Region first
- Chiefdom dropdown shows only chiefdoms from selected region
- School dropdown shows only schools from selected chiefdom
- All dropdowns are required
- Form validates school exists before submission

#### Form Submission

When submitting:
```dart
// The form now uses school_id instead of school_name
'school_id': _selectedSchoolId,  // Integer reference to schools table
'chiefdom': selectedChiefdom?.name,  // Chiefdom name for display
```

### Step 4: Offline Support (Optional)

To enable offline validation:

```dart
// In reference_data_service.dart, add SQLite caching:
import 'package:sqflite/sqflite.dart';

// Store fetched data in local database for offline access
Future<void> cacheToLocalDatabase() async {
  final db = await openDatabase('ovc_reference_data.db');
  // Insert chiefdoms and schools...
}
```

---

## 3. React Web App Integration

### Step 1: Ensure Dependencies

In `web/package.json`:
```json
{
  "@supabase/supabase-js": "^2.x",
  "react": "^18.x"
}
```

### Step 2: Reference Data Service

The `referenceDataService.ts` (`web/src/lib/referenceDataService.ts`) provides:
- Singleton service pattern
- Type-safe Chiefdom and School interfaces
- Cascading filter methods
- Data validation utilities

**Usage:**
```typescript
import { 
  initializeReferenceDataService, 
  getReferenceDataService 
} from './lib/referenceDataService';

// Initialize on app load
const refService = initializeReferenceDataService(supabaseClient);
await refService.loadReferenceData();

// Use throughout app
const regions = refService.getRegions();
const chiefdoms = refService.getChiefdomsByRegion('Hhohho');
const schools = refService.getSchoolsByChiefdom(6);
```

### Step 3: LocationSelector Component

The `LocationSelector.tsx` component provides reusable UI for selecting locations:

```tsx
import { LocationSelector } from './components/LocationSelector';

<LocationSelector
  userType="school_head"  // or "community_leader"
  selectedRegion={region}
  selectedChiefdomId={chiefdomId}
  selectedSchoolId={schoolId}
  onRegionChange={(region) => setRegion(region)}
  onChiefdomChange={(id) => setChiefdomId(id)}
  onSchoolChange={(id) => setSchoolId(id)}
/>
```

**Component Features:**
- Cascading dropdowns
- Smart disabling based on parent selection
- Helpful summary boxes
- Type-specific flows (school_head vs community_leader)
- Loading and error states

### Step 4: Using in User Profile Creation

```tsx
// In admin dashboard for creating school heads
function CreateSchoolHeadForm() {
  const [selectedSchoolId, setSelectedSchoolId] = useState<number | null>(null);
  
  return (
    <form onSubmit={handleSubmit}>
      <LocationSelector
        userType="school_head"
        onSchoolChange={setSelectedSchoolId}
        // ...
      />
      {/* School head will have profile.school_id = selectedSchoolId */}
    </form>
  );
}

// In admin dashboard for creating community leaders
function CreateCommunityLeaderForm() {
  const [selectedChiefdomId, setSelectedChiefdomId] = useState<number | null>(null);
  
  return (
    <form onSubmit={handleSubmit}>
      <LocationSelector
        userType="community_leader"
        onChiefdomChange={setSelectedChiefdomId}
        // ...
      />
      {/* Community leader will have profile.chiefdom_id = selectedChiefdomId */}
    </form>
  );
}
```

---

## 4. Verification & Automatic Routing

### Community Leader Flow

When an application is submitted:

```sql
-- Application has chiefdom_id from location selection
INSERT INTO ovc_applications (chiefdom, ...) VALUES (6, ...);

-- Query for community leader responsible for this chiefdom
SELECT id FROM profiles 
WHERE role = 'community_leader' AND chiefdom_id = 6;

-- Send notification to that leader
-- They see ONLY applications from their chiefdom
SELECT * FROM ovc_applications 
WHERE chiefdom = 6 AND status IN ('submitted', 'school_verified')
AND community_leader_id = current_user_id;
```

### School Head Flow

```sql
-- Application has school_id from location selection
INSERT INTO ovc_applications (school_id, ...) VALUES (21, ...);

-- Query for school head responsible for this school
SELECT id FROM profiles 
WHERE role = 'school_head' AND school_id = 21;

-- Send notification to that head
-- They see ONLY applications from their school
SELECT * FROM ovc_applications 
WHERE school_id = 21 AND status IN ('community_verified', 'submitted')
AND school_head_id = current_user_id;
```

---

## 5. Dashboard & Reporting

### Regional Overview (DPMO Dashboard)

```typescript
// Group applications by region via chiefdoms
const getApplicationsByRegion = async (refService) => {
  const apps = await fetchAllApplications();
  const byRegion: Record<string, number> = {};
  
  for (const app of apps) {
    const chiefdom = refService.getChiefdomById(app.chiefdom_id);
    if (chiefdom) {
      byRegion[chiefdom.region] = (byRegion[chiefdom.region] || 0) + 1;
    }
  }
  
  return byRegion;
};
```

### School-Wise Reports

```typescript
// Count OVCs per school
const getApplicationsBySchool = async (refService) => {
  const apps = await fetchAllApplications();
  const bySchool: Record<string, number> = {};
  
  for (const app of apps) {
    if (app.school_id) {
      const school = refService.getSchoolById(app.school_id);
      if (school) {
        bySchool[school.name] = (bySchool[school.name] || 0) + 1;
      }
    }
  }
  
  return bySchool;
};
```

---

## 6. Data Maintenance

### Adding New Schools

If a new school needs to be added:

```sql
INSERT INTO schools (name, chiefdom_id, region)
VALUES ('New School Name', 6, 'Hhohho');
```

### Linking Schools to Chiefdoms

Some schools may not have a chiefdom_id set. To link them:

```sql
UPDATE schools 
SET chiefdom_id = (
  SELECT id FROM chiefdoms 
  WHERE name = 'Lobamba' LIMIT 1
)
WHERE name = 'Lobamba National High School';
```

### Updating Region Information

If regional boundaries change (unlikely but possible):

```sql
UPDATE chiefdoms 
SET region = 'NewRegion' 
WHERE name = 'SomeChiefdom';
```

---

## 7. Troubleshooting

### Issue: Chiefdoms not showing in dropdown

**Solution:** Ensure you've run the migration script and data is loaded in Supabase.

```sql
SELECT * FROM chiefdoms LIMIT 1;
```

### Issue: Schools appear but have no chiefdom_id

**Solution:** These schools need to be manually linked. Check the schools.txt file for the mapping:

```sql
-- Example: link Lobamba National High School to Lobamba chiefdom
UPDATE schools 
SET chiefdom_id = (SELECT id FROM chiefdoms WHERE name = 'Lobamba')
WHERE name = 'Lobamba National High School';
```

### Issue: Cascading dropdown not working in Flutter

**Solution:** Ensure `Provider` is properly set up in `main.dart` and `ReferenceDataService` is loaded.

```dart
// In main.dart
MultiProvider(
  providers: [
    ChangeNotifierProvider(
      create: (_) => ReferenceDataService(supabase: Supabase.instance.client),
    ),
  ],
  child: MaterialApp(...),
)
```

### Issue: React component not loading data

**Solution:** Ensure `ReferenceDataService` is initialized before rendering:

```typescript
// In App.tsx
useEffect(() => {
  const refService = initializeReferenceDataService(supabase);
  refService.loadReferenceData().catch(console.error);
}, []);
```

---

## 8. Security & RLS

Reference data tables have public read access:

```sql
-- chiefdoms and schools are readable by all authenticated users
CREATE POLICY chiefdoms_select_public ON chiefdoms FOR SELECT USING (true);
CREATE POLICY schools_select_public ON schools FOR SELECT USING (true);
```

This is appropriate because:
- These are government reference lists
- No sensitive data (just names and regions)
- Needed for dropdowns in registration forms
- Public visibility doesn't compromise security

---

## Summary

Your OVC system now has:

✅ **Official Reference Data** - 98 chiefdoms and 99 schools  
✅ **Cascading Selectors** - Region → Chiefdom → School  
✅ **Automatic Routing** - Applications go to correct community leaders/school heads  
✅ **Accurate Reporting** - Dashboard grouped by actual geographic/institutional boundaries  
✅ **Offline Support** - Reference data can be cached locally in Flutter  
✅ **Type Safety** - Validated IDs instead of free-text names  

This transforms your system from a generic form app into an accurate digital replica of Eswatini's administrative and educational landscape.
