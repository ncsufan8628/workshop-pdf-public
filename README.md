# Workshop PDF Publisher

This repository distributes a prebuilt container that converts compatible workshop procedure files into an offline HTML navigator with linked, screen-sized PDF volumes. Source code is not included in this public release repository.

The publisher reads HTML, XML, WRL, and optional navigation-tree files; captures authored procedure steps with a locally supplied 3D runtime; preserves available tree headings and source order; and builds searchable PDF volumes with navigable procedure references.

## Important notice

This repository contains publishing software only. It does not include workshop manuals, service information, source procedure libraries, 3D runtime files, model assets, credentials, access tokens, or generated manuals.

Users must provide their own legally obtained source files and compatible runtime. Users are responsible for complying with applicable licences, copyright restrictions, access controls, and distribution requirements. Do not redistribute generated manuals unless you have permission to do so.

Do not upload source manuals, runtime files, credentials, generated PDFs, or publisher caches to this repository.

## Container image

```text
ghcr.io/ncsufan8628/workshop-pdf-public:latest
```

Pull the public image:

```bash
docker pull ghcr.io/ncsufan8628/workshop-pdf-public:latest
```

Available tags:

- `latest` and `html`: HTML navigator with segmented PDF volumes.
- `legacy`: unified single-PDF edition.
- `v0.1.9`: immutable tag for the current public release.

## Required input

A typical procedure directory contains files following this pattern:

```text
X0000004693.topic.html
TOPIC_0000004693.interactivity.xml
TOPIC_0000004693.wrl
```

The runtime directory must contain the JavaScript, WebAssembly, and supporting files required to display the associated WRL models. Optional navigation-tree XML files control the headings, nesting, ordering, and procedure placement used by the navigator and PDF tables of contents.

The publisher includes only the newest detected revision of each procedure.

## Sample configuration

Create a `.env` file beside `docker-compose.yml`:

```dotenv
WORKSHOP_SOURCE_DIR=/path/to/workshop-source
WORKSHOP_RUNTIME_DIR=/path/to/3d-runtime
WORKSHOP_TREE_DIR=/path/to/navigation-trees
PDF_OUTPUT_DIR=/path/to/generated-output
PDF_WORK_DIR=/path/to/publisher-cache

PDF_TITLE=Workshop Manual
PDF_FILENAME=workshop-manual.pdf

PDF_CONCURRENCY=0
PDF_SAMPLE_SIZE=0
PDF_MERGE_ONLY=0
PDF_ALLOW_PARTIAL=0
PDF_FORCE=0

PDF_GPU_BACKEND=auto
PDF_SHM_SIZE=16gb

PDF_CAPTURE_SCALE=0.75
PDF_IMAGE_MAX_DIMENSION=1400
PDF_IMAGE_JPEG_QUALITY=78

PDF_SEGMENT_DEPTH=3
PDF_SEGMENT_TARGET_MB=300

PDF_STATE_CHECKPOINT_INTERVAL=25
PDF_PROGRESS_INTERVAL=10
PDF_HEARTBEAT_SECONDS=60

PUID=1000
PGID=1000
```

Example `docker-compose.yml`:

```yaml
services:
  workshop-pdf:
    image: ghcr.io/ncsufan8628/workshop-pdf-public:latest
    container_name: workshop-pdf
    user: "${PUID:-1000}:${PGID:-1000}"
    network_mode: none
    init: true
    environment:
      PDF_TITLE: ${PDF_TITLE:-Workshop Manual}
      PDF_FILENAME: ${PDF_FILENAME:-workshop-manual.pdf}
      PDF_CONCURRENCY: ${PDF_CONCURRENCY:-0}
      PDF_SAMPLE_SIZE: ${PDF_SAMPLE_SIZE:-0}
      PDF_MERGE_ONLY: ${PDF_MERGE_ONLY:-0}
      PDF_ALLOW_PARTIAL: ${PDF_ALLOW_PARTIAL:-0}
      PDF_FORCE: ${PDF_FORCE:-0}
      PDF_GPU_BACKEND: ${PDF_GPU_BACKEND:-auto}
      PDF_CAPTURE_SCALE: ${PDF_CAPTURE_SCALE:-0.75}
      PDF_IMAGE_MAX_DIMENSION: ${PDF_IMAGE_MAX_DIMENSION:-1400}
      PDF_IMAGE_JPEG_QUALITY: ${PDF_IMAGE_JPEG_QUALITY:-78}
      PDF_SEGMENT_DEPTH: ${PDF_SEGMENT_DEPTH:-3}
      PDF_SEGMENT_TARGET_MB: ${PDF_SEGMENT_TARGET_MB:-300}
      PDF_STATE_CHECKPOINT_INTERVAL: ${PDF_STATE_CHECKPOINT_INTERVAL:-25}
      PDF_PROGRESS_INTERVAL: ${PDF_PROGRESS_INTERVAL:-10}
      PDF_HEARTBEAT_SECONDS: ${PDF_HEARTBEAT_SECONDS:-60}
    volumes:
      - "${WORKSHOP_SOURCE_DIR}:/source:ro"
      - "${WORKSHOP_RUNTIME_DIR}:/runtime:ro"
      - "${WORKSHOP_TREE_DIR}:/trees:ro"
      - "${PDF_OUTPUT_DIR}:/output:rw"
      - "${PDF_WORK_DIR}:/work:rw"
    devices:
      - /dev/dri:/dev/dri
    read_only: true
    tmpfs:
      - /tmp:size=4g,mode=1777
    shm_size: ${PDF_SHM_SIZE:-16gb}
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    restart: "no"
```

