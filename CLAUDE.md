# CLAUDE.md — Behe Constructions Fitout Management Application

## Project Overview

Build a full-stack web application for **Behe Constructions Pty Ltd**, a Sydney-based shop fitout company. The app enables their internal team (up to 5 users) to upload architectural plans, extract material quantities using AI, manage project documents, generate supplier emails, export material schedules to Excel, and produce scope-of-works quotation documents — all within a branded, professional dashboard.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 (App Router) |
| Backend/API | Next.js API Routes (Route Handlers) |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth (email/password) |
| File Storage | Supabase Storage |
| AI Analysis | Anthropic Claude API (`claude-opus-4-5` or `claude-sonnet-4-6`) |
| PDF Processing | `pdf2pic` + `sharp` (convert PDF pages to images for Claude vision) |
| DWG/AI Conversion | `LibreOffice` CLI or `cloudconvert` API (convert DWG/AI → PDF before processing) |
| Excel Export | `exceljs` |
| Styling | Tailwind CSS |
| Deployment | Vercel (frontend) + Supabase (hosted) |

---

## Brand & Design System

```
Primary CTA Background:  #393536  (dark charcoal)
Primary CTA Text:        #e5b96b  (warm gold)
Logo:                    Top-left, clickable → returns to dashboard
Navigation:              Left sidebar on desktop, top bar on mobile
Font:                    Inter (clean, professional)
```

### Color Variables (add to `tailwind.config.js`)
```js
colors: {
  behe: {
    dark:  '#393536',
    gold:  '#e5b96b',
    light: '#f5f0e8',
    gray:  '#6b7280',
  }
}
```

### CTA Component Pattern
All primary buttons use: `bg-[#393536] text-[#e5b96b] hover:opacity-90 font-semibold px-6 py-2 rounded`

---

## Application Architecture

```
/app
  /dashboard              ← All projects overview
  /projects/[id]          ← Individual project dashboard
  /projects/[id]/plans    ← Plan upload & version management
  /projects/[id]/quantities ← AI-extracted quantities + user editing
  /projects/[id]/emails   ← Supplier email drafts
  /projects/[id]/excel    ← Excel export
  /projects/[id]/scope    ← Scope of works / quotation document
  /settings               ← Suppliers, materials config, users
  /login                  ← Auth page

/api
  /projects               ← CRUD
  /projects/[id]/upload   ← File upload + conversion pipeline
  /projects/[id]/analyse  ← Trigger Claude AI analysis
  /projects/[id]/quantities ← GET/PATCH quantities
  /projects/[id]/export-excel ← Generate and download Excel
  /projects/[id]/export-scope ← Generate scope of works PDF/Word
  /suppliers              ← CRUD for supplier list
  /materials              ← CRUD for material config
```

---

## Database Schema (Supabase PostgreSQL)

### `users` (managed by Supabase Auth)
- Supabase handles this automatically

### `projects`
```sql
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  name TEXT NOT NULL,                    -- e.g. "Maharaja's Indian Restaurant"
  client_name TEXT,
  location TEXT,                         -- e.g. "Showground, Castle Hill"
  designer TEXT,                         -- e.g. "Lexis Design"
  required_date DATE,
  status TEXT DEFAULT 'active',          -- active | completed | archived
  cover_image_url TEXT,                  -- manual upload or auto-extracted
  created_by UUID REFERENCES auth.users(id)
);
```

### `plan_versions`
```sql
CREATE TABLE plan_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,       -- 1, 2, 3...
  label TEXT,                            -- e.g. "Issue C", "Rev 2"
  file_url TEXT NOT NULL,                -- Supabase Storage path
  original_filename TEXT,
  file_type TEXT,                        -- pdf | dwg | ai
  converted_pdf_url TEXT,                -- Always stored after conversion
  uploaded_at TIMESTAMPTZ DEFAULT NOW(),
  uploaded_by UUID REFERENCES auth.users(id),
  analysis_status TEXT DEFAULT 'pending' -- pending | processing | complete | failed
);
```

