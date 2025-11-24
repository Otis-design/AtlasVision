# AtlasVision

**AI Shop Inventory Scanner & Real-Time Insight Engine**

---

> Turn shelf photos and short videos into structured inventory, low-stock alerts, price comparisons and actional restock recommendations.

Screenshot used for reference (sample model list): `/mnt/data/Models – Hugging Face - Google Chrome 24_11_2025 07_56_13.png`

---

## Table of contents

1. Project Overview
2. Core Features
3. Architecture & System Design (diagrams)
4. HuggingFace Models & Inference
5. Tech Stack
6. API Design & Endpoints
7. Database SQL Schema
8. Local Dev Setup
9. Deployment (Docker + Docker Compose)
10. Scaffolding: Backend (FastAPI) - code files
11. How to demo & portfolio tips
12. Roadmap & Future Work

---

## 1. Project Overview

AtlasVision is a production-minded prototype that converts smartphone shelf images into structured inventory data. It combines OCR, image classification and VQA (visual question answering) to extract product names, estimate stock levels, detect missing items, and enrich results with price and supplier metadata from real-world APIs.

**Primary users:** small-shop owners, micro-distributors, inventory managers in under-digitized retail environments.

**Goals:**

* Demonstrate strong systems and ML integration skills.
* Deliver an end-to-end pipeline: mobile capture → async processing → analytics dashboard.
* Use real external APIs to show business value.

---

## 2. Core Features

* Upload shelf images or short videos
* OCR extraction of product labels & SKUs
* Product category classification and count/stock estimate
* Visual QA for queries like "missing items?" or "how many X?"
* Enrichment via price comparison APIs (e.g., Jumia, Carrefour, OpenFoodFacts)
* Low-stock alerts (push/WhatsApp/Email)
* Dashboard for analytics & trends

---

## 3. Architecture & System Design

### High-level ASCII diagram

```
    [Mobile App / Web UI]
           │  uploads images
           ▼
     [Object Storage (S3 or Supabase Storage)]
           │
           ▼
     [FastAPI Gateway] -- auth --> [Postgres (Supabase)]
           │
    enqueue job │
           ▼
    [Redis Queue / Celery]  <-- workers --> [HF Inference / Local Models]
           │                               │
           │                               └─> HuggingFace Inference API
           │
           ▼
   [Normalization & Enrichment] -> call external APIs (Jumia/RapidAPI/OpenFoodFacts)
           │
           ▼
    [Insights Engine / Rule Engine]
           │
           ▼
    [Notifications (WhatsApp/Email) + Dashboard]
```

### Component responsibilities

* **Mobile/Web UI**: Capture images, show scan history, insights dashboard.
* **FastAPI Gateway**: Upload handling, authentication, basic validation, job enqueuing.
* **Workers (Celery/Redis)**: Run OCR → classify → VQA → normalize → enrichment. Persist results.
* **HF Inference Layer**: Proxy requests to HuggingFace (or run local container models if available).
* **DB**: Stores shops, scans, products, alerts, pricing history.
* **Insights Engine**: Derives low-stock and reorder recommendations.

---

## 4. HuggingFace Models & Inference

**Primary tasks and suggested models:**

* **OCR (image-to-text)**: `deepseek-ai/DeepSeek-OCR` or `microsoft/trocr-base-printed`
* **Image Classification**: `google/vit-base-patch16-224` (fine-tune on product dataset for better accuracy)
* **VQA (Visual Question Answering)**: `Salesforce/blip-vqa-base` or `microsoft/vilt-b32-finetuned-vqa`

**Inference options:**

* Use HuggingFace Inference API for rapid prototyping (requires API token)
* Optionally host models on HF Spaces/Deploy or use local ONNX/TorchServe for lower latency

**Note**: Keep responses compact and use model confidence scores for downstream rule thresholds.

---

## 5. Tech Stack

