# HealthyBite AI — Delivery Platform

A food delivery platform where every restaurant lists ingredients per dish, every
customer has a health profile, and an AI check tells the customer whether a
dish is a good idea for them before they order. Restaurants are onboarded by a
super admin, set their own delivery radius, and manage their own menu.

- **Frontend:** React (Vite) + Tailwind, styled from `DESIGN.md`'s tokens.
- **Backend:** FastAPI + SQLite (SQLAlchemy).
- **AI:** OpenAI Chat Completions, used for two things:
  - The customer-facing health check, with a deterministic rule-based fallback
    so it works before you add a real key.
  - A vendor-facing "AI Auto-fill" button on the menu editor that guesses a
    known dish's ingredients + nutrition facts from just its name (this one
    has no rule-based fallback — it needs a real key).
- **Maps:** 100% free — OpenStreetMap tiles + Leaflet for the map UI, and
  OpenStreetMap's Nominatim service (proxied through the backend) for address
  search/geocoding. No API key, no billing account, no signup required.

## Roles

| Role | How they get an account | What they do |
|---|---|---|
| Super Admin | Seeded from `.env` | Onboards restaurants (creates the vendor login for them) |
| Vendor (Restaurant) | Created by super admin | Sets address + delivery radius, manages menu + ingredients, updates order status |
| Customer | Self-registers with a health questionnaire | Searches restaurants in range, gets an AI verdict on dishes, orders |
| Delivery Partner | Not implemented yet | Stub dashboard only (see "What's stubbed" below) |

## Prerequisites

- Python 3.11+ on your PATH
- Node.js 18+ and npm
- (Optional, can add later) an OpenAI API key — needed for the AI health
  check to use real AI instead of the rule-based fallback, and required
  (no fallback) for the menu editor's "AI Auto-fill" button
- Nothing else to sign up for — maps/geocoding are free (OpenStreetMap)

## 1. Backend setup

```powershell
cd backend
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
python seed.py
uvicorn app.main:app --reload --port 8000
```

`seed.py` creates the SQLite database (`app.db`) and a super admin account from
`SUPERADMIN_EMAIL` / `SUPERADMIN_PASSWORD` in `.env` (defaults:
`admin@healthybite.ai` / `ChangeMe123!`). It also seeds a small ingredient
library so the vendor menu editor has common ingredients to suggest.

Leave `backend/.env`'s `OPENAI_API_KEY` empty for now — everything still works:
- No `OPENAI_API_KEY` → `/ai/menu-items/{id}/analyze` (the customer health
  check) uses a deterministic rule-based check (allergen match, sodium/sugar
  vs. daily limits) instead of calling OpenAI.
- No `OPENAI_API_KEY` → the vendor menu editor's **AI Auto-fill** button
  returns a clear "not configured" message instead of guessing
  ingredients/nutrition (there's no sensible rule-based substitute for that
  one — it needs real food knowledge).
- Maps/address search need no key at all, ever — they run on free
  OpenStreetMap + Nominatim.

Once you add a real OpenAI key, paste it into `backend/.env` and restart
uvicorn — no code changes needed.

Visit `http://localhost:8000/docs` for interactive Swagger docs of every
endpoint.

## 2. Frontend setup

In a second terminal:

```powershell
cd frontend
npm install
copy .env.example .env
npm run dev
```

Open `http://localhost:5173`.

Every address input (vendor onboarding, vendor settings, customer delivery
address) is the same free OpenStreetMap + Leaflet picker: type to search
(autocomplete suggestions come from the backend's `/maps/search` proxy to
Nominatim), or drag the pin / click the map to set the exact spot. Nothing to
configure — no key, no `.env` value needed for this at all.

## 3. Walk the golden path

1. **Super admin** — go to `/login`, sign in with the seeded super admin
   credentials, then **Onboard Restaurant**. Enter a restaurant name, search
   for its address (or drag the pin on the map), and a delivery radius — the
   map draws a live circle showing that coverage area. You'll get a generated
   vendor email + temp password — copy it down.
