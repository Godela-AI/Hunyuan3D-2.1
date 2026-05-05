# Hunyuan3D-2.1 API Documentation

## Base URL

`http://localhost:8081` (pod-internal)  
`https://{api-gw}/dev/hunyuan3d/` (via API Gateway, when configured)

## Endpoints

### POST /generate

Synchronous 3D generation. Blocks until complete. Use for quick tests.

**Request:**
```json
{
  "image": "<base64 encoded image>",
  "prompt": null,
  "texture": true,
  "remove_background": true,
  "seed": 1234,
  "num_inference_steps": 5,
  "guidance_scale": 5.0,
  "octree_resolution": 256,
  "num_chunks": 8000,
  "face_count": 40000
}
```

**Input modes (provide one):**
- `image`: base64 encoded image (PNG/JPEG) -- image-to-3D
- `prompt`: text description (e.g. "a futuristic sports car") -- text-to-3D (requires `--enable_t2i` flag)

If both provided, `prompt` takes priority.

**Response (200):**
```json
{
  "uid": "abc123",
  "status": "completed",
  "model_base64": "<base64 encoded GLB file>"
}
```

### POST /send

Asynchronous 3D generation. Returns immediately with a uid. Poll `/status/{uid}` for results.

**Request:** Same as `/generate`

**Response (200):**
```json
{
  "uid": "abc123"
}
```

### GET /status/{uid}

Poll status of an async generation task.

**Response (200) -- in progress:**
```json
{
  "status": "processing",
  "model_base64": null,
  "message": null
}
```

**Response (200) -- completed:**
```json
{
  "status": "completed",
  "model_base64": "<base64 encoded GLB file>",
  "message": null
}
```

**Response (200) -- error:**
```json
{
  "status": "error",
  "model_base64": null,
  "message": "Error description"
}
```

### GET /health

Health check.

**Response (200):**
```json
{
  "status": "healthy",
  "worker_id": "abc123"
}
```

### GET /docs

Swagger UI for interactive API testing.

## Request Parameters

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `image` | string (base64) | null | Input image. Required if prompt not provided |
| `prompt` | string | null | Text prompt for text-to-3D. Required if image not provided |
| `texture` | bool | false | Generate PBR textures (slower, uses ~21GB VRAM) |
| `remove_background` | bool | true | Auto-remove background from input image |
| `seed` | int | 1234 | Random seed for reproducibility |
| `octree_resolution` | int | 256 | Mesh resolution (64-512, higher = more detail + slower) |
| `num_inference_steps` | int | 5 | Diffusion steps (1-20, higher = better quality + slower) |
| `guidance_scale` | float | 5.0 | Classifier-free guidance (0.1-20.0) |
| `num_chunks` | int | 8000 | Processing chunks (1000-20000) |
| `face_count` | int | 40000 | Max faces for textured mesh (1000-100000) |

## VRAM Requirements

| Mode | VRAM |
|------|------|
| Shape only (texture=false) | ~10 GB |
| Texture only | ~21 GB |
| Shape + texture | ~29 GB (sequential) |
| Text-to-3D (prompt) | +2 GB for SDXL |

A10G (24GB) can run shape + texture sequentially but not simultaneously.

## Text-to-3D Flow

When `prompt` is provided:
1. SDXL generates a 2D image from the text prompt (~5s)
2. Image is passed to Hunyuan3D shape generator (~10-30s)
3. Optional: texture generation on the mesh (~30-60s)

Server must be started with `--enable_t2i` flag to load the SDXL pipeline.
