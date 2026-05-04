# ImgProxy URL Rule

## Rule

Only S3 URLs may be passed to `generate_imgproxy_url()`. Never pass Replicate URLs.

## Why

The imgproxy service is only configured to proxy images from our S3 bucket. Replicate URLs are temporary and external, and imgproxy cannot access them. Passing a Replicate URL to imgproxy will result in broken images.

## Correct Pattern

```python
# CORRECT - Only use S3 URLs, never fallback to replicate
imgproxy_url = generate_imgproxy_url(image.s3_url, width=380, height=0) if image.s3_url else None
```

```python
# WRONG - passes source_url which may be a Replicate URL
source_url = image.s3_url or image.replicate_url
imgproxy_url = generate_imgproxy_url(source_url, width=380, height=0) if image.s3_url else source_url
```

```python
# WRONG - never expose replicate_url in responses
image_url = image.s3_url or image.replicate_url
```

## Key Principles

1. **Only S3 URLs go to imgproxy** - Pass `image.s3_url` directly to `generate_imgproxy_url()`, never a combined `source_url`
2. **No replicate_url fallback** - If no S3 URL exists, return `None` for the image URL
3. **Never expose replicate URLs** - Don't include `replicate_url` in API responses for images

## Affected Files

All files that return image URLs to the frontend:

- `backend/app/crud/crud_gallery.py` - `get_image_data()` function
- `backend/app/crud/crud_image_curation.py` - Image data functions
- `backend/app/routers/image/generate/image.py` - Image endpoints
- `backend/app/routers/gallery.py` - Gallery endpoints
- `backend/app/routers/admin_gallery.py` - Admin gallery
- `backend/app/routers/admin_talent_featured_images.py` - Talent featured images
- `backend/app/routers/user.py` - Download history
- Any new endpoints that return image URLs

## Verification

Search for this anti-pattern:

```bash
grep -n "generate_imgproxy_url(source_url" backend/app/**/*.py
```

Result should be empty.