### `quantity_extractions`
```sql
CREATE TABLE quantity_extractions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_version_id UUID REFERENCES plan_versions(id) ON DELETE CASCADE,
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  material_id UUID REFERENCES materials(id),
  material_name TEXT NOT NULL,
  ai_quantity NUMERIC,                   -- Claude's raw extraction
  ai_unit TEXT,                          -- m², LM, each, etc.
  ai_confidence TEXT,                    -- high | medium | low
  ai_notes TEXT,                         -- Claude's reasoning/caveats
  user_quantity NUMERIC,                 -- User-corrected value (null = accept AI)
  user_unit TEXT,
  wastage_pct NUMERIC DEFAULT 0.10,      -- e.g. 0.10 = 10%
  final_quantity NUMERIC,                -- GENERATED: user_quantity ?? ai_quantity * (1 + wastage_pct)
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### `materials`
```sql
CREATE TABLE materials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  category TEXT,                         -- Tiling | Plastering | Painting | Flooring | etc.
  default_unit TEXT,                     -- m², LM, each, sheets
  default_wastage_pct NUMERIC,           -- e.g. 0.10
  sort_order INTEGER
);

-- Seed with Behe's core materials:
INSERT INTO materials (name, category, default_unit, default_wastage_pct, sort_order) VALUES
  ('Floor Tiles', 'Tiling', 'm²', 0.10, 1),
  ('Wall Tiles', 'Tiling', 'm²', 0.10, 2),
  ('Tile Adhesive', 'Tiling', 'kg', 0.10, 3),
  ('Tile Grout', 'Tiling', 'kg', 0.05, 4),
  ('10mm Plasterboard', 'Plastering', 'sheets', 0.10, 5),
  ('13mm Plasterboard', 'Plastering', 'sheets', 0.10, 6),
  ('Furring Channels', 'Plastering', 'LM', 0.10, 7),
  ('Steel Stud Framing', 'Plastering', 'LM', 0.10, 8),
  ('Cornice', 'Plastering', 'LM', 0.05, 9),
  ('Base Coat Compound', 'Plastering', 'bags', 0.10, 10),
  ('Finish Coat Compound', 'Plastering', 'bags', 0.10, 11),
  ('Jointing Tape', 'Plastering', 'rolls', 0.05, 12),
  ('Wall Paint', 'Painting', 'L', 0.15, 13),
  ('Ceiling Paint', 'Painting', 'L', 0.15, 14),
  ('Timber Flooring', 'Flooring', 'm²', 0.10, 15),
  ('Skirting Board', 'Flooring', 'LM', 0.10, 16),
  ('Vinyl Flooring', 'Flooring', 'm²', 0.10, 17),
  ('Carpet', 'Flooring', 'm²', 0.10, 18),
  ('Timber Slats (Ceiling)', 'Joinery', 'LM', 0.10, 19),
  ('Structural Timber', 'Structural', 'LM', 0.10, 20);
```

### `suppliers`
```sql
CREATE TABLE suppliers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  contact_name TEXT,
  email TEXT,
  phone TEXT,
  material_categories TEXT[],            -- array: ['Tiling', 'Flooring']
  is_default BOOLEAN DEFAULT false,
  notes TEXT
);
```

### `project_suppliers` (per-project supplier overrides)
```sql
CREATE TABLE project_suppliers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  supplier_id UUID REFERENCES suppliers(id),
  material_category TEXT,
  override_contact TEXT,
  override_email TEXT
);
```

### `email_drafts`
```sql
CREATE TABLE email_drafts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  supplier_id UUID REFERENCES suppliers(id),
  material_category TEXT,
  subject TEXT,
  body TEXT,
  status TEXT DEFAULT 'draft',           -- draft | sent
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Feature 1: File Upload & Conversion Pipeline

### Supported File Types
- `.pdf` → Store directly
- `.dwg` → Convert to PDF via LibreOffice or CloudConvert API, then store
- `.ai` → Convert to PDF (LibreOffice or Ghostscript), then store

### Upload Flow
1. User drags/drops file onto project upload area
2. Frontend: POST to `/api/projects/[id]/upload` with multipart form
3. Backend:
   - Detect file type
   - If DWG/AI: run conversion → save converted PDF to Supabase Storage
   - Store original file AND converted PDF paths in `plan_versions`
   - Auto-increment `version_number` for the project
   - Extract first page as image → offer as cover photo if project has none