2. **Vendor** — log out, log back in with the vendor credentials from step 1.
   Go to **Menu Editor**, type a dish name, and try **AI Auto-fill** (needs
   `OPENAI_API_KEY` set — otherwise it shows a clear message and you fill the
   nutrition facts and comma-separated ingredient list in by hand; new
   ingredient names are created automatically either way). You can also paste
   a photo URL. Go to **Settings** to confirm/adjust the address and delivery
   radius (same live-circle map).
3. **Customer** — log out, click **Create an account**, fill in account info
   then the health questionnaire (allergies, hypertension/diabetes/etc.,
   sodium/sugar limits). You'll land on **Discover** — enter a delivery
   address close to the restaurant's (within its radius) to see it appear.
   Open it, click **Analyze Health** on a dish to see the AI verdict, add it
   to your cart, and place an order (Cash on Delivery or mock online payment).
4. **Vendor** — log back in as the vendor to see the order in **Orders** and
   move it through pending → confirmed → preparing → out for delivery →
   delivered.
5. **Customer** — log back in to see the updated status under **Orders**.

## What's stubbed / simplified for this pass

- **Delivery partner** role has a placeholder dashboard only — no real order
  assignment, routing, or live tracking yet.
- **Payments** are Cash on Delivery or a mock "always succeeds" online
  payment — no real payment gateway is integrated.
- Pages not present in the original design set (login, registration, restaurant
  onboarding, menu editor, vendor settings, cart/checkout) were newly designed
  using the same color/typography tokens rather than pixel-matched bento
  layouts.

## Project layout

```
backend/
  app/
    main.py            FastAPI app + router registration
    config.py           Settings loaded from .env
    database.py          SQLAlchemy engine/session
    models/               users, health_profiles, restaurants, ingredients,
                           menu_items, orders, ai_food_analyses
    routers/              auth, admin, restaurants, discovery, health, ai,
                           orders, maps
    services/              security (JWT/bcrypt), distance (Haversine),
                           openai_service (health check + menu autofill),
                           geocoding_service (OSM Nominatim proxy)
  seed.py                 Creates DB + super admin + sample ingredients
frontend/
  src/
    api/client.js          Axios instance + auth header handling
    context/                 AuthContext, CartContext
    components/               DashboardShell, CustomerNav, AddressAutocomplete,
                               AIVerdictModal, ProtectedRoute
    pages/
      auth/                   Login, Register (account + health questionnaire)
      admin/                   AdminDashboard, OnboardRestaurant
      vendor/                  VendorDashboard, VendorMenuEditor, VendorSettings
      customer/                Discover, RestaurantMenu, HealthProfile, Cart, Orders
      delivery/                DeliveryStub
```

## Notes on the radius logic

Each restaurant has its own `delivery_radius_km` (settable at onboarding, and
editable later in Vendor → Settings). Customer search
(`GET /discovery/restaurants?lat=&lng=`) computes the straight-line
(Haversine) distance from the customer's delivery address to every
approved/open restaurant, and only returns the ones where that distance is
within *that restaurant's own* radius — there's no global fixed radius. This
distance math is pure lat/lng arithmetic and is entirely independent of which
map provider is used.

## A note on OpenStreetMap at scale

Nominatim's public instance (used for the free address search/geocoding) asks
that you keep usage light and identify your app — this project already
proxies every request through the backend with a descriptive `User-Agent`
and debounces search-as-you-type on the frontend. That's fine for
development and small deployments. If you outgrow it, either self-host
Nominatim or swap `backend/app/services/geocoding_service.py` for a
commercial geocoder — the rest of the app (Leaflet map, radius math, all the
UI) doesn't need to change, since it only talks to your own `/maps/search`
and `/maps/reverse` endpoints.
