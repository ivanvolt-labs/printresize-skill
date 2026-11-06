---
name: printresize
description: Turn an image into a print file with the PrintResize API. Use when the user wants to resize an image to an exact print size in centimetres, upscale to 2K, 4K, or 8K, make a pattern repeat seamlessly, create a tablecloth or placemat mockup, swap the scene of a finished cloth or mat, or extract a flat pattern from a product photo. Runs spend credits from the user's PrintResize account.
---

# PrintResize

Every tool works the same way: submit a form request, poll the task, download the file. Runs spend real credits, so check the price before you submit.

## Setup

The API key goes in `PRINTRESIZE_API_KEY`. Keys start with `pr_` and are created at https://printresize.com/api-keys.

If the variable isn't set, ask the user to create a key and set it. Don't ask them to paste it into chat, and never print it, write it to a file, or put it in a URL.

```bash
BASE=https://printresize.com
AUTH="Authorization: Bearer $PRINTRESIZE_API_KEY"
```

## Workflow

1. **Pick the tool and values** from the table below. Ask the user about anything that changes the output and has no default. Don't guess a print size.
2. **Check the price.** Send the full request with `-F dryRun=true`. You get `{"success":true,"dryRun":true,"credits":6,"balance":150}`. Nothing is charged and no task is created.
3. **Confirm the cost** with the user, unless they already approved a limit that covers it. If `credits` is more than `balance`, stop and tell them to top up.
4. **Submit** the same request without `dryRun`, adding `clientRequestId`. You get `{"success":true,"taskId":"...","credits":6}`.
5. **Poll** `GET /api/ai-tasks/{taskId}` every 5 seconds until `status` is `completed` or `failed`. Runs usually take 1 to 3 minutes, and extra-large resizes up to 5.
6. **Download** from `GET /api/ai-creations/{taskId}/download` only after `completed`. Report the saved path, the size in pixels, the credits spent, and the `dpi` when there is one.

```bash
# Price check
curl -sS -X POST "$BASE/api/print-resize" -H "$AUTH" \
  -F image=@swatch.png -F widthCm=120 -F heightCm=40 -F dryRun=true

# Submit
curl -sS -X POST "$BASE/api/print-resize" -H "$AUTH" \
  -F image=@swatch.png -F widthCm=120 -F heightCm=40 \
  -F clientRequestId=order-1042-swatch-1

# Poll, then download
curl -sS -H "$AUTH" "$BASE/api/ai-tasks/$TASK_ID"
curl -sS -H "$AUTH" -o result.png "$BASE/api/ai-creations/$TASK_ID/download"
```

Balance on its own: `GET /api/credits` returns `{"balance":150}`.

## Request rules

- Send form fields (`-F`). A JSON body returns `400 image_required`.
- Send the image as `image=@file` for a local file, or `imageUrl=https://...` for a public URL. The URL must return an image and can't point to a private address.
- Use the real image in the price check, because the Print Resize price depends on the source size.
- `clientRequestId` is a unique string you choose. If a submit times out, resend the same request with the same id: you get the original task back with no second charge. An id is unique across the account and never expires, so a reused id returns the old task even if the parameters differ, and a new attempt after a failed task needs a new id.

## Tools

All are `POST`. `quality` is `standard` or `enhanced`. `enhanced` needs an active Studio or Production subscription and otherwise returns `403 enhanced_requires_subscription`.