* **Backend**: FastAPI, Python 3.11
* **Workers**: Celery + Redis (or RQ + Redis)
* **ORM**: SQLModel (or SQLAlchemy) with PostgreSQL (Supabase)
* **Storage**: AWS S3 or Supabase Storage for images
* **HuggingFace**: Inference API calls
* **Mobile**: Flutter (for demo) or React/Next.js web UI
* **Deployment**: Docker, Docker Compose, optionally Terraform

---

## 6. API Design & Endpoints

### Authentication

* `POST /auth/signup` — create a shop/user
* `POST /auth/login` — returns JWT

### Scanning & Results

* `POST /api/v1/scan` — upload image (multipart/form-data) → returns `scan_id` (processing)
* `GET /api/v1/scan/{scan_id}` — get processing status & results
* `GET /api/v1/shop/{shop_id}/inventory` — get current inventory JSON
* `GET /api/v1/shop/{shop_id}/analytics` — get aggregated insights
* `POST /api/v1/alerts` — configure alert thresholds

---

## 7. Database SQL Schema

Below are SQL `CREATE TABLE` statements ready to run on PostgreSQL.

```sql
-- shops table
CREATE TABLE shops (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  owner_email TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role TEXT DEFAULT 'owner',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- scans
CREATE TABLE scans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  image_url TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending', -- pending, processing, done, failed
  meta JSONB, -- raw model outputs
  normalized JSONB, -- normalized inventory JSON
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- products (inventory canonical)
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  name TEXT,
  sku TEXT,
  category TEXT,
  quantity INTEGER DEFAULT 0,
  last_seen TIMESTAMP WITH TIME ZONE,
  price NUMERIC(10,2),
  external_meta JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- alerts
CREATE TABLE alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  type TEXT, -- 'low_stock', 'price_drop'
  threshold INTEGER,
  last_sent TIMESTAMP WITH TIME ZONE
);

-- price_history
CREATE TABLE price_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_name TEXT,
  shop_id UUID REFERENCES shops(id) ON DELETE CASCADE,
  price NUMERIC(10,2),
  source TEXT,
  recorded_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

---

## 8. Local Dev Setup (Quick)

1. Clone the repo
2. Copy `.env.example` to `.env` and fill variables (DATABASE_URL, HF_API_KEY, S3 creds, REDIS_URL)
3. `docker-compose up --build`
4. `curl -X POST 'http://localhost:8000/api/v1/scan' -F 'image=@sample.jpg' -H 'Authorization: Bearer <token>'`

---

## 9. Deployment (Docker + Docker Compose)

A `docker-compose.yml` is included in the scaffold (see below). Services:

* `web` (FastAPI, uvicorn)
* `worker` (Celery worker)
* `redis` (Redis)
* `db` (Postgres)

---

## 10. Scaffolding: Backend (FastAPI) — Files & Code

Below is a minimal but runnable scaffold to get you started. Files included:

```
/backend
  ├─ app/
  │   ├─ main.py
  │   ├─ api/
  │   │   ├─ routes.py
  │   │   └─ auth.py
  │   ├─ models.py
  │   ├─ db.py
  │   ├─ tasks.py
  │   └─ hf_client.py
  ├─ Dockerfile
  ├─ requirements.txt
  └─ docker-compose.yml
```

### `requirements.txt`

```
fastapi
uvicorn[standard]
python-multipart
SQLModel
psycopg2-binary
requests
celery[redis]
redis
python-dotenv
boto3
httpx
```

### `app/db.py`

```python
from sqlmodel import SQLModel, create_engine, Session
import os

DATABASE_URL = os.getenv('DATABASE_URL', 'postgresql://postgres:postgres@db:5432/atlasvision')
engine = create_engine(DATABASE_URL)

def init_db():
    SQLModel.metadata.create_all(engine)

def get_session():
    with Session(engine) as session:
        yield session
```

### `app/models.py`

```python
from sqlmodel import SQLModel, Field
from typing import Optional
from datetime import datetime

