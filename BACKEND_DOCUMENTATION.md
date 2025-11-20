# Narrative API - Complete Backend Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [API Endpoints](#api-endpoints)
4. [Services](#services)
5. [Data Models & Schemas](#data-models--schemas)
6. [Configuration](#configuration)
7. [Dependencies & Integrations](#dependencies--integrations)
8. [Error Handling](#error-handling)
9. [Deployment](#deployment)

---

## Overview

**Project:** Narrative API (narrative-api)
**Version:** 0.1.0
**Framework:** FastAPI + Uvicorn
**Python Version:** 3.12+

The Narrative API is a backend service for generating AI-powered product video thumbnails. It analyzes product videos, intelligently samples frames, and uses Google Gemini 2.5 Flash Lite to select the best representative frames for product listings.

### Key Features

- Video processing with dynamic frame sampling
- AI-powered frame selection using Google Gemini
- GPU-accelerated video decoding (CUDA)
- Automatic first/last frame skipping
- Full-resolution output with base64 encoding
- Support for URL-based and file upload inputs

---

## Architecture

### Project Structure

```
Narrative-API/
├── app/
│   ├── __init__.py
│   ├── main.py                      # FastAPI application entry point
│   ├── dependencies.py              # Dependency injection
│   ├── exceptions.py                # Custom exceptions
│   ├── config/
│   │   └── settings.py              # Environment configuration
│   ├── models/                      # Database models (future use)
│   ├── schemas/
│   │   ├── example_schema.py        # Example schemas
│   │   └── thumbnail_schema.py      # Thumbnail API schemas
│   ├── routers/
│   │   ├── health.py                # Health check endpoint
│   │   ├── analysis.py              # Dynamic thumbnail analysis
│   │   ├── thumbnails.py            # File upload thumbnails
│   │   └── example_router.py        # Example router template
│   └── services/
│       └── thumbnail_service.py     # Core video processing service
├── pyproject.toml                   # Project dependencies
├── Dockerfile                       # Container configuration
└── Makefile                         # Build and deployment tasks
```

### Design Patterns

- **Dependency Injection:** FastAPI `Depends()` for service instantiation
- **Service Pattern:** Business logic encapsulated in service classes
- **Schema/DTO Pattern:** Pydantic models for request/response validation
- **Async/Await:** Non-blocking I/O with thread pool for CPU-bound operations

---

## API Endpoints

### Root Endpoint

**GET /**

Returns welcome message and API information.

**Response:**
```json
{
  "message": "Welcome to Narrative API",
  "version": "0.1.0",
  "docs": "/docs"
}
```

---

### Health Check

**GET /health**

Check API server status.

**Tags:** Health

**Response:**
```json
{
  "status": "healthy",
  "message": "API is running"
}
```

---

### Dynamic Thumbnail Generation (URL-based)

**POST /analysis/dynamic-thumbnails**

Generate product thumbnails from a video URL using AI.

**Tags:** Analysis

**Request Body:**
```json
{
  "video_url": "https://your-s3-url.com/video.mp4"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| video_url | string | Yes | S3 signed URL or public URL of the video |

**Response:**
```json
{
  "thumbnails": [
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD..."
  ],
  "count": 3
}
```

| Field | Type | Description |
|-------|------|-------------|
| thumbnails | array[string] | Base64-encoded JPEG images (3 for ≤15s, 5 for >15s) |
| count | integer | Number of thumbnails returned (1-5) |

**Processing Pipeline:**
1. Download video from URL
2. Sample ~100 frames using ffmpeg (540p, dynamic rate)
3. Skip first 3 and last 3 frames
4. Send frames to Gemini 2.5 Flash Lite for analysis
5. Extract full-resolution frames at selected indices
6. Return base64-encoded JPEG images (95% quality)

**Error Responses:**

| Status | Description |
|--------|-------------|
| 400 | Video URL is invalid or inaccessible |
| 400 | Video format not supported or file is corrupted |
| 500 | Model internal error |

**Example:**
```bash
curl -X POST "http://localhost:8000/analysis/dynamic-thumbnails" \
  -H "Content-Type: application/json" \
  -d '{"video_url": "https://your-video-url.mp4"}'
```

---

### Thumbnail Generation (File Upload)

**POST /thumbnails/generate**

Generate product thumbnails from uploaded video file.

**Tags:** Thumbnails

**Request:** Multipart/form-data

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| video | file | Yes | Video file to process |

**Response:**
```json
{
  "thumbnails": [
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD...",
    "/9j/4AAQSkZJRgABAQAAAQABAAD..."
  ],
  "count": 5
}
```

**Processing Logic:**
- ≤15 seconds video: Returns 3 thumbnails
- >15 seconds video: Returns 5 thumbnails
- Skips first 3 and last 3 frames
- Full-resolution JPEG output (95% quality)

**Error Responses:**

| Status | Description |
|--------|-------------|
| 500 | Failed to generate thumbnails |

**Example:**
```bash
curl -X POST "http://localhost:8000/thumbnails/generate" \
  -F "video=@videos/my_video.mp4" \
  --max-time 180
```

```python
import requests
import base64

with open("video.mp4", "rb") as f:
    response = requests.post(
        "http://localhost:8000/thumbnails/generate",
        files={"video": ("video.mp4", f, "video/mp4")},
        timeout=180
    )

data = response.json()
for i, thumb in enumerate(data["thumbnails"], 1):
    with open(f"thumb_{i}.jpg", "wb") as out:
        out.write(base64.b64decode(thumb))
```

---

### Example Endpoints (Template)

**POST /examples/**

Create example resource (template implementation).

**GET /examples/{example_id}**

Get example by ID (template implementation).

---

## Services

### DynamicThumbnailService

**File:** `app/services/thumbnail_service.py`

Core service for video processing and intelligent frame selection using Google Gemini API.

#### Constructor

```python
DynamicThumbnailService(google_api_key: str, save_raw_frames_dir: str = None)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| google_api_key | string | Google API key for Gemini |
| save_raw_frames_dir | string | Optional directory to save raw frames for debugging |

#### Main Method

```python
async def generate_dynamic_thumbnails(video_bytes: bytes) -> List[str]
```

**Input:** Raw video file bytes
**Output:** List of base64-encoded JPEG images

**Processing Steps:**
1. Save video to temporary file
2. Get video duration using OpenCV
3. Determine thumbnail count (3 if ≤15s, 5 if >15s)
4. Sample ~100 frames using ffmpeg
5. Send to Gemini for intelligent selection
6. Extract full-resolution frames
7. Base64 encode and return
8. Clean up temporary files

#### Internal Methods

| Method | Purpose |
|--------|---------|
| `_save_video_bytes()` | Save video bytes to temporary file |
| `_get_video_duration()` | Get video duration using OpenCV |
| `_sample_frames_with_ffmpeg()` | Sample frames with dynamic fps, 540p scaling |
| `_select_best_with_gemini()` | AI-powered frame selection |
| `_extract_fullres_frames()` | Extract full-resolution frames at selected indices |
| `_cleanup_directory()` | Remove temporary files |

#### Frame Sampling Logic

- **Target:** ~100 frames per video
- **Skip:** First 3 and last 3 frames (avoid intros/outros)
- **Scaling:** 540p while maintaining aspect ratio
- **GPU Acceleration:** CUDA hardware decoding when available
- **Format:** JPEG with quality=5 (fast encoding for sampling)

#### Gemini Selection Prompt

**For short videos (≤15s):**
- Focus on entire product visibility
- Crystal clear image quality
- Central focus on product
- Single best angle

**For longer videos (>15s):**
- Variety of angles/perspectives
- Best representation of product
- Quality and composition

**Response Format:** Comma-separated frame numbers sorted BEST to WORST
Example: `"12,5,18"` or `"3,7,12,18,25"`

---

## Data Models & Schemas

### Request Schemas

#### DynamicThumbnailRequest

```python
class DynamicThumbnailRequest(BaseModel):
    video_url: str  # S3 signed URL or public URL
```

#### ThumbnailRequest

```python
class ThumbnailRequest(BaseModel):
    video_data: str  # Base64-encoded video data
```

#### ExampleRequest

```python
class ExampleRequest(BaseModel):
    name: str           # Required, min_length=1
    value: int          # Required, ge=0
    description: str    # Optional
```

### Response Schemas

#### ThumbnailResponse

```python
class ThumbnailResponse(BaseModel):
    thumbnails: List[str]  # Base64-encoded JPEG images (1-5)
    count: int             # Number of thumbnails (1-5)
```

**Constraints:**
- `thumbnails`: min_length=1, max_length=5
- `count`: ge=1, le=5

#### DynamicThumbnailResponse

```python
class DynamicThumbnailResponse(BaseModel):
    thumbnails: List[str]  # Base64-encoded JPEG images
    count: int             # Number of thumbnails
```

#### ExampleResponse

```python
class ExampleResponse(BaseModel):
    id: int        # Unique identifier
    name: str      # Name field
    value: int     # Value field
    status: str    # Status field
```

---

## Configuration

### Settings Class

**File:** `app/config/settings.py`

```python
class Settings(BaseSettings):
    # App settings
    app_name: str = "Narrative API"
    app_version: str = "0.1.0"
    debug: bool = False

    # API settings
    api_prefix: str = "/api/v1"

    # Google Gemini settings
    google_api_key: str  # REQUIRED

    # Database settings
    database_url: Optional[str] = None

    # Security settings
    secret_key: Optional[str] = None
    access_token_expire_minutes: int = 30

    class Config:
        env_file = ".env"
        case_sensitive = False
        extra = "ignore"
```

### Environment Variables

**Required:**
- `GOOGLE_API_KEY` - Google Generative AI API key

**Optional:**
- `DEBUG` - Enable debug mode
- `DATABASE_URL` - Database connection string
- `SECRET_KEY` - Application secret key
- `GOOGLE_APPLICATION_CREDENTIALS` - Path to service account JSON

### Example .env File

```env
GOOGLE_API_KEY=AIzaSy...your_key_here...
DEBUG=False
```

---

## Dependencies & Integrations

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| fastapi | >=0.115.0 | Web framework |
| uvicorn | >=0.32.0 | ASGI server |
| pydantic | >=2.9.0 | Data validation |
| pydantic-settings | >=2.6.0 | Configuration management |
| google-genai | >=1.49.0 | Google Gemini API client |
| httpx | >=0.27.0 | Async HTTP client |
| python-dotenv | >=1.2.1 | Environment variable loading |
| opencv-python | >=4.10.0 | Video/image processing |
| python-multipart | >=0.0.20 | Multipart form data parsing |

### System Dependencies

- **FFmpeg** - Video frame extraction and processing
- **libgl1** - OpenGL library for OpenCV
- **libglib2.0-0** - GLib library for OpenCV
- **NVIDIA CUDA** - Optional GPU acceleration

### External API Integrations

#### Google Generative AI (Gemini)

**Model:** `gemini-2.0-flash-lite`

**Configuration:**
```python
types.GenerateContentConfig(
    max_output_tokens=100,
    temperature=0.1  # Consistent results
)
```

**Authentication:** API key-based

**Usage:**
- Frame selection and analysis
- Product thumbnail quality assessment
- Best frame identification

---

## Error Handling

### Custom Exceptions

#### CustomException (Base)

```python
class CustomException(Exception):
    def __init__(self, message: str, status_code: int = 400):
        self.message = message
        self.status_code = status_code
```

#### NotFoundException

```python
class NotFoundException(CustomException):
    def __init__(self, message: str = "Resource not found"):
        super().__init__(message, status_code=404)
```

#### ValidationException

```python
class ValidationException(CustomException):
    def __init__(self, message: str = "Validation error"):
        super().__init__(message, status_code=422)
```

### HTTP Error Codes

| Code | Scenario |
|------|----------|
| 200 | Successful thumbnail generation |
| 400 | Invalid input, inaccessible URL, unsupported format |
| 422 | Pydantic validation error |
| 500 | Processing failure, model errors, ffmpeg errors |

### Error Response Format

```json
{
  "detail": "Error message describing what went wrong"
}
```

---

## Deployment

### Docker Configuration

**Base Image:** `python:3.12-slim`

**System Dependencies:**
- ffmpeg
- libgl1
- libglib2.0-0

**Exposed Port:** 8080

**Start Command:**
```bash
uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8080}
```

### Makefile Commands

| Command | Description |
|---------|-------------|
| `make install` | Install dependencies via UV |
| `make dev` | Run development server with hot reload |
| `make build` | Build Docker image |
| `make run` | Run Docker container |
| `make sync` | Sync frozen dependencies |
| `make test` | Run tests with pytest |
| `make clean` | Clean temporary files |
| `make format` | Format code with black and isort |
| `make deploy` | Deploy to Google Cloud Run |

### Cloud Run Configuration

**Resources:**
- Memory: 16 GB
- CPU: 8 cores
- Timeout: 540 seconds (9 minutes)
- Allow unauthenticated: Yes
- Region: us-central1

**Environment Variables:**
- `GOOGLE_API_KEY`
- `USE_GPU_ACCELERATION`

### Quick Start

```bash
# Install dependencies
make install

# Set environment variables
export GOOGLE_API_KEY='your-key-here'

# Run development server
make dev

# Access API documentation
# Swagger UI: http://localhost:8000/docs
# ReDoc: http://localhost:8000/redoc
```

---

## Middleware

### CORS Configuration

**File:** `app/main.py`

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],       # WARNING: Not production-safe
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Note:** Current CORS configuration allows all origins. This should be restricted in production.

---

## Logging

### Configuration

```python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    datefmt='%H:%M:%S'
)
```

### Logged Information

- Processing step timings
- Video file sizes
- Video duration and thumbnail count decisions
- Frame sampling counts
- Gemini API responses
- Frame extraction details
- Total processing time
- Error details

### Example Log Output

```
🎬 Starting thumbnail generation process
💾 STEP 1: Saving video data...
   Video size: 12.34 MB
✅ Video saved in 0.45s

⏱️  STEP 2: Getting video duration...
✅ Duration: 18.25s -> 5 thumbnails

🎞️  STEP 3: Sampling frames...
✅ Sampled 102 frames in 8.92s

🤖 STEP 4: Sending frames to Gemini...
✅ Gemini analysis completed in 45.23s

📦 STEP 5: Extracting full-resolution frames...
✅ Extracted 5 full-res images in 1.23s

🎉 COMPLETE! Total time: 56.45s
```

---

## Performance Characteristics

### Typical Processing Times

| Step | Duration |
|------|----------|
| Video download | 2-5 seconds |
| Frame sampling | 5-10 seconds |
| Gemini analysis | 30-60 seconds |
| Frame extraction | 1-3 seconds |
| **Total** | **35-80 seconds** |

### Resource Usage

- **GPU Acceleration:** 2-5x speedup on frame extraction
- **Memory:** 50-200 MB temporary storage per request
- **CPU:** Multi-threaded ffmpeg and OpenCV operations

### Scalability

- Async request handling
- Thread pool for CPU-bound operations
- Stateless processing (no database)
- Suitable for containerized deployment

---

## Security Considerations

### Current Implementation

- No user authentication
- No API key validation at endpoints
- Open CORS policy
- Environment variable-based secrets

### Recommendations for Production

1. Implement JWT or API key authentication
2. Restrict CORS origins to specific domains
3. Add rate limiting
4. Use secret management service (e.g., Google Secret Manager)
5. Implement request logging and monitoring
6. Add input validation beyond Pydantic
7. Configure firewall rules
8. Enable HTTPS only

---

## Future Enhancements

### Not Yet Implemented

- Database persistence layer
- User authentication and authorization
- Rate limiting
- Response caching
- Background job queue (Celery)
- API versioning
- Comprehensive monitoring
- Auto-scaling configuration

### Planned Features

- Result storage and retrieval
- User accounts and usage tracking
- Webhook notifications
- Batch processing
- Custom thumbnail selection criteria
- Additional AI models support

---

## API Documentation Access

- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc
- **OpenAPI Schema:** http://localhost:8000/openapi.json

---

## Support & Resources

- **API Guide:** `API_GUIDE.md`
- **Environment Setup:** `ENV_SETUP.md`
- **GPU Setup:** `GPU_SETUP.md`
- **Quick Start:** `QUICKSTART.md`
- **Example Code:** `example_usage.py`, `example_usage_analysis.py`
- **Test Scripts:** `test_video.py`, `test_api_endpoint.py`
