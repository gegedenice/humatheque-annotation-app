# Humatheque Annotation App

Gradio app to create layout annotations (bounding boxes) on document images and save them through an API.

## What this app does

- Loads images from:
  - a direct image URL
  - a MinIO bucket browser (optional)
- Creates boxes with a 2-click workflow:
  - first click sets corner 1
  - second click sets corner 2
- Stores one label (`block_code`) per box
- Lets users relabel each box from a table
- Saves:
  - one case via `POST /cases/upsert`
  - one annotation per box via `POST /layout-annotations`

## Tech stack

- Python 3.11+
- Gradio UI
- Requests for API calls
- Pillow + NumPy for image handling/rendering
- python-dotenv for local env config
- MinIO client for object storage sync/load

## Project files

- `app.py`: full application (UI + logic + API integration)
- `requirements.txt`: declared Python dependencies
- `.example.env`: example environment variables

## Prerequisites

- Python 3.11+
- Reachable backend API implementing the endpoints used in `app.py`
- Seed data in backend:
  - at least 1 campaign (`GET /campaigns`)
  - at least 1 block type (`GET /block-types`)

## Installation

1. Create and activate a virtual environment.
2. Install dependencies:

```bash
pip install -r requirements.txt
pip install minio
```

Note: `app.py` imports `minio`, but `requirements.txt` currently does not include it.

## Configuration

Create `.env` in the project root (or copy from `.example.env`) and set:

```env
API_BASE_URL=http://localhost:8000
API_KEY=your_api_key

# Optional: enable MinIO file tree/image loading
MINIO_ENDPOINT_URL=http://localhost:9000
MINIO_ACCESS_KEY=minio_access_key
MINIO_SECRET_KEY=minio_secret_key
MINIO_BUCKET_NAME=images
```

### Environment variables used by the app

- `API_BASE_URL`: base URL for backend API (default: `http://localhost:8000`)
- `API_KEY`: sent as `X-API-Key` on POST requests
- `MINIO_ENDPOINT_URL`: MinIO endpoint
- `MINIO_ACCESS_KEY`: MinIO access key
- `MINIO_SECRET_KEY`: MinIO secret key
- `MINIO_BUCKET_NAME`: MinIO bucket name (default: `images`)

If MinIO variables are missing, the app still runs, but MinIO features are disabled.

## Run the app

```bash
python app.py
```

or:

```bash
uv run app.py
```

The app launches on:

- Host: `0.0.0.0`
- Port: `7860`

## Annotation workflow

1. Select a campaign.
2. Load an image:
   - paste URL and click **Load image**, or
   - pick a file in **MinIO Bucket Files**
3. Draw boxes on the canvas (2 clicks per box).
4. Use the table to select a box row and adjust its label.
5. Fill metadata (document info, annotator, notes).
6. Click **Save all drawn rectangles**.
7. Inspect API response in **Status** and **Stored annotations (JSON)**.

## API interactions

### Reads

- `GET /campaigns`
- `GET /block-types`
- `GET /layout-annotations?campaign_id=...&case_id=...` (after save)

### Writes

- `POST /cases/upsert`
- `POST /layout-annotations` (one request per box)

### Headers behavior

- `POST` requests include:
  - `X-API-Key: <API_KEY>`
  - `Accept: application/json`
  - `Content-Type: application/json`
- `GET` requests currently do **not** include API headers in `app.py`.

## Data model used in UI state

Each annotation row is stored as:

```text
[x1, y1, x2, y2, block_code]
```

Coordinates are saved as integer pixel values, based on the loaded image dimensions.

## Notes and limitations

- `API_KEY` is only applied to POST calls in current code.
- On startup, `make_app()` calls MinIO sync directly; if MinIO is not configured, startup may fail unless this behavior is adjusted.
- `BUCKET` is hardcoded to `"images"` for sync pathing, while `MINIO_BUCKET_NAME` is used for object reads.
- `build_case_name()` uses `doc_id` + `page_no` (`<doc_id>_p0001` format) and falls back to `doc_unknown`.
- The app expects backend schema and endpoint behavior to match payloads built in `save_annotations()`.

## Troubleshooting

- **No campaigns/block types found**
  - seed backend data for `/campaigns` and `/block-types`
- **Image load fails from URL**
  - verify URL is directly accessible and returns image bytes
- **MinIO refresh/load errors**
  - re-check MinIO endpoint, credentials, and bucket values
- **Save errors**
  - confirm `API_BASE_URL`, `API_KEY`, and backend endpoint compatibility