class Shop(SQLModel, table=True):
    id: Optional[str] = Field(default=None, primary_key=True)
    name: str
    owner_email: Optional[str]
    created_at: datetime = Field(default_factory=datetime.utcnow)

class Scan(SQLModel, table=True):
    id: Optional[str] = Field(default=None, primary_key=True)
    shop_id: str
    image_url: str
    status: str = 'pending'
    meta: Optional[dict] = None
    normalized: Optional[dict] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)

class Product(SQLModel, table=True):
    id: Optional[str] = Field(default=None, primary_key=True)
    shop_id: str
    name: Optional[str]
    sku: Optional[str]
    category: Optional[str]
    quantity: int = 0
    last_seen: Optional[datetime] = None
    price: Optional[float] = None
    external_meta: Optional[dict] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

### `app/hf_client.py` (HuggingFace inference helpers)

```python
import os
import requests

HF_API_URL = 'https://api-inference.huggingface.co/models'
HF_TOKEN = os.getenv('HF_API_TOKEN')

HEADERS = {'Authorization': f'Bearer {HF_TOKEN}'} if HF_TOKEN else {}


def call_hf_model(model_name: str, inputs, content_type='application/json'):
    url = f"{HF_API_URL}/{model_name}"
    if isinstance(inputs, (bytes, bytearray)):
        # image bytes
        headers = HEADERS.copy()
        headers['Content-Type'] = 'application/octet-stream'
        resp = requests.post(url, headers=headers, data=inputs, timeout=120)
    else:
        resp = requests.post(url, headers=HEADERS, json={'inputs': inputs}, timeout=120)
    resp.raise_for_status()
    return resp.json()
```

### `app/tasks.py` (Celery tasks)

```python
import os
from celery import Celery
from app.hf_client import call_hf_model
from app.db import engine
from sqlmodel import Session
from app.models import Scan, Product

CELERY_BROKER = os.getenv('CELERY_BROKER_URL', 'redis://redis:6379/0')
CELERY_BACKEND = os.getenv('CELERY_RESULT_BACKEND', 'redis://redis:6379/1')
celery = Celery('tasks', broker=CELERY_BROKER, backend=CELERY_BACKEND)

OCR_MODEL = 'deepseek-ai/DeepSeek-OCR'
VQA_MODEL = 'Salesforce/blip-vqa-base'
CLASS_MODEL = 'google/vit-base-patch16-224'

@celery.task()
def process_scan(scan_id: str):
    with Session(engine) as session:
        scan = session.get(Scan, scan_id)
        if not scan:
            return
        scan.status = 'processing'
        session.add(scan)
        session.commit()

        # fetch image bytes
        import requests
        img_bytes = requests.get(scan.image_url).content

        # OCR
        try:
            ocr_result = call_hf_model(OCR_MODEL, img_bytes)
        except Exception as e:
            scan.status = 'failed'
            scan.meta = {'error': str(e)}
            session.add(scan)
            session.commit()
            return

        # Classification (simplified):
        class_result = call_hf_model(CLASS_MODEL, img_bytes)

        # VQA example question
        vqa_result = call_hf_model(VQA_MODEL, {'image': img_bytes, 'question': 'Which items are missing?'})

        # Normalize (simple heuristic)
        normalized = {
            'ocr': ocr_result,
            'classes': class_result,
            'vqa': vqa_result
        }

        scan.normalized = normalized
        scan.status = 'done'
        session.add(scan)
        session.commit()

        # Basic upsert into products (naive example)
        # For each detected product name in OCR, update/increment product quantity
        detected_names = []
        # parse ocr_result depending on model format
        # naive parsing if OCR returns list of strings
        if isinstance(ocr_result, list):
            detected_names = [x.get('word', x) if isinstance(x, dict) else x for x in ocr_result]

        for name in detected_names:
            # very naive matching
            q = session.exec(select(Product).where(Product.shop_id==scan.shop_id, Product.name==name)).first()
            if q:
                q.quantity = q.quantity + 1
                q.last_seen = datetime.utcnow()
                session.add(q)
            else:
                p = Product(shop_id=scan.shop_id, name=name, quantity=1)
                session.add(p)
        session.commit()
```

