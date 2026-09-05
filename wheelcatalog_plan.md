# Master Wheel Catalog Architecture Plan

## 1. Project Goal & Purpose
* **Primary Objective:** Build a clean, responsive, filterable wheel catalog for the Castellon Tires static website so customers can browse wheel styles, inspect detailed technical specifications, view multi-angle photos, and request quotes or fitment confirmation.
* **Pricing & Data Policy:**
  * **Public Web Assets (Customer Facing):** **Zero prices.** Wholesale cost, retail MSRP, MAP, and profit margins are strictly purged from public JSON files, frontend code, and network payloads.
  * **Internal Master Dataset (Shop Facing):** Internal master files (`data/master_catalog.json`) retain wholesale dealer cost, MAP, and retail MSRP for shop reference. This file is git-ignored and never deployed.
  * **Public Identifiers (SKUs):** Manufacturer part numbers / SKUs (e.g. `AFF11885511435MB`, `D53817908345`) are public. In the wheel industry, SKUs identify specific diameter, width, bolt pattern, offset, and finish combinations and are required when a customer calls or messages the shop.

---

## 2. Deployment Architecture & Migration Path

### Current State vs. Target State
* **Current State:** The repository serves root `index.html` and root `assets/` directly from the `main` branch. There are currently no `docs/`, `scripts/`, or `data/` directories.
* **Target State:** Migrate the public website into `/docs` (GitHub Pages supported source directory). 
* **Why Migrate to `/docs`:**
  Under GitHub Pages with `.nojekyll` enabled, serving from the root directory exposes every unignored folder in the repo to public HTTP requests. Migrating public web files into `/docs` creates a physical directory boundary: `scripts/`, internal plans, and local tooling live at the repo root and cannot be requested over the web.

### Explicit Migration Procedure
Execute the following commands during project setup:

```bash
# 1. Create docs directory and move public web assets
mkdir -p docs
git mv index.html docs/
git mv assets docs/
git mv CNAME docs/
git mv .nojekyll docs/

# 2. Commit the structural migration
git commit -m "chore(infra): migrate static site root to docs/ for GitHub Pages boundary"
git push origin main
```

**GitHub Pages Configuration Step:**
1. Navigate to repository **Settings** > **Pages**.
2. Under **Build and deployment** > **Source**, verify **Deploy from a branch**.
3. Under **Branch**, select `main` and set the folder dropdown to **`/docs`**.
4. Click **Save**.

---

## 3. Industry Wheel Data Taxonomy

Automotive wheels require precise technical parameters. The catalog normalizes and indexes the following attributes:

| Attribute | Technical Term | Example | Purpose & Validation |
| :--- | :--- | :--- | :--- |
| **Diameter** | Rim Diameter | `17"`, `18"`, `19"`, `20"`, `22"` | Primary size filter. Numeric integer (e.g., `18`). |
| **Width** | Rim Width | `8.5"`, `9.5"`, `10.5"` | Dictates tire section width. Numeric float (e.g., `8.5`). |
| **Bolt Pattern** | P.C.D. (Pitch Circle Diameter) | `5x114.3`, `5x112`, `6x139.7` | Lug count $\times$ diameter in mm. Standardized format. |
| **Offset** | ET (Einpresstiefe) | `+35`, `0`, `-12` | Distance in mm from centerline. Note: `0` is a valid offset, not missing data. Missing values stored as `null`. |
| **Center Bore** | Hub Bore (CB) | `73.1`, `66.6` | In millimeters. Identifies if hubcentric rings are required. |
| **Backspacing** | Backspace | `4.5"`, `5.75"` | In inches. Critical for lifted trucks and suspension clearance. |
| **Finish** | Surface Finish | `Matte Bronze`, `Gloss Black` | Distinct visual style and SKU variant. |
| **Construction** | Manufacturing Method | `Cast Monoblock`, `Flow Formed` | Indicates weight and structural strength tier. |
| **Load Rating** | Max Wheel Load | `1800 lbs`, `2500 lbs` | Essential for truck/SUV safety verification. |
| **SKU / Part #** | Manufacturer Part Number | `AFF11885511435MB` | Public identifier used for orders and fitment verification. |

