---
title: Generative Expand v4 - Masks and Placement
description: Prepare an explicit mask or let Expand construct one from output size and placement.
---

# Generative Expand v4: masks and placement

**Deployment-dependent capability:** maskless placement mode requires the updated
v4 service. Confirm availability with your Adobe contact before adopting it.
This guide does not announce deployment or change an SLA.

Use `POST https://firefly-api.adobe.io/v4/images/expand`, without an `-async`
suffix. Supply either a prepared canvas and mask, or an original image with
optional output size and placement. Generative Fill still requires a mask.

## Upload your assets

Upload each PNG separately using `POST /v2/storage/image`. Copy the returned
`images[0].id` into `source.id`. The v4 field is **not** `source.uploadId`.
Alternatively, supply an accessible HTTPS `source.presignedUrl` with sufficient
validity for the asynchronous job to fetch it.

Every API request requires `Authorization: Bearer <ACCESS_TOKEN>` and
`x-api-key: <CLIENT_ID>`. Use an entitled Firefly Services project.

## Explicit-mask mode

Supply a prepared canvas and a same-sized grayscale mask. Black (`0`) designates
preservation and white (`255`) designates generation. Do not specify `size`
or `placement` alongside the mask; that combination is a validation error.

```json
{
  "input": {
    "image": {"source": {"id": "<canvas-id>"}},
    "mask": {"source": {"id": "<mask-id>"}}
  },
  "seeds": [333],
  "output": {"format": "png"}
}
```

## Placement mode: no mask upload

Upload the original, unpadded photograph and omit `input.mask`. The service
contain-fits it without cropping or upscaling, places it on a white canvas,
and constructs a mask that protects the image footprint and generates the
space outside it. Larger inputs are downscaled if necessary.

An image-only request defaults to a centered **2048 x 2048** output:

```json
{
  "input": {
    "image": {"source": {"id": "<original-image-id>"}}
  }
}
```

For a 2640 x 1795 source, this generates a 1322-pixel strip on the right:

```json
{
  "input": {
    "image": {"source": {"id": "<original-image-id>"}}
  },
  "size": {"width": 3962, "height": 1795},
  "placement": {
    "alignment": {"horizontal": "left", "vertical": "center"},
    "inset": {"left": 0, "top": 0, "right": 0, "bottom": 0}
  },
  "seeds": [333],
  "output": {"format": "png"}
}
```

## All placement options

- `size.width` and `size.height`: integers from 1 to 4000 pixels inclusive;
  each defaults to 2048.
- Horizontal alignment: `left`, `center` (default), or `right`. Left placement
  leaves additional space on the right; right placement leaves it on the left.
- Vertical alignment: `top`, `center` (default), or `bottom`. Top placement
  leaves additional space below; bottom placement leaves it above.
- Insets: `left`, `top`, `right`, and `bottom`, each a nonnegative integer
  defaulting to zero. These reserve minimum margins, not absolute crop coordinates.

The available rectangle is `width - left - right` by `height - top - bottom`.
Contain-fit preserves aspect ratio to integer-pixel precision. Center alignment
assigns an odd extra pixel to the right or bottom.

For a large square photograph, request a 2048 x 2048 canvas with each inset set
to 128 to downscale the source to 1792 x 1792 and generate a border on all sides.
Omitted inset edges default to zero.

Invalid insets are rejected: available width and height must each be at least
`min(canvas width, canvas height, 20)` pixels. Resizing must leave at least one
source pixel per dimension. If the source occupies the entire canvas, there
is nothing to expand; change the output size/aspect ratio or add insets.

## White pixels are not automatically editable

Existing white padding, snow, and white objects remain source content.
Placement mode does not detect white borders or infer an edit mask from
transparency. Remove existing padding before using placement mode, or supply
an explicit mask for that padding.

This capability changes input preparation, not the generation model.
It does not add prompt-guided Expand, reposition the original subjects,
guarantee exact pixel preservation, or reconstruct a particular off-camera scene.
Existing seed and output options retain their behavior; start with one seed.
Content Credentials retain the original source as provenance. The synthesized
mask is not a user-uploaded ingredient.

## Submit and poll

Save a request as `expand.json`:

```bash
curl --silent --show-error --fail-with-body \
  -X POST 'https://firefly-api.adobe.io/v4/images/expand' \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "x-api-key: $CLIENT_ID" \
  -H 'Content-Type: application/json' \
  --data-binary '@expand.json'
```

HTTP `202` means accepted. Poll the actual `links.result.url` from the response,
preserving its host and query parameters. Do not substitute `statusUrl` or
construct a polling URL. HTTP `200` can still mean `IN_PROGRESS`; completed
responses contain top-level `outputs`. Download each `image.presignedUrl`
promptly without attaching Adobe authentication to the storage request.
Respect `Retry-After`, use bounded polling, and handle non-2xx failures explicitly.

See the [API reference](../../api/index.md) for request schemas and the
[async guide](using-async-apis.md) for the v3/v4 polling differences.
