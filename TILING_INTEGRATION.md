# CAED AI — Tiling Pipeline Integration Guide

> **Target codebase:** `pdf_to_dxf/`  
> **New module:** `caed_tiling_pipeline.py`  
> **Goal:** Wire the 8-tile overlap pipeline into your existing `api.py` → `pdfextract` → `oda_converter.py` flow without breaking anything that already works.

---

## 1. Where the New File Lives

```
pdf_to_dxf/
├── api.py                      ← expose new /convert-tiled endpoint here
├── run.py
├── oda_converter.py
├── diagnose_raster.py
├── requirements.txt            ← add 3 lines (see §2)
├── pdfextract/
│   ├── __init__.py
│   └── ... (your existing extraction logic)
├── caed_tiling_pipeline.py     ← DROP THE NEW FILE HERE  ✅
├── frontend/
└── temp_data/                  ← tiles + debug images go here
```

Place `caed_tiling_pipeline.py` in the **project root**, next to `api.py`. No sub-package needed — it is a self-contained module.

---

## 2. Update `requirements.txt`

Add these three lines. Everything else it uses (`numpy`, `pathlib`, `logging`) is already in stdlib or pulled in transitively.

```txt
opencv-python-headless>=4.9.0
pdf2image>=1.17.0
Pillow>=10.0.0
```

Then reinstall:

```bash
# Make sure your venv is active first
source venv/bin/activate
pip install -r requirements.txt
```

> **Why `opencv-python-headless`?**  
> The regular `opencv-python` pulls in GUI libs (Qt/GTK) that crash in headless server environments. The headless variant is identical for all image-processing work — it just drops the `cv2.imshow` window functions you won't use anyway.

---

## 3. Integrate Into `api.py`

### 3a. Import the pipeline

Add these imports near the top of `api.py`, below your existing imports:

```python
# ── CAED AI Tiling Pipeline ──────────────────────────────────────────────
from caed_tiling_pipeline import (
    run_pipeline,          # full end-to-end orchestrator
    compute_tile_specs,    # if you need the tile layout separately
    slice_tiles,           # if you need just the crops
    process_tile,          # placeholder — replace with your real processor
    local_to_global,       # coordinate translation helper
    line_local_to_global,  # line-specific coordinate helper
    nms,                   # duplicate suppression
    Detection,             # dataclass for detected features
    TileSpec,              # dataclass for tile metadata
)
```

### 3b. Add the new endpoint

Paste this block into `api.py`. Adjust the Flask/FastAPI decorator to match whichever framework you are using:

```python
# ── If you use Flask ─────────────────────────────────────────────────────
@app.route("/convert-tiled", methods=["POST"])
def convert_tiled():
    """
    Accepts a PDF upload, runs the 8-tile pipeline, and returns:
      - global detections as JSON
      - path to the debug overlay image
    """
    import os, tempfile
    from flask import request, jsonify

    if "file" not in request.files:
        return jsonify({"error": "No file uploaded"}), 400

    pdf_file = request.files["file"]
    if not pdf_file.filename.endswith(".pdf"):
        return jsonify({"error": "Only PDF files are supported"}), 400

    # Save upload to temp_data/ (matches your existing pattern)
    save_dir = os.path.join(os.path.dirname(__file__), "temp_data")
    os.makedirs(save_dir, exist_ok=True)
    tmp_path = os.path.join(save_dir, pdf_file.filename)
    pdf_file.save(tmp_path)

    try:
        results = run_pipeline(
            pdf_path=tmp_path,
            output_dir=save_dir,
            dpi=600,            # keep at 600 — lower DPI loses fine dimension lines
            n_rows=4,
            n_cols=2,           # 4×2 = 8 tiles
            overlap_frac=0.12,  # 12 % overlap on each edge
            save_tiles=True,    # individual tile PNGs in temp_data/
            processor=your_real_processor,   # ← see §4
        )

        # Serialise detections for the JSON response
        detections_json = [
            {
                "tile_id": d.tile_id,
                "label":   d.label,
                "score":   round(d.score, 4),
                "bbox":    [round(d.x1), round(d.y1), round(d.x2), round(d.y2)],
            }
            for d in results["final_detections"]
        ]

        return jsonify({
            "status":       "ok",
            "n_tiles":      len(results["specs"]),
            "n_detections": len(detections_json),
            "detections":   detections_json,
            "debug_image":  os.path.join(save_dir, "debug_tile_grid.png"),
        })

    except Exception as e:
        return jsonify({"error": str(e)}), 500


# ── If you use FastAPI instead, replace the above with: ──────────────────
# from fastapi import UploadFile, File, HTTPException
# from fastapi.responses import JSONResponse
#
# @app.post("/convert-tiled")
# async def convert_tiled(file: UploadFile = File(...)):
#     if not file.filename.endswith(".pdf"):
#         raise HTTPException(400, "Only PDF files are supported")
#     contents = await file.read()
#     save_dir = Path("temp_data")
#     save_dir.mkdir(exist_ok=True)
#     tmp_path = save_dir / file.filename
#     tmp_path.write_bytes(contents)
#     results = run_pipeline(pdf_path=tmp_path, output_dir=save_dir, ...)
#     return JSONResponse({"status": "ok", ...})
```