| Goal | Endpoint | Required | Optional |
|---|---|---|---|
| Exact print size | `/api/print-resize` | image; `widthCm`, `heightCm` (10 to 240, not needed for oval) | `shape` (`rect` default, `square`, `round`, `oval`), `maskId` (required for oval), `quality` (`standard` default), `padEdge` (`solid` default, `extend`), `prompt` (500 chars) |
| Upscale | `/api/upscale` | image (10 MB, 4096 px long side); `mode` (`faithful`, `restore`; `restore` stops at 4K); `resolution` (`2k`, `4k`, `8k`) | |
| Seamless repeat | `/api/seamless-pattern` | image (10 MB, 4096 px long side, long side at most 6x short side); `join` (`both`, `horizontal`, `vertical`) | `seamWidth` (`medium` default, `narrow`, `wide`) |
| Tablecloth mockup | `/api/industry/tablecloth-mockup` | image; `shape` (`rect`, `square`, `round`); `drape` (`half`, `full`, `custom`); `edge` (`straight`, `wave`); `ratio`; `resolution`; `quality` | `drapeDistanceCm` (1 to 200, required with `custom`), `styleId`, `extraPrompt` (400 chars) |
| Placemat mockup | `/api/industry/placemat-mockup` | image; `shape` (`rect`, `square`, `round`, `oval`, `coffee`, `nightstand`, `tv`, `sideboard`, `student`, `irregular`); `ratio`; `resolution`; `quality` | `styleId`, `extraPrompt` (400 chars) |
| Scene swap | `/api/industry/tablecloth-scene-swap`, `/api/industry/placemat-scene-swap` | image; `mode` (`scene`, `object`); `ratio`; `resolution`; `quality`; in scene mode `envId` or `sceneDesc`; in object mode `replaceDesc` | `objectImage` or `objectImageUrl` (object mode) |
| Extract flat pattern | `/api/industry/extract-pattern`, `/api/industry/placemat-extract-pattern` | image (10 MB, 4096 px long side); `shapeId`; `patternId`; `ratio` (`21:9`, `2:3`, `3:4`, `4:5`, `9:16`, `1:1`); `resolution`; `quality` | `extraPrompt` (2000 chars) |

Mockups and scene swaps take `ratio` from `1:1`, `3:2`, `2:3`, `4:3`, `3:4`, `4:5`, `5:4`, `16:9`, `21:9`, `9:16`. Home textile `resolution` is `1k`, `2k`, or `4k`.

Look up long value lists when you need them:

- Oval `maskId` values (19): https://printresize.com/docs/api/print-resize
- `styleId`, `envId`, `shapeId`, and `patternId` values: https://printresize.com/docs/api/home-textiles

## What the tools can't do

- Print Resize sizes run from 10 to 240 cm per side. Oval placemats come in 19 fixed sizes. A source too small for the target returns `400 too_far`.
- Uploads to Print Resize are limited to 20 MB. The upscale, seamless, and extract tools take 10 MB and 4096 px on the long side.
- Seamless Pattern returns a repeat tile of about 1024 px on the short side, at most 1536 px on the long side. It isn't a print file: send the result to Print Resize or Upscale to reach print size.
- Only Print Resize sets `dpi`, after the task completes; read it before calling a file print-ready. Other tools return `dpi: null` because they don't set a print size.

## Handling errors

Branch on the HTTP status, then the code. Tool endpoints return `{"success":false,"errorCode":"..."}`, polling returns `{"errorCode":"..."}`, and the download endpoint returns `{"error":"..."}`.

| Response | What to do |
|---|---|
| `400` | Fix the request. The same request fails the same way. `invalid_params` and `invalid_preset` mean a wrong or missing value. |
| `401 unauthorized` | Don't retry. The key is missing or wrong. |
| `402 insufficient_credits` | Stop and tell the user to top up. |
| `403` | `enhanced_requires_subscription`: ask before retrying with `quality=standard`. `account_banned`: stop. |
| `404` on download | The task has no result yet. Keep polling until `completed`. |
| `413 image_too_large` | Shrink the file, then retry. |
| `429 concurrency_limit` | Wait for a running task to finish, then retry. There is no `Retry-After` header. |
| `500` | Wait briefly, then retry with the same `clientRequestId`. |
| `502 image_fetch_failed` | Check that the URL is public and returns an image. |
| Task `status: failed` | Credits come back on their own (`refunded: true`). Report `errorCode`. Retry with a new `clientRequestId`, but ask first: the retry spends credits again. |
| Task `errorCode` starting with `content_policy`, or `text_moderation_blocked`, `local_policy_violation_blocked` | Don't retry unchanged. Ask the user to change the image or prompt. |

Treat any code you don't recognise as a generic failure.

## Spending credits

- Never submit a real run without a price check.
- For a batch, price each item, then confirm the total before the first submit. Run them within the plan's task limit.
- Don't switch tools, raise `quality`, or raise `resolution` to get a better result without asking. Each of these can change the cost.

Full reference: https://printresize.com/docs
