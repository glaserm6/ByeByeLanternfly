# Bye-Bye Lanternfly — Render deployment copy

A small Flask website that uses the trained YOLOv8 model to draw boxes around visible **adult spotted lanternflies** in one uploaded photograph and report the number of retained detections. A photo count is not an estimate of an area's total population or infestation level.

## What is included

- `app.py`: Flask upload/prediction routes, validation, model startup, and annotated response.
- `templates/index.html`: mobile-friendly page, camera/photo picker, preview, result and messages (CSS and JavaScript are inline).
- `models/best.pt`: the existing trained detector checkpoint copied without modification.
- `requirements.txt`, `.python-version`, `.gitignore`: runtime and repository setup.
- `reports/`: the measured exploratory evaluation summary and per-image counts.
- `docs/roboflow_sources.md`: the dataset source, license and screening notes.

The application does not import training code or use a dataset at runtime. Uploads are read and processed in memory; the app does not save submitted photos. No API key, database, or secret environment variable is required. There is no external service for inference.

## Model and limitations

The model is `models/best.pt` (6,457,962 bytes, about 6.16 MiB). It is the existing best checkpoint from the completed five-epoch CPU fine-tune; this deployment copy contains the same file (SHA-256 `34a44eba7cdb1d115c86420e03826a430937ae4b6c3c73e0872ed85f14fbb46e`). It detects the adult spotted lanternfly class only. It can miss insects, especially small or overlapping ones, and can produce false detections on nymphs or other objects.

Exploratory evaluation at 640 px used confidence 0.20, chosen on validation data: on the source test split, test-set box-match precision/recall were 0.741/0.450 and count MAE was 1.728 insects per image; mAP@0.50 was 0.538 and mAP@0.50:0.95 was 0.294. The test set has 276 images and 765 adult annotations, but its previous use is unknown; its results were consulted when comparing checkpoints. Labels were machine-screened, not individually verified, and no verified empty-scene images were available. These numbers are exploratory, not an independent final estimate. Dense examples are undercounted (106 annotated adults yielded 10 detections). A single image must not be treated as an area-level population or infestation estimate.

## Run locally on Windows PowerShell

Use 64-bit Python 3.13.7 (the `.python-version` file documents the Render/local target):

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
$env:LANTERNFLY_WEIGHTS = "models\best.pt"
$env:LANTERNFLY_CONFIDENCE = "0.20"
$env:PORT = "5000"
python app.py
```

Open `http://127.0.0.1:5000`. `app.py` uses Flask's development server only for local testing. Do not use it as the Render production server.

Optional environment variables:

- `LANTERNFLY_WEIGHTS`: model path. Defaults to `models/best.pt`; a relative value is resolved from the project folder.
- `LANTERNFLY_CONFIDENCE`: detector confidence, default `0.20` (selected using validation data).
- `PORT`: local development server port; Render supplies this automatically to the production server.

## Upload this copy to GitHub

1. Extract `Bye-Bye-Lanternfly-Render.zip` and open the extracted `Bye-Bye-Lanternfly-Render` folder.
2. Create a new, empty GitHub repository yourself.
3. Upload the **contents** of the extracted folder while preserving its directories, especially `templates/`, `models/`, `docs/`, and `reports/`. Do not upload only the README or omit `models/best.pt`.
4. Confirm the repository contains `app.py`, `requirements.txt`, `.python-version`, `templates/index.html`, and `models/best.pt` at its root-relative paths.

The checkpoint is 6.46 MB, below GitHub's 25 MB browser-upload limit and 100 MB Git hard limit, so it can be added directly. It is not omitted or replaced with a different model. The original project remains separate and untouched.

## Configure a Render Web Service manually

No `render.yaml` is needed. In Render, create a **Web Service**, connect your new GitHub repository, select the branch, and leave Root Directory blank (or set it to the repository root). Use:

- **Runtime:** Python 3
- **Build command:** `pip install -r requirements.txt`
- **Start command:** `OMP_NUM_THREADS=1 MKL_NUM_THREADS=1 gunicorn --workers 1 --timeout 120 --bind 0.0.0.0:$PORT app:app`
- **Environment variables:** none required. Optionally set `LANTERNFLY_CONFIDENCE=0.20` and `LANTERNFLY_WEIGHTS=models/best.pt`.

The `app:app` target matches the Flask object in `app.py`. One Gunicorn worker avoids loading a separate model copy per worker and uses Render's supplied port. The 120-second timeout allows CPU inference on larger accepted photos. No persistent disk is needed because uploads are not retained.

### Compute and availability constraints

The app and model loaded locally with about 239 MiB process RSS; after an inference it was about 370 MiB on this Windows host. This is not a Linux/Render measurement, but indicates limited headroom on Render's Free plan (0.1 CPU, 512 MB RAM). PyTorch/native memory and larger uploaded photos can increase the peak, so Free deployment may be killed for memory or feel slow. For a classroom demo with real uploads, a service with 2 GB RAM and at least 1 CPU is the safer target if available to you; no service or paid plan has been created or purchased here. Free services can spin down after 15 minutes idle and take about a minute to wake. Their filesystem is ephemeral, which is acceptable for this stateless application.

The uploaded-file limit is 12 MB; accepted content must be JPEG, PNG, or WebP, and images above 30 megapixels are rejected. One synchronous worker handles a request at a time. The public service has no authentication or rate limiting, so share the URL for a small demonstration rather than an unrestricted high-volume service.

## Verify after deployment

1. Wait for Render's first build/deploy to finish and open the provided HTTPS `onrender.com` URL.
2. On a phone, use **Take a photo** or **Choose a photo**; on a computer, choose a JPEG, PNG, or WebP image.
3. Confirm the preview appears, tap **Analyze image**, then check the annotated image and count. The shown count is the number of returned target-class boxes.
4. Try a photo with no visible adults, and an invalid file or a file over 12 MB. A zero is a successful detection response; invalid uploads and inference failures should show errors.
5. Check that the page fits the narrow screen without horizontal scrolling. Do not interpret one image as the population of a site.

## Deployment verification performed before handoff

The copied model hash/size and required files were checked. Local validation uses the existing project unit tests plus a Flask test-client smoke test with the real checkpoint: model loads, a labeled photo produces an annotated response with `count == len(detections)`, a synthetic blank image returns a successful zero, and an invalid upload returns HTTP 400. The local app imports and starts. Browser/device testing and Linux Render deployment cannot be completed from this preparation copy; verify them after you connect the repository. The full model's Linux load and the target Render plan's actual peak memory therefore remain unverified.

For Render's current Flask deployment steps and resource limits, see [Render's Flask guide](https://render.com/docs/deploy-flask), [Python version configuration](https://render.com/docs/python-version), [compute plans](https://render.com/docs/compute-plans), and [free service limitations](https://render.com/docs/free). The CPU PyTorch and torchvision versions are pinned to wheels published for Python 3.13 Linux x86_64 at the [official PyTorch CPU wheel index](https://download.pytorch.org/whl/cpu/torch/) and [torchvision CPU wheel index](https://download.pytorch.org/whl/cpu/torchvision/).