---

## 4. Plug In Your Real Processor

This is the **only** function you need to swap out. Open `caed_tiling_pipeline.py` and replace the body of `process_tile` — or, better, pass your own function via the `processor=` argument to `run_pipeline` so you don't modify the pipeline file at all.

### Pattern A — pass a custom processor (recommended)

```python
# In api.py (or wherever you call run_pipeline)

def my_vectoriser(tile_image, tile_spec):
    """
    Your real OCR / vectorisation logic.

    Parameters
    ----------
    tile_image : np.ndarray  BGR image of one tile (already cropped)
    tile_spec  : TileSpec    metadata — tile_id, col, row, width, height

    Returns
    -------
    list[Detection]  — bounding boxes in TILE-LOCAL pixels.
                       DO NOT add offsets here; the pipeline does that.
    """
    detections = []

    # ── Example: call your existing pdfextract logic on the tile ─────────
    # from pdfextract import extract_features
    # raw = extract_features(tile_image)
    # for item in raw:
    #     detections.append(Detection(
    #         x1=item.bbox[0], y1=item.bbox[1],
    #         x2=item.bbox[2], y2=item.bbox[3],
    #         label=item.category,
    #         score=item.confidence,
    #         tile_id=tile_spec.tile_id,
    #     ))

    return detections

# Then pass it:
results = run_pipeline(pdf_path=..., processor=my_vectoriser)
```

### Pattern B — inline replacement inside `process_tile`

If you prefer to edit the pipeline file directly, find the block labelled `§ 4  PROCESSING PLACEHOLDER` and replace everything below `Current stub behaviour` with your logic. Keep the function signature identical:

```python
def process_tile(tile_image: np.ndarray, tile_spec: TileSpec) -> list[Detection]:
    # YOUR CODE HERE
    ...
    return list_of_detections   # tile-local coords only
```

---

## 5. Connect to `oda_converter.py` (DWG Export)

Once detections are in global coordinates, pass them to your ODA conversion step. Here is the bridge pattern:

```python
# After run_pipeline returns:
results = run_pipeline(pdf_path=pdf_path, output_dir=save_dir)
global_detections = results["final_detections"]   # list[Detection], global coords

# Build the entity list your ODA converter expects.
# Adjust field names to match your existing ODA wrapper.
dwg_entities = []
for det in global_detections:
    dwg_entities.append({
        "type":  "INSERT",          # or LINE, CIRCLE, TEXT — whatever your schema uses
        "layer": det.label,
        "x":     det.x1,
        "y":     det.y1,
        "width": det.x2 - det.x1,
        "height": det.y2 - det.y1,
    })

# If you have line features specifically, use the helper:
# from caed_tiling_pipeline import line_local_to_global
# gx1, gy1, gx2, gy2 = line_local_to_global(lx1, ly1, lx2, ly2, tile_spec)
# dwg_entities.append({"type": "LINE", "x1": gx1, "y1": gy1, "x2": gx2, "y2": gy2})

# Pass to your existing ODA step
from oda_converter import convert_to_dwg   # adjust import to your actual function name
output_dwg = convert_to_dwg(dwg_entities, output_path="output/result.dwg")
```