4. Return `plan_version_id` to frontend
5. Frontend auto-triggers AI analysis (or user clicks "Analyse Plans")

### Cover Photo Logic
- On upload, extract page 1 of PDF as a 1200px JPEG using `pdf2pic`
- If `projects.cover_image_url` is null → auto-set to this extracted image
- User can always replace with a manual upload
- Show both options in UI: "Use extracted preview" or "Upload cover photo"

---

## Feature 2: AI Material Quantity Extraction

### Claude Vision Prompt Strategy

Convert ALL pages of the uploaded PDF to images (max 2048px width) and send them to Claude with the following structured prompt:

```
SYSTEM PROMPT:
You are an expert quantity surveyor specialising in Australian commercial shop fitouts. 
You analyse architectural and fitout drawings to extract precise material quantities.
Always look for scale annotations (e.g. 1:50, 1:100) and dimension markings in mm or metres.
Calculate areas in m², linear lengths in LM (linear metres), and counts as 'each'.
Be conservative — only extract quantities you can reasonably calculate from visible dimensions.
Return your response as valid JSON only.

USER PROMPT:
Analyse the attached architectural drawings for [PROJECT NAME] at [LOCATION].
Extract quantities for ALL of the following materials where visible in the plans:
[LIST ALL MATERIALS FROM materials TABLE]

For each material found, return:
- material_name: exact name from the provided list
- quantity: numeric value (raw, before wastage)
- unit: m² | LM | sheets | each | L | kg | bags | rolls
- confidence: "high" | "medium" | "low"
- calculation_notes: brief explanation of how you calculated it (e.g. "Floor area = 8.4m x 12.2m = 102.5m²")
- relevant_drawing_reference: which drawing/page contained this info

Also return:
- scale_detected: the scale annotation found (e.g. "1:50")
- overall_confidence: "high" | "medium" | "low"  
- analyst_notes: any important caveats or assumptions made

Return ONLY valid JSON matching this structure:
{
  "scale_detected": "1:50",
  "overall_confidence": "medium",
  "analyst_notes": "...",
  "materials": [
    {
      "material_name": "Floor Tiles",
      "quantity": 102.5,
      "unit": "m²",
      "confidence": "high",
      "calculation_notes": "Floor area 8.4m x 12.2m = 102.5m²",
      "relevant_drawing_reference": "Sheet A02 - Floor Plan"
    }
  ]
}
```

### AI Analysis API Route (`/api/projects/[id]/analyse`)
1. Fetch `plan_version_id` from request
2. Download PDF from Supabase Storage
3. Convert each page to image using `pdf2pic`
4. Send all page images to Claude API with above prompt
5. Parse JSON response
6. For each material in response:
   - Match to `materials` table by name
   - Insert into `quantity_extractions` with `ai_quantity`, `ai_unit`, `ai_confidence`, `ai_notes`
   - Set `wastage_pct` from `materials.default_wastage_pct`
   - Calculate `final_quantity = ai_quantity * (1 + wastage_pct)`
7. Update `plan_versions.analysis_status = 'complete'`
8. Return extraction results to frontend

### User Correction UI
- Show a table of all extracted materials
- Each row: Material Name | AI Quantity | Unit | Confidence Badge | Wastage % | Final Qty | Edit button
- Confidence badge: 🟢 High / 🟡 Medium / 🔴 Low
- User can click any row to edit `user_quantity` and `user_unit`
- When user saves edit: update `quantity_extractions.user_quantity`, recalculate `final_quantity`
- Show AI's `calculation_notes` as a tooltip/expandable detail
- "Accept All AI Values" button at top for efficiency

---

## Feature 3: Version Control & Quantity Diff

### Version History Panel
- Sidebar or tab on the project showing all `plan_versions` ordered by `version_number`
- Each version shows: version number, label, upload date, uploaded by, analysis status
- User can click any version to view its extracted quantities

### Version Comparison (Quantity Diff)
When a new version is uploaded and analysed, automatically generate a diff:

