# ScrapX — AI-Powered Waste Material Marketplace

A hackathon project that lets sellers list industrial waste/scrap materials and buyers discover and request them. Upload an image of your waste material and AI automatically identifies the type, category, hazard level, and generates a description — no manual entry needed.

---

## What it does

- **AI waste detection** — upload a photo, YOLO models identify the material type (metal, plastic, electronics, wood, PPE, chemical barrels)
- **Auto-filled listings** — title, category, hazard level, and description are all filled automatically after detection
- **Marketplace** — buyers browse listings, filter by category, and send pickup/purchase requests
- **User profiles** — sellers manage their listings, buyers track their requests
- **Offline fallback** — if the Python backend is offline, detection falls back to an on-device COCO-SSD model running in the browser

---

## Tech stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React 18 + TypeScript | UI framework |
| Build tool | Vite | Dev server and bundler |
| Styling | Tailwind CSS + shadcn/ui | Components and utility CSS |
| Routing | React Router v6 | Page navigation |
| Database | Supabase (PostgreSQL) | Listings, profiles, requests |
| Auth | Supabase Auth | User signup/login/sessions |
| Storage | Supabase Storage | Listing images |
| Backend | FastAPI + Python | YOLO detection API |
| AI (server) | Ultralytics YOLO | Custom waste detection models |
| AI (browser) | TensorFlow.js + COCO-SSD | Offline detection fallback |

---

## Project structure

```
HackatonProject/
├── src/
│   ├── pages/
│   │   ├── Index.tsx          # Landing page
│   │   ├── Listings.tsx       # Browse all listings
│   │   ├── ListingDetail.tsx  # Single listing + request form
│   │   ├── ListingForm.tsx    # Create / edit a listing (AI detection here)
│   │   ├── MyListings.tsx     # Seller's own listings
│   │   ├── MyRequests.tsx     # Buyer's sent requests
│   │   ├── Profile.tsx        # User profile
│   │   └── Auth.tsx           # Login / signup
│   ├── components/
│   │   ├── Navbar.tsx
│   │   ├── ListingCard.tsx
│   │   ├── CategoryBadge.tsx
│   │   ├── HazardBadge.tsx
│   │   └── ui/                # shadcn/ui components
│   ├── contexts/
│   │   └── AuthContext.tsx    # Global auth state
│   ├── lib/
│   │   ├── utils.ts
│   │   └── wasteImageDetection.ts  # On-device detection + description builder
│   └── integrations/
│       └── supabase/
│           ├── client.ts      # Supabase JS client
│           └── types.ts       # Auto-generated DB types
│
├── YoloCv/
│   ├── main.py                # FastAPI backend — all YOLO endpoints
│   ├── requirements.txt       # Python dependencies
│   ├── bestWood.pt            # Custom YOLO model — wood
│   ├── bestPlastic.pt         # Custom YOLO model — plastic
│   ├── bestElectronics.pt     # Custom YOLO model — electronics
│   ├── bestProtection.pt      # Custom YOLO model — PPE
│   ├── best1.pt               # Custom YOLO model — barrels
│   ├── best2.pt               # Custom YOLO model — metal scrap
│   └── YOLO_Models/           # Additional base YOLO models
│
├── supabase/
│   └── migrations/            # Database schema SQL
│
├── .env                       # Environment variables (see below)
└── package.json
```

---

## Getting started

### Prerequisites

- Node.js 18+
- Python 3.10+
- A Supabase project (free tier works)

---

### 1. Clone the repo

```bash
git clone <your-repo-url>
cd HackatonProject
```

---

### 2. Set up environment variables

Create a `.env` file in the root with:

```env
VITE_SUPABASE_URL=https://your-project-id.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=your-supabase-anon-key
VITE_DETECTION_API_URL=http://localhost:8000
```

- `VITE_SUPABASE_URL` and `VITE_SUPABASE_PUBLISHABLE_KEY` come from your Supabase project settings under **API**
- `VITE_DETECTION_API_URL` points to your running Python backend — remove this line to use browser-only detection

---

### 3. Install frontend dependencies

```bash
npm install
```

---

### 4. Set up the database

Go to your Supabase project → **SQL Editor** and run the migration file:

```
supabase/migrations/20260405131531_*.sql
```

