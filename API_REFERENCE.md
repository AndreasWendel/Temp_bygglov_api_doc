# Huddinge Bygglov - API Quick Reference

## Base URL

- **Local Development**: `http://127.0.0.1:8000`
- **Production**: `http://huddinge-bygglov-api.azurewebsites.net`

## Authentication

All endpoints (except `/`) require API key authentication via header:

```
access_token: your-api-key-here
```

---

## Endpoints

### 1. Health Check

```http
GET /
```

**Response**: `200 OK`

```json
{
  "message": "Huddinge Bygglov API is running"
}
```

---

### 2. Authentication Status

```http
GET /auth
```

**Headers**:

```
access_token: your-api-key
```

**Response**: `200 OK`

```json
{
  "status": "authenticated"
}
```

**Error**: `403 Forbidden`

```json
{
  "detail": "Could not validate credentials"
}
```

---

### 3. Classify Documents

```http
POST /classify_handling
```

**Headers**:

```
access_token: your-api-key
Content-Type: multipart/form-data
```

**Body** (form-data):

```
handlingar: file1.pdf
handlingar: file2.xlsx
handlingar: file3.docx
```

**Supported File Types**:

- PDF (`.pdf`)
- Word (`.docx`)
- Image (`.png`, `.jpg`, `.jpeg`, `.tiff`, `.tif`)

**Response**: `200 OK`

```json
{
  "handlingar": [
    {
      "handling": "document.pdf",
      "type": "Fasadritning",
      "predicted": "Fasadritning",
      "confidence": 0.9876,
      "tag": "AI-classified"
    }
  ]
}
```

```json
{
  "handlingar": [
    {
      "handling": "document_2.pdf",
      "type": "not classified",
      "predicted": "Markplaneringsritning",
      "confidence": 0.5021,
      "tag": "manual_review_required"
    }
  ]
}
```

**Response Fields**:

- `handling`: Original filename
- `type`: Classification result (The predicted class or "not classified")
- `predicted`: Models prediction (currently only returns first page prediction)
  - `possible prediction values`: "Fasadritning", "Konstruktionsritning", "Markplaneringsritning", "Övrig_handlingstyp", "Planritning", "Sektionsritning", "Situationsplan"
- `confidence`: Confidence score for the prediction (0.0 to 1.0)
- `tag`: Classification tag ("AI-classified" or "Manual review required" if low confidence)

**Error Responses**:

`400 Bad Request` - Invalid file format

```json
{
  "detail": "Unsupported file type"
}
```

`403 Forbidden` - Invalid API key

```json
{
  "detail": "Could not validate credentials"
}
```

`500 Internal Server Error` - Processing error

```json
{
  "detail": "Error processing document"
}
```

---

## Document Classes

| Class | Description |
|-------|-------------|
| Fasadritning | Building facade plans |
| Konstruktionsritning | Structural drawings |
| Markplaneringsritning | Land/site planning |
| Planritning | Building floor layouts |
| Sektionsritning | Cross-section views |
| Situationsplan | Overall site layout |
| Övrig_handlingstyp | Other document type |

---

## Code Examples

### Python (requests)

```python
import requests

url = "http://127.0.0.1:8000/classify_handling"
headers = {"access_token": "your-api-key"}

# Single file
with open("document.pdf", "rb") as f:
    files = {"handlingar": ("document.pdf", f, "application/pdf")}
    response = requests.post(url, headers=headers, files=files)
    print(response.json())

# Multiple files
files = [
    ("handlingar", ("doc1.pdf", open("doc1.pdf", "rb"), "application/pdf")),
    ("handlingar", ("doc2.pdf", open("doc2.pdf", "rb"), "application/pdf")),
]
response = requests.post(url, headers=headers, files=files)
print(response.json())
```

### Python (httpx - async)

```python
import httpx
import asyncio

async def classify_documents():
    url = "http://127.0.0.1:8000/classify_handling"
    headers = {"access_token": "your-api-key"}
    
    async with httpx.AsyncClient() as client:
        with open("document.pdf", "rb") as f:
            files = {"handlingar": ("document.pdf", f, "application/pdf")}
            response = await client.post(url, headers=headers, files=files)
            return response.json()

result = asyncio.run(classify_documents())
# may run await instead "result = await classify_documents()" if event loop is interfering with asyncio
print(result)
```

### cURL

```bash
# Single file
curl -X POST "http://127.0.0.1:8000/classify_handling" \
  -H "access_token: your-api-key" \
  -F "handlingar=@document.pdf"

# Multiple files
curl -X POST "http://127.0.0.1:8000/classify_handling" \
  -H "access_token: your-api-key" \
  -F "handlingar=@document1.pdf" \
  -F "handlingar=@document2.xlsx" \
  -F "handlingar=@document3.docx"
```

---

## Rate Limiting & Best Practices

1. **Batch Requests**: The api has the possibility to handle multiple files in a single request but as the model will only handle one file at a time, there wont be a performance gain from sending multiple files in a single request.
2. **File Size**: Keep files under 20MB for optimal performance, upper limit is 50MB
3. **Timeout**: Set appropriate timeouts (60-90 seconds recommended)
4. **Error Handling**: Always implement retry logic with exponential backoff
5. **API Key Security**: Never commit API keys to version control

---

## Interactive Documentation

Visit these URLs when the server is running:

- **Swagger UI**: `https://huddinge-bygglov-api.azurewebsites.net/docs`
- **ReDoc**: `https://huddinge-bygglov-api.azurewebsites.net/redoc`
- **OpenAPI Schema**: `https://huddinge-bygglov-api.azurewebsites.net/openapi.json`

---

## Environment Configuration

Key environment variables affecting API behavior:

```bash
# Minimum confidence threshold (0.0 to 1.0), this has been optimized for the current model and current test data
CONFIDENCE_THRESHOLD=0.9895655917448025

# Model file to use
MODEL_NAME=your-model-file.pkl

# Enable GPU acceleration if nvidia gpu is available, currently only CPU is supported by azures api compute
USE_GPU=false
```

---

## Monitoring & Logging

The API logs the following information:

- Request timestamp
- File names and sizes
- Classification results
- Response time
- Confidence scores
- Errors and exceptions

Logs are saved to the configured `LOG_PATH` and `FILE_PATH`.

---
