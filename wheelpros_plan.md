# Wheel Pros Catalog Ingestion & Asset Plan

## 1. Overview & Brand Scope
Wheel Pros distributes dozens of major automotive wheel brands. Castellon Tires focuses on core truck, off-road, tuner, and classic lines:
* **Truck & Off-Road:** Fuel Off-Road, KMC Wheels, XD Series, American Force.
* **Tuner & Sport:** Rotiform, Motegi Racing.
* **Classic & Muscle:** American Racing, US Mags.
* **Luxury:** Asanti.

**Objective:** Ingest the shop's selected brands from the official Wheel Pros dealer export, structure parent models and fitment variants, retain wholesale and MAP pricing for internal shop records, and export sanitized assets for the public website.

---

## 2. Ingestion Strategy: Portal Export vs. Web Scraping

### Primary Strategy: Dealer Data Portal CSV/SFTP (Recommended)
Because Castellon Tires is an authorized dealer:
* **Portal:** [data.wheelpros.com](https://data.wheelpros.com)
* **Format:** Comprehensive CSV/XLSX export.
* **Key Fields Extracted:**
  * Identifiers: `Brand_Name`, `Style_Name`, `Part_Number` (SKU).
  * Specifications: `Diameter`, `Width`, `Bolt_Pattern_1`, `Bolt_Pattern_2`, `Offset`, `Backspacing`, `Center_Bore`, `Load_Rating`.
  * Visuals: `Finish_Description`, `Image_URL`.
  * Pricing (Internal Only): `Dealer_Price`, `MAP_Price`, `MSRP_Price`.

### Why Scraping `wheelpros.com` is Discouraged
* Wheel Pros operates heavy bot-mitigation layers (Akamai / Cloudflare).
* Public pages obscure wholesale dealer cost and technical engineering fitment tables.
* Scraping 30,000+ variants across 20+ brands is fragile, slow, and prone to IP bans.

---

## 3. Data Processing & Parsing Logic

### 3.1 Strict Numeric Sanitization (Preventing `NaN` / False `0` Values)
* In standard CSV files, empty cells are read by pandas as `float('nan')`, which serializes into non-standard JSON `NaN`.
* Conversely, converting missing values to `0` corrupts valid technical data (e.g. an offset of `0mm` / ET0 is a real offset for rear-wheel-drive trucks, not missing data).
* **Rule:** Missing numeric values are converted strictly to Python `None` (which renders as valid JSON `null`).

### 3.2 Dual Bolt Pattern Support
Many wheels are dual-drilled (e.g. 5x114.3 and 5x120 on the same wheel). The parser evaluates both `Bolt_Pattern_1` and `Bolt_Pattern_2`:
```python
def normalize_bolt_patterns(p1, p2):
    patterns = []
    if p1 and str(p1).strip() not in ("", "nan", "None"):
        patterns.append(str(p1).strip())
    if p2 and str(p2).strip() not in ("", "nan", "None"):
        p2_str = str(p2).strip()
        if p2_str not in patterns:
            patterns.append(p2_str)
    return patterns
```

### 3.3 Parent Model Grouping
The script aggregates individual part numbers into parent wheel models:
```
Parent Wheel: Fuel Maverick (Brand: Fuel Off-Road)
├── Finishes: ["Matte Black with Milled Accents", "Chrome", "Gloss Black"]
├── Diameters: [17, 18, 20, 22]
├── Widths: [8.5, 9.0, 10.0, 12.0]
├── Offsets: [-44, -24, -12, 0, 14, 20]
├── Bolt Patterns: ["5x127", "5x139.7", "6x135", "6x139.7", "8x170", "8x180"]
└── Variants: [ List of individual SKUs with exact fitment specs ]
```

---

## 4. Execution Script (`scripts/ingest_wheelpros.py`)

```python
# /// script
# dependencies = ["pandas", "pillow", "requests"]
# ///

from pathlib import Path
import json
import math
import sys
import pandas as pd
from PIL import Image
import requests

RAW_CSV = Path("scripts/raw/wheelpros.csv")
OUTPUT_INTERNAL = Path("data/wheelpros_raw.json")
THUMBNAIL_DIR = Path("docs/assets/images/wheels/wheelpros")

ALLOWED_BRANDS = {
    "fuel off-road", "rotiform", "kmc wheels", "american racing",
    "motegi racing", "xd series", "asanti", "us mags"
}

def safe_float(val):
    if val is None:
        return None
    try:
        f = float(val)
        return None if math.isnan(f) else f
    except (ValueError, TypeError):
        return None

def safe_str(val):
    if val is None:
        return ""
    s = str(val).strip()
    return "" if s.lower() in ("nan", "none", "null") else s

def download_and_optimize_thumbnail(image_url, brand, model):
    if not image_url or not image_url.startswith("http"):
        return None
    
    clean_model = "".join(c if c.isalnum() else "-" for c in model.lower()).strip("-")
    clean_brand = "".join(c if c.isalnum() else "-" for c in brand.lower()).strip("-")
    out_filename = f"{clean_brand}-{clean_model}.webp"
    dest_path = THUMBNAIL_DIR / out_filename

    if dest_path.exists():
        return f"assets/images/wheels/wheelpros/{out_filename}"

    try:
        dest_path.parent.mkdir(parents=True, exist_ok=True)
        resp = requests.get(image_url, timeout=10)
        if resp.status_code == 200:
            import io
            img = Image.open(io.BytesIO(resp.content)).convert("RGB")
            img.thumbnail((600, 600))
            img.save(dest_path, "WEBP", quality=75)
            return f"assets/images/wheels/wheelpros/{out_filename}"
    except Exception as e:
        print(f"Warning: Failed to process thumbnail for {model}: {e}")
    
    return None

def process_wheelpros():
    if not RAW_CSV.exists():
        print(f"Error: Required file '{RAW_CSV}' not found. Place dealer CSV export here.")
        sys.exit(1)

    print(f"Loading {RAW_CSV}...")
    df = pd.read_csv(RAW_CSV, low_memory=False)

    # Filter to approved wheel brands
    brand_col = next((c for c in df.columns if c.lower() in ("brand_name", "brand")), None)
    if not brand_col:
        print("Error: Could not identify Brand column in CSV.")
        sys.exit(1)

    df_filtered = df[df[brand_col].astype(str).str.lower().isin(ALLOWED_BRANDS)]
    print(f"Filtered to {len(df_filtered)} matching SKUs for allowed brands.")

    grouped = {}
    for _, row in df_filtered.iterrows():
        brand = safe_str(row.get("Brand_Name", row.get("Brand", "")))
        model = safe_str(row.get("Style_Name", row.get("Style", row.get("Model", ""))))
        if not brand or not model:
            continue

        model_key = f"{brand.lower()}::{model.lower()}"
        if model_key not in grouped:
            grouped[model_key] = {
                "brand": brand,
                "model": model,
                "image_url": safe_str(row.get("Image_URL", row.get("Image", ""))),
                "finishes": set(),
                "diameters": set(),
                "widths": set(),
                "offsets": set(),
                "bolt_patterns": set(),
                "variants": []
            }

        sku = safe_str(row.get("Part_Number", row.get("SKU", "")))
        finish = safe_str(row.get("Finish_Description", row.get("Finish", "")))
        diam = safe_float(row.get("Diameter"))
        width = safe_float(row.get("Width"))
        offset = safe_float(row.get("Offset"))
        cb = safe_float(row.get("Center_Bore"))
        load = safe_float(row.get("Load_Rating"))
        
        bp1 = safe_str(row.get("Bolt_Pattern_1", row.get("BoltPattern", "")))
        bp2 = safe_str(row.get("Bolt_Pattern_2", ""))
        bp_list = [bp for bp in [bp1, bp2] if bp]
        bp_str = " / ".join(bp_list) if bp_list else ""

        if finish: grouped[model_key]["finishes"].add(finish)
        if diam is not None: grouped[model_key]["diameters"].add(int(diam) if diam.is_integer() else diam)
        if width is not None: grouped[model_key]["widths"].add(width)
        if offset is not None: grouped[model_key]["offsets"].add(int(offset) if offset.is_integer() else offset)
        for bp in bp_list: grouped[model_key]["bolt_patterns"].add(bp)

        # Internal variant retaining cost/MAP for master records
        grouped[model_key]["variants"].append({
            "sku": sku,
            "finish": finish,
            "diameter": diam,
            "width": width,
            "bolt_pattern": bp_str,
            "offset": offset,
            "center_bore": cb,
            "load_rating": load,
            "pricing": {
                "dealer_cost": safe_float(row.get("Dealer_Price", row.get("Cost", None))),
                "map": safe_float(row.get("MAP_Price", row.get("MAP", None))),
                "msrp": safe_float(row.get("MSRP_Price", row.get("MSRP", None)))
            }
        })

    # Prepare structured records
    output_records = []
    for m in grouped.values():
        thumb_path = download_and_optimize_thumbnail(m["image_url"], m["brand"], m["model"])
        output_records.append({
            "id": f"wp-{m['brand'].lower()}-{m['model'].lower()}".replace(" ", "-"),
            "brand": m["brand"],
            "model": m["model"],
            "thumbnail": thumb_path,
            "image_url": m["image_url"],
            "finishes": sorted(list(m["finishes"])),
            "available_diameters": sorted(list(m["diameters"])),
            "available_widths": sorted(list(m["widths"])),
            "available_offsets": sorted(list(m["offsets"])),
            "available_bolt_patterns": sorted(list(m["bolt_patterns"])),
            "variants": m["variants"]
        })

    OUTPUT_INTERNAL.parent.mkdir(parents=True, exist_ok=True)
    with open(OUTPUT_INTERNAL, "w") as f:
        json.dump(output_records, f, indent=2)

    print(f"Successfully processed {len(output_records)} parent wheel models into {OUTPUT_INTERNAL}")

if __name__ == "__main__":
    process_wheelpros()
```