```
// Comparison logic (backend)
For each material:
  v_prev = quantity from previous version (final_quantity)
  v_new  = quantity from new version (final_quantity)
  change = v_new - v_prev
  pct    = (change / v_prev) * 100
```

Display as a table:
| Material | v1 | v2 | Change | % |
|---|---|---|---|---|
| Floor Tiles | 102.5 m² | 118.0 m² | +15.5 m² | +15.1% |
| Wall Paint | 85 L | 85 L | — | — |

Colour code: green for decreases, red for increases, grey for no change.

---

## Feature 4: Supplier Email Drafts

### Email Format (match Behe's exact tone)

```
Subject: Quote Request – [MATERIAL CATEGORY] – [PROJECT NAME]

Hi [SUPPLIER CONTACT NAME] Team,

I hope you're well.

Could you please provide a quote and lead time/stock availability for the following materials for our upcoming project?

Project Name: [PROJECT NAME]
Project Location: [PROJECT LOCATION]
Designer: [DESIGNER NAME]
Required On-Site Date: [REQUIRED DATE]

Materials Required:
[TABLE: Material Name | Quantity | Unit]

Please don't hesitate to reach out if you need any further information or specifications.

Looking forward to your response.

Kind regards,
[USER FULL NAME]
Behe Constructions Pty Ltd
ABN: 32 662 000 759
Email: brian.he@behe.com.au
Address: 27/398 Marion Street, Condell Park NSW 2200
```

### Email Generation Logic
1. Group `quantity_extractions` by material category
2. For each category, find the assigned supplier (project override → default supplier)
3. Generate one email draft per supplier per category
4. Save to `email_drafts` table
5. User can edit the draft inline before copying/sending
6. "Copy to Clipboard" button on each draft
7. "Mark as Sent" button to update status

---

## Feature 5: Excel Export

### Format: Single sheet, all materials

Using `exceljs`, generate a workbook that mirrors Behe's existing `Project_Estimation` template style:

**Sheet: "Materials Schedule"**

| # | Category | Material | Quantity | Unit | Wastage % | Qty incl. Wastage | Unit Cost | Subtotal | Markup % | Markup Amount | Total | Comments |
|---|---|---|---|---|---|---|---|---|---|---|---|---|

- Row headers styled with `#393536` background and `#e5b96b` text
- Category rows (e.g. "Tiling", "Plastering") as bold section headers with light gold background
- Quantity formulas: `=D2*(1+E2)` (keep as Excel formulas, not hardcoded values)
- Summary rows per category with `SUM` formulas
- Grand total row at bottom
- Column widths set appropriately
- Font: Arial 10pt throughout
- Auto-filter enabled on header row
- Include project name, location, date, prepared by in top header block

**Download endpoint:** `GET /api/projects/[id]/export-excel`
- Stream the `.xlsx` file as a download
- Filename: `[Project Name] - Materials Schedule - v[version].xlsx`

---

## Feature 6: Scope of Works / Quotation Document

### Based on Behe's Exact Quotation Template

The scope of works document must closely follow the structure of `Quotation - ABC Company (Site).xlsx`. Generate this as a formatted Excel file (`.xlsx`) using `exceljs` that users can then open and print/PDF themselves.

**Document Structure:**