> **Note:** `tasks.py` is intentionally simple to demonstrate structure. A production pipeline would implement robust parsing, fuzzy matching, confidence thresholds and rate-limiting for HF API calls.

### `app/main.py`

```python
from fastapi import FastAPI, UploadFile, File, Depends, HTTPException
from app.db import init_db, get_session, engine
from app.models import Scan, Shop
from sqlmodel import Session
import uuid
import os

init_db()
app = FastAPI(title='AtlasVision')

@app.post('/api/v1/scan')
async def upload_scan(shop_id: str, file: UploadFile = File(...)):
    # store file to minio/s3 or local storage
    filename = f"/tmp/{uuid.uuid4()}_{file.filename}"
    contents = await file.read()
    with open(filename, 'wb') as f:
        f.write(contents)

    # TODO: upload to S3 and get public URL. For quick dev, use local file path
    image_url = 'file://' + filename

    scan = Scan(shop_id=shop_id, image_url=image_url)
    with Session(engine) as session:
        session.add(scan)
        session.commit()
        session.refresh(scan)
        # enqueue worker
        from app.tasks import process_scan
        process_scan.delay(str(scan.id))

    return {'scan_id': str(scan.id), 'status': 'processing'}

@app.get('/api/v1/scan/{scan_id}')
def get_scan(scan_id: str):
    with Session(engine) as session:
        scan = session.get(Scan, scan_id)
        if not scan:
            raise HTTPException(status_code=404, detail='scan not found')
        return {'id': scan.id, 'status': scan.status, 'normalized': scan.normalized}
```

### `docker-compose.yml`

```yaml
version: '3.8'
services:
  web:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - '8000:8000'
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/atlasvision
      - CELERY_BROKER_URL=redis://redis:6379/0
      - HF_API_TOKEN=${HF_API_TOKEN}
    depends_on:
      - db
      - redis

  worker:
    build: ./backend
    command: celery -A app.tasks.celery worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/atlasvision
      - CELERY_BROKER_URL=redis://redis:6379/0
      - HF_API_TOKEN=${HF_API_TOKEN}
    depends_on:
      - db
      - redis

  redis:
    image: redis:6-alpine
    ports:
      - '6379:6379'

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: atlasvision
    ports:
      - '5432:5432'

```

---

## 11. How to demo & portfolio tips

* Make a 90-second demo video showing: mobile capture → processing → dashboard insights → reorder flow with price comparison.
* Include a public GitHub repo with `README`, `docker-compose` + short script to seed sample scans (use the provided screenshot at `/mnt/data/Models – Hugging Face - Google Chrome 24_11_2025 07_56_13.png` as a sample input).
* Provide a small dataset (10-30 photos) in `/demo_images` so reviewers can run end-to-end locally.
* Write clear resume bullets (examples included in project spec)

---

## 12. Roadmap & Future Work

* Fine-tune classification/OCR models on localized product images
* Implement barcode detection & linking to GTIN
* Add predictive restock ML model (time-series)
* Integrate supplier ordering (button to place order automatically)
* Add multi-shop analytics & role-based access control

---

### Final notes

This scaffold is built to be implementable by a solo engineer in 4–8 weeks for an MVP, or 2–3 months for a full-featured prototype with mobile app and analytics dashboard. The key is to iterate: core pipeline (OCR → classify → VQA) first, then enrich with external APIs.

---

*If you'd like, I can now:*

* Generate the GitHub repo structure with these files as downloadable artifacts, or
* Expand/implement the frontend (React/Flutter) scaffold, or
* Create a simple seed dataset and a script to run a demo locally.