Remove the `devices` section on a host without `/dev/dri`. The publisher automatically uses software rendering when a compatible render device is unavailable.

Run the publisher:

```bash
docker compose run --rm workshop-pdf
```

## Configuration variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `WORKSHOP_SOURCE_DIR` | Required | Procedure source directory, mounted read-only. |
| `WORKSHOP_RUNTIME_DIR` | Required | Compatible 3D runtime directory, mounted read-only. |
| `WORKSHOP_TREE_DIR` | Optional | Navigation-tree XML directory, mounted read-only. |
| `PDF_OUTPUT_DIR` | Required | Generated navigator, PDF volumes, launchers, and manifests. |
| `PDF_WORK_DIR` | Required | Persistent procedure cache and resumable state. |
| `PDF_TITLE` | `Workshop Manual` | Generated page title and PDF metadata title. |
| `PDF_FILENAME` | `workshop-manual.pdf` | Filename used by the legacy single-PDF edition. |
| `PDF_CONCURRENCY` | `0` | Parallel workers. `0` uses all CPUs visible to the container. |
| `PDF_SAMPLE_SIZE` | `0` | Publishes only the first N procedures for testing. `0` publishes everything. |
| `PDF_MERGE_ONLY` | `0` | Rebuilds output from cached procedure PDFs without rendering again. |
| `PDF_ALLOW_PARTIAL` | `0` | Allows the legacy edition to finish when procedures fail. |
| `PDF_FORCE` | `0` | Ignores compatible cached renders and renders procedures again. |
| `PDF_GPU_BACKEND` | `auto` | Attempts hardware rendering and falls back to software rendering. |
| `PDF_SHM_SIZE` | `16gb` | Shared memory available to browser rendering processes. |
| `PDF_CAPTURE_SCALE` | `0.75` | Screenshot density. Higher values improve detail and increase size. |
| `PDF_IMAGE_MAX_DIMENSION` | `1400` | Maximum embedded screenshot width or height in pixels. |
| `PDF_IMAGE_JPEG_QUALITY` | `78` | JPEG quality used for embedded screenshots. |
| `PDF_SEGMENT_DEPTH` | `3` | Navigation-tree depth used for initial PDF volume groups. |
| `PDF_SEGMENT_TARGET_MB` | `300` | Approximate size at which a volume is divided into additional parts. |
| `PDF_STATE_CHECKPOINT_INTERVAL` | `25` | Completed procedures between resumable state writes. |
| `PDF_PROGRESS_INTERVAL` | `10` | Completed procedures between progress log messages. |
| `PDF_HEARTBEAT_SECONDS` | `60` | Interval for activity messages during long renders. |
| `PUID` | `1000` | Host user ID used for generated files. |
| `PGID` | `1000` | Host group ID used for generated files. |

## Generated output

```text
workshop-manual/
├── index.html
├── catalog.json
├── Start Workshop Manual.command
├── Start Workshop Manual.bat
├── START HERE - Local Viewing.txt
└── volumes/
    ├── 01-workshop-manual-....pdf
    └── ...
```

Use the included launcher when copying the bundle to another computer. It starts a loopback-only server and opens the main navigator at `http://127.0.0.1:8765/index.html`. PDF volumes remain supporting files opened through that navigator.

## Unraid

Create the container manually using the image and variables documented above. Use `none` for the network type, leave privileged mode disabled, mount `/source`, `/runtime`, and `/trees` read-only, and mount `/output` and `/work` read/write. The publisher is a one-shot job and stops after the output is complete.