```
BEHE CONSTRUCTIONS PTY LTD
ABN: 32 662 000 759
Email: brian.he@behe.com.au
Address: 27/398 Marion Street, Condell Park NSW 2200
Prepared By: [USER NAME] | [USER PHONE] | [USER EMAIL]
Date: [AUTO DATE]
Project: [PROJECT NAME]
Location: [PROJECT LOCATION]
Attn: [CLIENT NAME]
QUOTE: [AUTO-INCREMENTED QUOTE NUMBER]
Ref: QUOTATION – Shop Fit Out

Hi [CLIENT NAME],

We, Behe Constructions Pty Ltd (The Contractor) hereby issue an estimated quotation 
for the shop fit out work of [COMPANY NAME] at [LOCATION] as requested.

                        SCOPE OF WORK

[AI GENERATES RELEVANT SECTIONS FROM BELOW LIST BASED ON PLAN ANALYSIS]

1. General Conditions
   1.1 Provision for shopping centre induction fee expenses.
   1.2 Provision for contractor travel and parking expenses.
   ... [all standard items]

2. Demolition (if applicable)
3. Plastering, Ceiling and Walls (if applicable)
4. Structural Elements (if applicable)
5. Electrical and Data (if applicable)
6. Plumbing (if applicable)
7. Painting (if applicable)
8. Flooring and Tiling (if applicable)
9. Stainless Steel (if applicable)
10. Glazing (if applicable)
11. Powdercoating and 2pac (if applicable)
12. Refrigeration (if applicable)
13. Joinery (if applicable)
14. Masonry (if applicable)
15. Metals, Steel and Aluminium (if applicable)
16. Signage (if applicable)
17. Access (if applicable)
18. Site and Waste Management

                        PRICING
Total Fit-Out Cost Excl GST:  $[VALUE]
GST (10%):                    $[VALUE]
TOTAL FIT-OUT COST INCL GST:  $[VALUE]

Terms and Conditions:
Note 1: Exclusions
  [Standard exclusions list from template]

Note 2: Approved Drawings
  All above work apply as per approved drawing issued by [DESIGNER]. 
  Drawing No. [REF], Issue [ISSUE].

Note 3: Base Building Services [standard text]
Note 4: Payments [standard payment schedule text]
Note 5: Provision of Service [standard text]
Note 6: Contract Value and Payment Terms [standard text]
```

### AI Scope Generation Logic
After plan analysis, use Claude to:
1. Determine which trade sections are RELEVANT to this specific project
2. For each relevant section, customize the line items with project-specific details (e.g. replace "X of sinks" with "3 of sinks")
3. Return a structured JSON of applicable scope items
4. Populate the quotation template accordingly

**Prompt for scope generation:**
```
Based on the material quantities extracted from these shop fitout plans, 
generate a detailed scope of works in the style of an Australian commercial fitout contractor.

Project: [NAME] | Location: [LOCATION] | Designer: [DESIGNER]

Extracted materials and quantities: [JSON OF QUANTITIES]

Using the following standard sections, include ONLY sections that are applicable 
to this project based on the extracted quantities. For each applicable section, 
list specific line items with actual quantities substituted where "[X]" appears.

[PASTE FULL LIST OF SCOPE SECTIONS FROM TEMPLATE]

Return as JSON: { "applicable_sections": [ { "number": 1, "title": "...", "items": ["1.1 ...", "1.2 ..."] } ] }
```

---

## Feature 7: Project Dashboard

### Dashboard Homepage (`/dashboard`)
- Grid of project cards (3 columns desktop, 1 mobile)
- Each card shows:
  - Cover photo (extracted from plans OR manual upload)
  - Project Name (large)
  - Client name + Location
  - Status badge: Active / Completed / Archived
  - Last updated date
  - Quick action buttons: View | Upload New Plans
- Top bar: Behe logo (left) + "New Project" CTA button (right, `#393536` / `#e5b96b`)
- Left sidebar: Dashboard | Projects | Settings | Logout

### Individual Project Dashboard (`/projects/[id]`)
- Cover photo hero image (full width, ~200px tall)
- Project title + metadata (client, location, designer, required date)
- Tab navigation: Overview | Plans & Versions | Quantities | Emails | Scope | Export
- **Overview tab:**
  - Quick stats: Total materials extracted, # suppliers, last plan version, last updated
  - Version history timeline
  - Recent activity log

### New Project Creation Modal
Fields:
- Project Name (required)
- Client Name
- Project Location (address)
- Designer / Architect
- Required On-Site Date
- Upload first set of plans (or skip)
- Cover photo (optional, auto-set from plans)

---

## Authentication

### Setup (Supabase Auth)
- Email + password login only
- No self-registration — admin creates accounts in Supabase dashboard
- Protect all routes with middleware: redirect to `/login` if not authenticated
- Store user session in Supabase Auth cookies (Next.js middleware)
- Show user name + logout in sidebar footer