---

## 6. Expose Tile Crops to the Frontend

The pipeline saves each tile crop to `temp_data/tile_00_r0_c0.png` … `tile_07_r3_c1.png`. If your frontend needs to display them (e.g. for a tile-by-tile review UI), serve the folder as a static route:

```python
# Flask
from flask import send_from_directory

@app.route("/tiles/<filename>")
def serve_tile(filename):
    return send_from_directory("temp_data", filename)

# FastAPI
from fastapi.staticfiles import StaticFiles
app.mount("/tiles", StaticFiles(directory="temp_data"), name="tiles")
```

Tile filenames follow the pattern: `tile_{id:02d}_r{row}_c{col}.png`  
Debug overlay: `temp_data/debug_tile_grid.png`

---

## 7. Coordinate System Quick-Reference

Use this table whenever you need to move coordinates between spaces:

| Transform | Formula | Where it happens |
|---|---|---|
| Tile-local → Global | `global_x = local_x + spec.x_offset` | `local_to_global()` in pipeline |
| Global → Tile-local | `local_x = global_x - spec.x_offset` | Needed if you re-project back |
| Global pixel → DWG unit | `dwg_x = global_x / dpi * 25.4` (mm) | In your ODA converter |
| Global pixel → A4 mm (600 DPI) | `mm = px / 600 * 25.4` | Sanity-check dimensions |

> **A4 at 600 DPI:** 4,960 × 7,016 px ≈ 210 × 297 mm. One pixel ≈ 0.042 mm.

---

## 8. Configuration Cheat-Sheet

All tunable parameters live in the `run_pipeline()` call — nothing is hard-coded:

| Parameter | Default | When to change |
|---|---|---|
| `dpi` | `600` | Lower to `300` for speed during dev; raise to `1200` for ultra-fine drawings |
| `n_rows` | `4` | Increase for very tall drawings with dense annotation at top/bottom |
| `n_cols` | `2` | Increase for wide drawings (e.g. A3 landscape) |
| `overlap_frac` | `0.12` | Raise to `0.20` if features near tile edges are still being clipped |
| `nms_iou_threshold` | `0.45` | Lower (e.g. `0.30`) if duplicate detections survive; raise if valid detections are being merged |
| `save_tiles` | `True` | Set `False` in production to skip disk writes and speed up throughput |

---

## 9. Testing the Integration

Run these in order to verify each layer before touching production:

```bash
# 1. Synthetic test — no PDF needed, confirms the pipeline itself is healthy
python caed_tiling_pipeline.py
# Expected: "=== Self-test PASSED ===" + output/synthetic_debug.png

# 2. Real PDF test — checks pdf2image + your DPI settings
python - <<'EOF'
from caed_tiling_pipeline import run_pipeline
r = run_pipeline("temp_data/your_test_drawing.pdf", output_dir="temp_data")
print(f"Tiles: {len(r['specs'])}  |  Detections: {len(r['final_detections'])}")
EOF

# 3. API smoke test (Flask example)
curl -X POST http://localhost:5000/convert-tiled \
     -F "file=@temp_data/your_test_drawing.pdf" | python -m json.tool
```

---

## 10. Common Errors & Fixes

| Error | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: pdf2image` | venv not active or dep missing | `pip install -r requirements.txt` with venv active |
| `PDFPageCountError` or blank image | Poppler not installed system-wide | `sudo apt install poppler-utils` (Ubuntu) / `brew install poppler` (Mac) |
| `ImportError: libGL.so.1` | `opencv-python` (GUI) vs headless | Switch to `opencv-python-headless` in requirements.txt |
| Tiles look black / zero-size | `x_end <= x_start` from bad DPI | Confirm the PDF renders at the expected size; print `spec.width, spec.height` |
| Duplicate detections survive NMS | `iou_threshold` too high | Lower `nms_iou_threshold` to `0.30` |
| Features cut at tile edges | Overlap too small | Raise `overlap_frac` to `0.18`–`0.20` |
| `temp_data/` fills up fast | `save_tiles=True` in production | Set `save_tiles=False` or add a cleanup job |
