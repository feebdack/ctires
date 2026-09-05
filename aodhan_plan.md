# AodHan Wheels Ingestion & Asset Plan

## 1. Overview & Brand Scope
AodHan Wheels specializes in flow-formed and cast wheels targeted primarily at Japanese, European, and American sport/tuner platforms:
* **AFF Series:** Flow-formed (Single Phase Forged / SPF) lightweight designs (AFF1, AFF2, AFF7).
* **AH Series:** Classic stepped lip, multi-piece aesthetic, and retro mesh (AH02, AH06, AH08).
* **DS Series:** Deep dish and aggressive concavity (DS02, DS05, DS07).
* **LS Series:** Luxury sport profiles.

**Objective:** Ingest the complete AodHan wheel catalog directly from their online storefront API, parse technical specifications (diameters, widths, bolt patterns, offsets, hub bores, weights, concavity/lip sizes), retain retail pricing for internal shop records, generate local WebP thumbnails, and prepare sanitized records for the public catalog.

---

## 2. Scraping & Endpoint Architecture

### 2.1 Endpoint Specification & Pagination Contract
* **Target:** `https://www.aodhanwheels.com/collections/wheels/products.json?limit=250`
* **Verified Storefront Contract:**
  * Unlike authenticated Admin REST endpoints which use Link headers with cursor tokens, the public Shopify Storefront endpoint uses query parameter pagination: `?limit=250&page=1`, `?page=2`, etc.
  * Page 1 returns active wheel collections (25 parent models).
  * Page 2 returns `{"products": []}`, which signals clean termination.
* **Resilient HTTP Handling:**
  * The ingestion script enforces strict HTTP status validation.
  * If a request receives **HTTP 429 (Too Many Requests)**, the script does not abort or write partial data; it pauses and retries with exponential backoff on a **5s, 15s, 30s** schedule.
  * If an unrecoverable HTTP error occurs (e.g. persistent 500 or network drop), the script exits with an error status (`sys.exit(1)`) and **does not write a partial catalog**.

---

## 3. Variant & Specification Parsing Logic

### 3.1 Option Field Structure
On `aodhanwheels.com`, wheel products structure specifications across three discrete option values:
* **Option 1 (`Size`):** Diameter and width string (e.g. `"20x9"`, `"20x10.5"`, `"18x8.5"`).
  * Diameter: Parsed as integer (e.g. `20`).
  * Width: Parsed as float (e.g. `9.0` or `10.5`).
* **Option 2 (`PCD | ET | HB`):** Pipe-delimited engineering specifications (e.g. `"5x114.3 | +32 | 73.1"`).
  * Bolt Pattern / PCD: Extracted directly (e.g. `"5x114.3"`). Missing/null option returns `None`.
  * Offset (ET): Extracted and parsed as integer (e.g. `+32` $\rightarrow$ `32`, `-12` $\rightarrow$ `-12`).
  * Hub Bore (HB): Extracted as center bore float (e.g. `73.1`).
* **Option 3 (`Color`):** Finish description (e.g. `"Matte Bronze"`, `"Gloss Black"`, `"Silver Machined Face"`).

### 3.2 HTML Description & Specification Table Parsing
The script parses `body_html` using BeautifulSoup to extract:
* Marketing summary description from leading paragraphs.
* Bulleted feature highlights (e.g., "Single Phase Forging Construction", "Concave Spoke Profile").
* Specification table: Parses row cells (`Size`, `Offset`, `Weight (lbs)`, `Concavity` / `Lip Size`) and matches them to variants by `(size, offset)` key.

---

## 4. Execution Script (`scripts/ingest_aodhan.py`)