### Next.js Middleware (`middleware.ts`)
```ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'
import { NextResponse } from 'next/server'

export async function middleware(req) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })
  const { data: { session } } = await supabase.auth.getSession()
  if (!session && !req.nextUrl.pathname.startsWith('/login')) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
  return res
}
```

---

## Settings Page

### Suppliers Management
- Table of all suppliers with: Name | Contact | Email | Categories | Actions
- Add/Edit/Delete suppliers
- "Default for category" toggle
- Per-project overrides available when inside a project

### Materials Configuration
- Table of all materials: Name | Category | Default Unit | Wastage % | Sort Order
- Edit wastage percentages inline
- Add custom materials
- Reorder via drag-and-drop

### User Management
- List of users (read-only, managed via Supabase)
- Show user emails and last login

---

## File & Folder Structure

```
behe-constructions/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx              ← sidebar + auth guard
│   │   ├── dashboard/page.tsx
│   │   ├── projects/
│   │   │   ├── new/page.tsx
│   │   │   └── [id]/
│   │   │       ├── page.tsx        ← overview tab
│   │   │       ├── plans/page.tsx
│   │   │       ├── quantities/page.tsx
│   │   │       ├── emails/page.tsx
│   │   │       ├── scope/page.tsx
│   │   │       └── export/page.tsx
│   │   └── settings/
│   │       ├── suppliers/page.tsx
│   │       ├── materials/page.tsx
│   │       └── users/page.tsx
├── api/
│   ├── projects/
│   │   ├── route.ts                ← GET all, POST new
│   │   └── [id]/
│   │       ├── route.ts            ← GET, PATCH, DELETE
│   │       ├── upload/route.ts
│   │       ├── analyse/route.ts
│   │       ├── quantities/route.ts
│   │       ├── emails/route.ts
│   │       ├── export-excel/route.ts
│   │       └── export-scope/route.ts
│   ├── suppliers/route.ts
│   └── materials/route.ts
├── components/
│   ├── layout/
│   │   ├── Sidebar.tsx
│   │   ├── TopBar.tsx
│   │   └── BeheLogoLink.tsx
│   ├── projects/
│   │   ├── ProjectCard.tsx
│   │   ├── ProjectHero.tsx
│   │   ├── NewProjectModal.tsx
│   │   └── VersionTimeline.tsx
│   ├── quantities/
│   │   ├── QuantityTable.tsx
│   │   ├── QuantityEditRow.tsx
│   │   ├── VersionDiffTable.tsx
│   │   └── ConfidenceBadge.tsx
│   ├── emails/
│   │   └── EmailDraftCard.tsx
│   └── ui/
│       ├── BeheButton.tsx          ← Primary CTA #393536 / #e5b96b
│       ├── StatusBadge.tsx
│       ├── FileUploadZone.tsx
│       └── LoadingSpinner.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── types.ts                ← Generated from Supabase CLI
│   ├── claude/
│   │   ├── analyse-plans.ts        ← PDF → images → Claude vision
│   │   └── generate-scope.ts       ← Scope of works generation
│   ├── excel/
│   │   ├── export-materials.ts     ← exceljs export
│   │   └── export-scope.ts
│   └── pdf/
│       └── convert.ts              ← pdf2pic, LibreOffice conversion
├── middleware.ts
├── .env.local
└── package.json
```

---

## Environment Variables (`.env.local`)

```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
ANTHROPIC_API_KEY=your_anthropic_key
CLOUDCONVERT_API_KEY=your_cloudconvert_key   # optional, for DWG conversion
```

---

## Implementation Order

Build in this order to ensure each phase is testable:

### Phase 1 — Foundation
1. Init Next.js 14 project with Tailwind CSS
2. Set up Supabase project, run all SQL migrations
3. Implement Supabase Auth (login page, middleware, session handling)
4. Build sidebar layout + Behe branding (logo, colours, fonts)
5. Dashboard page with placeholder project cards

### Phase 2 — Projects & File Upload
6. New project modal + project CRUD
7. Project detail page with tab navigation
8. File upload zone (PDF, DWG, AI) with drag-and-drop
9. DWG/AI → PDF conversion pipeline
10. PDF first-page extraction for cover photo
11. Supabase Storage integration for all files
12. Plan versions list with version numbering