---

## 4. Internal Protection & Build-Time Pricing Leak Assertion

### 4.1 Git Exclusions
The repository `.gitignore` explicitly blocks all internal datasets, scraper caches, and dealer exports:

```gitignore
# Internal datasets & supplier pricing data (strictly ignored)
data/
scripts/raw/
*.raw.json
master_catalog.json
.venv/
__pycache__/
```

### 4.2 Automated Leak Prevention Assertion (`scripts/merge_catalogs.py`)
To prevent accidental pricing disclosure through code changes, `merge_catalogs.py` enforces a mandatory programmatic assertion before writing the public asset:

```python
FORBIDDEN_PRICE_KEYS = {
    "price", "cost", "dealer", "dealer_cost", "wholesale", 
    "msrp", "map", "retail", "retail_price", "compare_at", 
    "compare_at_price", "margin", "pricing"
}

def scan_for_pricing_leak(obj, path=""):
    """Recursively checks that no pricing keys exist in the data structure."""
    if isinstance(obj, dict):
        for key, value in obj.items():
            current_path = f"{path}.{key}" if path else key
            if key.lower() in FORBIDDEN_PRICE_KEYS:
                raise ValueError(
                    f"CRITICAL SECURITY VIOLATION: Forbidden pricing key '{key}' "
                    f"found at path '{current_path}'. Public export aborted."
                )
            scan_for_pricing_leak(value, current_path)
    elif isinstance(obj, list):
        for index, item in enumerate(obj):
            scan_for_pricing_leak(item, f"{path}[{index}]")
```
If any forbidden key is detected, the build terminates with a non-zero exit code (`sys.exit(1)`) and deletes any partial export file.

---

## 5. Image Strategy & Licensing

To maintain high performance and avoid multi-gigabyte Git repository bloat, images follow a tiered storage model:

1. **Local Thumbnails (`docs/assets/images/wheels/<brand>/<model>.webp`):**
   * Downloaded and compressed during catalog ingestion.
   * Converted to WebP format (max dimension 600px, 75% quality, ~25–35 KB).
   * Stored directly in the repo under `docs/`.
   * **Guarantee:** The primary catalog grid renders instantly with zero external network requests and zero risk of broken third-party CDN links.
2. **Full-Resolution Gallery Photos (Hotlinked Supplier CDN with Fallback):**
   * Multi-angle gallery images are kept as direct CDN URLs inside `wheels.json`.
   * **Licensing & Terms:** Castellon Tires is an authorized dealer; marketing media provided via dealer portals and storefronts is licensed for dealer sales representation.
   * **Graceful Degradation:** If an external gallery image fails to load (due to ad blockers, 404s, or referrer restrictions), the modal JavaScript detects `onerror` and falls back to the local WebP thumbnail with an informative badge ("High-res preview unavailable").
   * **Privacy Note:** Browsing modal images fetches from supplier CDNs (Shopify or Wheel Pros). If customer IP privacy from suppliers is ever required, an opt-in batch script can sync gallery images to a private Cloudflare R2 bucket.

---

## 6. Public Catalog Schema (`docs/assets/data/wheels.json`)

The public catalog output groups variants under parent wheel model cards:

```json
[
  {
    "id": "aodhan-aff1",
    "brand": "AodHan",
    "model": "AFF1",
    "series": "AFF Series",
    "construction": "Flow Formed (Single Phase Forged)",
    "description": "The AFF01 born from a lightweight SPF platform, resulting in an unparalleled blend of style and performance.",
    "thumbnail": "assets/images/wheels/aodhan/aff1.webp",
    "finishes": ["Matte Black", "Matte Bronze", "Silver Machined Face"],
    "available_diameters": [19, 20],
    "available_widths": [8.5, 9.0, 9.5, 10.5],
    "available_offsets": [30, 32, 35, 45],
    "available_bolt_patterns": ["5x114.3", "5x120"],
    "gallery_images": [
      "https://cdn.shopify.com/s/files/1/0088/6643/1087/products/AFF1_Bronze_1.png",
      "https://cdn.shopify.com/s/files/1/0088/6643/1087/products/AFF1_Bronze_Angle.png"
    ],
    "variants": [
      {
        "sku": "AFF1209511432MB",
        "diameter": 20,
        "width": 9.0,
        "bolt_pattern": "5x114.3",
        "offset": 32,
        "center_bore": 73.1,
        "finish": "Matte Black"
      },
      {
        "sku": "AFF120105511445MB",
        "diameter": 20,
        "width": 10.5,
        "bolt_pattern": "5x114.3",
        "offset": 45,
        "center_bore": 73.1,
        "finish": "Matte Black"
      }
    ]
  }
]
```

---

## 7. Frontend User Experience & Fitment Verification Workflow

### 7.1 Filter Bar Capabilities
The catalog UI (`docs/assets/js/wheel-catalog.js`) provides comprehensive multi-facet filtering:
* **Brand:** Checkbox / pills (e.g. AodHan, Fuel Off-Road, Rotiform, KMC).
* **Diameter:** Multi-select dropdown (e.g. 17", 18", 19", 20", 22").
* **Width:** Multi-select dropdown (e.g. 8.0", 8.5", 9.0", 10.0", 12.0").
* **Bolt Pattern (PCD):** Standard dropdown (e.g. 5x114.3, 5x112, 5x120, 6x139.7).
* **Offset Range:** Slider or discrete buckets (Aggressive Negative < ET0, Mid ET15–ET30, High ET35+).
* **Finish:** Color swatches / multi-select (Black, Bronze, Silver, Milled, Chrome).

### 7.2 Fitment Verification & Lead Generation Action
Because wheel fitment depends on brake caliper dimensions, suspension travel, and tire aspect ratios, the catalog avoids automated guarantees. Instead, every card features a **"Check Fitment & Request Quote"** button.

Clicking opens a dedicated inquiry modal containing:
1. **Pre-filled Wheel Context:** Selected Brand, Model, SKU, Diameter, Width, Offset, and Finish.
2. **Vehicle Input Fields:**
   * Year, Make, Model, Trim / Submodel (e.g. 2022 Honda Civic Type R).
   * Suspension Setup: [ ] Stock  [ ] Lowered (Springs/Coilovers)  [ ] Lifted.
   * Customer Phone Number or Email.
3. **Fitment Disclaimer:**
   > *"Wheel clearance varies based on vehicle trim, brake caliper package, and tire sizing. Castellon Tires technicians manually inspect vehicle clearance and load ratings before confirming any order."*
4. **Action Routing:**
   * One-touch Call Button: `tel:+15103950626` strictly for quick dialing (protocols for `tel:` cannot transfer text context).
   * Pre-filled SMS link: `sms:+15103950626?&body=Hi%20Castellon%20Tires%2C%20I%20would%20like%20to%20check%20fitment%20for%20[Brand]%20[Model]%20(SKU%3A%20[SKU])%20on%20my%20[Year]%20[Make]%20[Model].` (handles context transfer on mobile devices).
   * Static Form Submission: Embedded form via Web3Forms or Formspree transmitting the full structured fitment request directly to the shop email.

---

## 8. Maintenance Commands (Executed via `uv`)

```bash
# 1. Harvest latest AodHan catalog directly from online storefront
uv run scripts/ingest_aodhan.py

# 2. Ingest Wheel Pros dealer export (placed in scripts/raw/wheelpros.csv)
uv run scripts/ingest_wheelpros.py

# 3. Merge, assert zero price leaks, generate master_catalog.json & docs/assets/data/wheels.json
uv run scripts/merge_catalogs.py

# 4. Review git diff to verify zero price keys are committed
git diff docs/assets/data/wheels.json

# 5. Commit and publish
git add docs/assets/data/wheels.json docs/assets/images/wheels/
git commit -m "feat(catalog): update wheel catalog dataset"
git push origin main
```