```python
# /// script
# dependencies = ["requests", "beautifulsoup4", "pillow"]
# ///

from pathlib import Path
import io
import json
import re
import sys
import time
from bs4 import BeautifulSoup
from PIL import Image
import requests

BASE_URL = "https://www.aodhanwheels.com/collections/wheels/products.json"
OUTPUT_INTERNAL = Path("data/aodhan_raw.json")
THUMBNAIL_DIR = Path("docs/assets/images/wheels/aodhan")
RETRY_DELAYS = [5, 15, 30]

def parse_size_option(size_str):
    """Parses '20x9' or '18x8.5' into diameter (int) and width (float)."""
    if not size_str:
        return None, None
    match = re.match(r"^(\d+)\s*[xX]\s*([\d\.]+)", str(size_str).strip())
    if match:
        diam = int(match.group(1))
        width = float(match.group(2))
        return diam, width
    return None, None

def parse_specs_option(spec_str):
    """Parses '5x114.3 | +32 | 73.1' into bolt pattern, offset, and center bore."""
    if not spec_str:
        return None, None, None
    
    parts = [p.strip() for p in str(spec_str).split("|")]
    bolt_pattern = parts[0] if len(parts) > 0 and parts[0] else None
    
    offset = None
    if len(parts) > 1:
        et_match = re.search(r"([+-]?\d+)", parts[1])
        if et_match:
            offset = int(et_match.group(1))
            
    center_bore = None
    if len(parts) > 2:
        try:
            center_bore = float(parts[2])
        except ValueError:
            center_bore = None

    return bolt_pattern, offset, center_bore

def parse_html_body(html_content):
    """Extracts description, features list, and specification table mappings."""
    if not html_content:
        return "", [], {}
    
    soup = BeautifulSoup(html_content, "html.parser")
    
    # Extract introductory paragraph
    p_tag = soup.find("p")
    description = p_tag.get_text(strip=True) if p_tag else ""
    
    # Extract feature list items
    features = [li.get_text(strip=True) for li in soup.find_all("li") if li.get_text(strip=True)]
    
    # Parse specifications table if present
    spec_table_lookup = {}
    table = soup.find("table")
    if table:
        rows = table.find_all("tr")
        if rows:
            headers = [th_td.get_text(strip=True).lower() for th_td in rows[0].find_all(["td", "th"])]
            
            size_idx = next((i for i, h in enumerate(headers) if "size" in h), None)
            offset_idx = next((i for i, h in enumerate(headers) if "offset" in h), None)
            weight_idx = next((i for i, h in enumerate(headers) if "weight" in h), None)
            concavity_idx = next((i for i, h in enumerate(headers) if "concav" in h), None)
            lip_idx = next((i for i, h in enumerate(headers) if "lip" in h), None)

            for row in rows[1:]:
                cells = [c.get_text(strip=True) for c in row.find_all(["td", "th"])]
                if len(cells) <= max(filter(lambda x: x is not None, [size_idx, offset_idx]), default=-1):
                    continue
                
                raw_size = cells[size_idx].lower().replace(" ", "") if size_idx is not None and size_idx < len(cells) else ""
                raw_offset = cells[offset_idx] if offset_idx is not None and offset_idx < len(cells) else ""
                
                offset_val = None
                et_m = re.search(r"([+-]?\d+)", raw_offset)
                if et_m:
                    offset_val = int(et_m.group(1))

                key = (raw_size, offset_val)
                weight_val = None
                if weight_idx is not None and weight_idx < len(cells):
                    w_m = re.search(r"([\d\.]+)", cells[weight_idx])
                    if w_m:
                        try: weight_val = float(w_m.group(1))
                        except ValueError: weight_val = None

                concavity_val = cells[concavity_idx] if concavity_idx is not None and concavity_idx < len(cells) else None
                lip_val = cells[lip_idx] if lip_idx is not None and lip_idx < len(cells) else None

                spec_table_lookup[key] = {
                    "weight_lbs": weight_val,
                    "concavity": concavity_val,
                    "lip_size": lip_val
                }

    return description, features, spec_table_lookup

def download_and_optimize_thumbnail(image_url, model_title):
    if not image_url or not image_url.startswith("http"):
        return None
    
    clean_model = "".join(c if c.isalnum() else "-" for c in model_title.lower()).strip("-")
    out_filename = f"{clean_model}.webp"
    dest_path = THUMBNAIL_DIR / out_filename

    if dest_path.exists():
        return f"assets/images/wheels/aodhan/{out_filename}"

    try:
        dest_path.parent.mkdir(parents=True, exist_ok=True)
        resp = requests.get(image_url, timeout=10)
        if resp.status_code == 200:
            img = Image.open(io.BytesIO(resp.content)).convert("RGB")
            img.thumbnail((600, 600))
            img.save(dest_path, "WEBP", quality=75)
            return f"assets/images/wheels/aodhan/{out_filename}"
    except Exception as e:
        print(f"Warning: Failed to create thumbnail for {model_title}: {e}")
    
    return None

def fetch_with_retry(url, max_retries=3):
    for attempt in range(1, max_retries + 1):
        try:
            resp = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}, timeout=15)
            if resp.status_code == 200:
                return resp.json()
            elif resp.status_code == 429:
                wait_time = RETRY_DELAYS[attempt - 1] if attempt <= len(RETRY_DELAYS) else 30
                print(f"Rate limited (HTTP 429). Retrying in {wait_time}s (attempt {attempt}/{max_retries})...")
                time.sleep(wait_time)
            elif resp.status_code >= 500:
                time.sleep(3)
            else:
                resp.raise_for_status()
        except requests.RequestException as e:
            if attempt == max_retries:
                raise RuntimeError(f"Failed to fetch {url} after {max_retries} attempts: {e}")
            time.sleep(2)
    raise RuntimeError(f"Exhausted retries for {url}")

def harvest_aodhan():
    wheels = []
    page = 1
    
    while True:
        url = f"{BASE_URL}?limit=250&page={page}"
        print(f"Fetching AodHan collection (page {page})...")
        data = fetch_with_retry(url)
        products = data.get("products", [])
        
        if not products:
            print(f"Reached end of collection at page {page}.")
            break

        for p in products:
            title = p.get("title", "").strip()
            handle = p.get("handle", "").strip()
            body_html = p.get("body_html", "")
            description, features, spec_lookup = parse_html_body(body_html)
            
            images = [img.get("src") for img in p.get("images", []) if img.get("src")]
            primary_image = images[0] if images else None
            thumbnail_rel_path = download_and_optimize_thumbnail(primary_image, title)

            diameters = set()
            widths = set()
            offsets = set()
            bolt_patterns = set()
            finishes = set()
            variants = []

            for v in p.get("variants", []):
                opt1 = v.get("option1")  # Size e.g. "20x9"
                opt2 = v.get("option2")  # PCD | ET | HB e.g. "5x114.3 | +32 | 73.1"
                opt3 = v.get("option3")  # Finish e.g. "Matte Black"

                diam, width = parse_size_option(opt1)
                bp, offset, cb = parse_specs_option(opt2)
                finish = str(opt3).strip() if opt3 else ""

                if diam is not None: diameters.add(diam)
                if width is not None: widths.add(width)
                if offset is not None: offsets.add(offset)
                if bp: bolt_patterns.add(bp)
                if finish: finishes.add(finish)

                # Match table specs by (size, offset)
                size_key = str(opt1).lower().replace(" ", "") if opt1 else ""
                matched_specs = spec_lookup.get((size_key, offset), {})

                # Variant entry retaining pricing for internal records
                variants.append({
                    "sku": v.get("sku"),
                    "title": v.get("title"),
                    "diameter": diam,
                    "width": width,
                    "bolt_pattern": bp,
                    "offset": offset,
                    "center_bore": cb,
                    "finish": finish,
                    "weight_lbs": matched_specs.get("weight_lbs"),
                    "concavity": matched_specs.get("concavity"),
                    "lip_size": matched_specs.get("lip_size"),
                    "available": v.get("available"),
                    "pricing": {
                        "retail_price": float(v.get("price", 0)),
                        "compare_at_price": float(v.get("compare_at_price") or 0)
                    }
                })

            wheel_record = {
                "id": f"aodhan-{handle}".lower(),
                "brand": "AodHan",
                "model": title,
                "handle": handle,
                "description": description,
                "features": features,
                "thumbnail": thumbnail_rel_path,
                "gallery_images": images,
                "finishes": sorted(list(finishes)),
                "available_diameters": sorted(list(diameters)),
                "available_widths": sorted(list(widths)),
                "available_offsets": sorted(list(offsets)),
                "available_bolt_patterns": sorted(list(bolt_patterns)),
                "variants": variants
            }
            wheels.append(wheel_record)

        page += 1
        time.sleep(1)

    OUTPUT_INTERNAL.parent.mkdir(parents=True, exist_ok=True)
    with open(OUTPUT_INTERNAL, "w") as f:
        json.dump(wheels, f, indent=2)

    print(f"Successfully harvested and processed {len(wheels)} AodHan models into {OUTPUT_INTERNAL}")

if __name__ == "__main__":
    harvest_aodhan()
```