### Phase 3 — AI Analysis
13. PDF → images conversion (pdf2pic, all pages)
14. Claude API integration with structured quantity extraction prompt
15. Parse and store extraction results in `quantity_extractions`
16. Quantities table UI with confidence badges
17. User edit/correction flow for quantities
18. Version diff table

### Phase 4 — Documents & Export
19. Supplier email draft generation (per category, per supplier)
20. Email draft editor UI with copy-to-clipboard
21. Excel export (exceljs, Behe template style)
22. Scope of works generation (Claude + template)
23. Scope document Excel export

### Phase 5 — Settings & Polish
24. Suppliers management page (CRUD)
25. Materials configuration page (wastage %, units)
26. Responsive mobile layout
27. Loading states, error handling, toast notifications
28. Empty states for new projects
29. Final QA pass

---

## Key Implementation Notes

### Claude API (Vision)
- Use `claude-sonnet-4-6` model for cost efficiency on analysis
- Send images as base64 with `image/jpeg` media type
- Process up to 20 pages per plan set (most fitout drawings are 5–15 pages)
- Include a clear JSON schema in the prompt — use `<json_schema>` tags for reliability
- Handle parsing errors gracefully — fall back to asking user to input quantities manually
- Set `max_tokens: 4096` for quantity extraction

### PDF Conversion
```bash
npm install pdf2pic sharp
# pdf2pic requires graphicsmagick or ghostscript installed
# On Vercel: use a Lambda layer or process files in API route with sharp only
# Alternative: use pdf.js-extract for text extraction + Claude for interpretation
```

### DWG Conversion
- Use CloudConvert API (reliable, handles DWG natively)
- Endpoint: `POST https://api.cloudconvert.com/v2/convert`
- Input format: `dwg`, Output format: `pdf`
- Store converted PDF to Supabase Storage

### Excel Export (exceljs)
```bash
npm install exceljs
```
- Always use Excel formulas (not hardcoded values) for calculations
- Style category headers: fill `#393536`, font `#e5b96b`, bold
- Use `autoFilter` on the materials table header row
- Set print area and page setup for A4 landscape

### Supabase Storage Buckets
Create these buckets in Supabase:
- `plan-files` (private) — original uploaded files
- `converted-pdfs` (private) — converted PDFs
- `cover-images` (public) — project cover photos
- `exports` (private) — generated Excel/scope files

### Row Level Security (RLS)
Enable RLS on all tables. All authenticated users can read/write all records (internal team only, no need for per-user isolation).

```sql
-- Example policy for projects table
CREATE POLICY "Authenticated users can do everything"
ON projects FOR ALL
TO authenticated
USING (true)
WITH CHECK (true);
-- Apply same pattern to all tables
```

---

## Testing Checklist

- [ ] Login/logout works correctly
- [ ] New project creates with correct data
- [ ] PDF upload stores to Supabase Storage
- [ ] DWG/AI files convert to PDF before storage
- [ ] First page of PDF extracts as cover image
- [ ] Claude analyses plans and returns valid JSON
- [ ] Quantities are stored with correct wastage applied
- [ ] User can edit quantities and see final qty update
- [ ] Uploading v2 plans creates version 2 and shows diff vs v1
- [ ] Supplier emails generate with correct project details and tone
- [ ] Excel export downloads with correct data and Behe styling
- [ ] Scope of works generates relevant sections only
- [ ] All pages are responsive on mobile
- [ ] All routes redirect to login when unauthenticated

---

## Notes for Claude Code

- Use TypeScript throughout — no `any` types where avoidable
- Use `shadcn/ui` for base UI components (Dialog, Table, Tabs, Toast, etc.) and override with Behe brand colours
- Prefer server components for data fetching, client components only where interactivity is needed
- Use `react-hook-form` + `zod` for all forms
- Use `@tanstack/react-query` for client-side data fetching and cache invalidation
- Handle all loading and error states explicitly — never leave users staring at blank screens
- All file upload components should show progress bars
- Use `sonner` for toast notifications
- Add proper TypeScript types generated from Supabase schema (`supabase gen types typescript`)