This creates the `waste_listings`, `listing_requests`, and `profiles` tables with the correct schema and row-level security policies.

---

### 5. Run the frontend

```bash
npm run dev
```

Frontend will be available at `http://localhost:5173`

---

### 6. Run the Python backend (optional but recommended)

```bash
cd YoloCv
pip install -r requirements.txt
python main.py
```

Backend will be available at `http://localhost:8000`

You can verify it's working at: `http://localhost:8000/health`

> If you skip this step, the app still works — it falls back to on-device detection using TensorFlow.js and COCO-SSD in the browser.

---

## How AI detection works

1. User uploads an image on the Create Listing page
2. Image is sent to `POST /detect` on the FastAPI backend
3. All 6 custom YOLO models run on the image simultaneously
4. The model with the highest confidence + most detections wins
5. The winning model maps to a waste category, title, and hazard level
6. `buildSuggestedDescription()` generates a description locally based on category and hazard
7. The form auto-fills all four fields — title, category, hazard level, description
8. If the backend is unreachable, step 2-6 are replaced by TensorFlow.js COCO-SSD running in the browser

---

## API endpoints

The FastAPI backend runs at `http://localhost:8000`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Health check message |
| GET | `/health` | Returns status and number of loaded models |
| POST | `/detect` | Detect waste from a multipart image upload |
| POST | `/detect-base64` | Detect waste from a base64 encoded image string |

### Example `/detect` response

```json
{
  "success": true,
  "best_match": "MetalScrap",
  "category": "metal",
  "suggested_title": "Metal Scrap Material",
  "hazard_level": "low",
  "confidence": 0.87,
  "all_detections": {
    "wood": { "detections": 0, "max_confidence": 0.0 },
    "plastic": { "detections": 0, "max_confidence": 0.12 },
    "MetalScrap": { "detections": 3, "max_confidence": 0.87 }
  },
  "multi_labels": ["MetalScrap"]
}
```

---

## Database schema

### `waste_listings`
| Column | Type | Description |
|---|---|---|
| id | uuid | Primary key |
| user_id | uuid | FK to auth.users |
| title | text | Listing title |
| description | text | Material description |
| category | text | metal / plastic / chemical / organic / electronic / textile / glass / other |
| quantity | numeric | Amount available |
| unit | text | kg / tons / liters etc. |
| hazard_level | text | none / low / medium / high |
| price | numeric | Price (0 = free) |
| currency | text | USD etc. |
| location | text | City or region |
| image_url | text | Public URL from Supabase Storage |
| status | text | active / inactive |
| created_at | timestamptz | Auto-set |

### `listing_requests`
| Column | Type | Description |
|---|---|---|
| id | uuid | Primary key |
| listing_id | uuid | FK to waste_listings |
| requester_id | uuid | FK to auth.users |
| message | text | Optional message from buyer |
| status | text | pending / accepted / rejected |

### `profiles`
| Column | Type | Description |
|---|---|---|
| user_id | uuid | FK to auth.users |
| display_name | text | Public name |
| company_name | text | Optional company |
| location | text | User location |
| phone | text | Contact number |
| bio | text | About the user |

---

## Waste categories

| Category | YOLO model | Hazard level |
|---|---|---|
| Metal | best2.pt | Low |
| Plastic | bestPlastic.pt | Low |
| Organic / Wood | bestWood.pt | Low |
| Electronic | bestElectronics.pt | Medium |
| Chemical / Barrel | best1.pt | High |
| PPE / Other | bestProtection.pt | None |

---

## Running tests

```bash
# Unit tests
npm test

# End-to-end tests (requires frontend running)
npx playwright test
```

---

## Known limitations

- The Python backend must be running locally — it is not deployed to any cloud server yet
- YOLO models are large files (50–100MB each) and take a few seconds to load on first start
- On-device COCO-SSD fallback uses generic object classes and is less accurate than the custom YOLO models
- No real-time notifications — sellers must manually check for new requests

---

## Possible next features

- WebSocket notifications so sellers get instant alerts when a buyer sends a request
- Price suggestion based on category and quantity
- Duplicate listing detection
- Deploy FastAPI backend to a cloud server (Railway, Render, or AWS)
- Admin dashboard with category and volume analytics