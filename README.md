# A³ Attache for YouTrack
Secure S3‑compatible storage integration for YouTrack Server (Amazon S3, MinIO, etc.)

**Designed for YouTrack Server** 


## Features

- **Secure S3‑compatible storage** for YouTrack attachments (Amazon S3, MinIO, and compatible services).
- **AWS Signature V4** presigned URLs with native runtime-compatible cryptographic implementation.
- **MinIO optimized** with proper path‑style addressing and query parameter handling.
- **Built-in file preview** for XLSX, DOCX, and ODT files with:
  - XLSX: up to 1000 rows × 50 columns with truncation warnings
  - DOCX/ODT: 30-second rendering timeout with size warnings for large files (>5MB)
  - Selectable preview engine (built‑in or native browser) and open mode (tab or popup)
  - Unified header with file metadata (size, date) auto-updated from response headers
- **Grid/List views** with automatic switching, lazy-loaded image thumbnails, and responsive design.
- **Clock skew auto-detection** and compensation for presigned URL signature issues.
- **Admin healthcheck** with real presigned operations (list/put/get/delete) for configuration validation.
- **Safe admin API**: credentials never returned; only configuration flags are exposed.
- **Modular backend** architecture with separated concerns (crypto, handlers, utilities, diagnostics).
- more features coming soon...


## Goal & Explanation
Provide an external S3‑compatible storage for YouTrack attachments: upload large files, browse and preview inline, and keep files separate from YouTrack data. The app uses only the S3 endpoint you configure in Admin Settings; no other external services are required.

### For System Administrators
- **Simplified Backups** — Files stored externally in your S3 bucket, separate from YouTrack database. Back up file attachments independently using your S3 provider's backup tools.
- **No File Size Limits** — Store files of any size without database constraints. Upload multi-gigabyte files directly to S3 without impacting YouTrack performance.
- **Reduced Database Size** — Keep your YouTrack database lean and fast by offloading file storage to dedicated S3-compatible infrastructure.
- **Scalable Storage** — Use existing S3/MinIO infrastructure with unlimited scaling capacity. Pay only for actual storage used.

## Architecture

A³ Attache consists of three integrated components:

- **File Browser Widget** — Displays in issue footer; grid/list view with thumbnails, upload/download/delete operations
- **Admin Configuration Panel** — Global settings page for S3 credentials, TTL, preview engine, and system diagnostics
- **File Previewer** — Standalone preview window for Office documents, images, PDFs, any text files with built-in or native browser rendering

## Getting Started

### 1. Install the app
Install A³ Attache from YouTrack Marketplace or upload manually to your YouTrack Server instance.

### 2. Obtain S3 credentials
Get Access Key ID and Secret Access Key from your S3 provider:

- **Amazon S3**: [AWS IAM Access Keys Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- **MinIO**: [MinIO User Management](https://min.io/docs/minio/linux/administration/identity-access-management/minio-user-management.html)
- **Other S3-compatible**: Check your provider's documentation for API credentials (Wasabi, DigitalOcean Spaces, Backblaze B2, etc.)

### 3. Configure the app
Go to YouTrack Admin → Applications → A3 Attache and configure:

- **S3 endpoint URL**: e.g., `https://s3.amazonaws.com` or `http://localhost:9000`
- **Region**: e.g., `us-east-1` (required even for MinIO)
- **Bucket name**: Your S3 bucket name
- **Base directory**: Optional prefix for all files (e.g., `youtrack-attachments/`)
- **Access Key ID and Secret Access Key**: From step 2
- **Path-style addressing**: Enable for MinIO and some S3-compatible services
- **Preview engine**: Choose built-in (with CDN) or native browser
- **TTL settings**: Configure presigned URL expiration times

### 4. Test configuration
Click **"Run S3 Healthcheck"** button to verify all operations (list, upload, download, delete) work correctly.

### 5. Start using
Navigate to any issue and use the A³ Attache widget in the issue footer to upload and manage files.

## Settings

- **s3Endpoint** — S3 server endpoint URL (e.g., `https://s3.amazonaws.com`, `http://localhost:9001`).
- **s3Region** — S3 region. Default: `us-east-1`.
- **s3Bucket** — Bucket name for file storage (no leading/trailing slashes).
- **s3BaseDir** — Optional base directory inside the bucket (normalized automatically).
- **s3AccessKeyId** — Access key ID used for presigning.
- **s3AccessKeySecret** — Secret access key used for presigning (never logged or returned by API).
- **s3ForcePathStyle** — Use path‑style URLs (`/bucket/key`). Required for MinIO. Default: `true`.
- **s3Provider** — Provider preset adjusting presign nuances (list params, content‑sha256). Default: `minio_default`.
- **clockSkewAdjustSec** — Offset (sec) to compensate clock skew; can be negative. Default: `0`.
- **presignNoCache** — Add cache‑busting to presign requests, disable intermediate caching. Default: `true`.
- **logLevel** — Logging mode: `no_log` | `minimal` | `debug`. Default: `minimal`.
- **listTtlSec** — TTL (sec) for presigned List URLs. Default: `120`.
- **uploadTtlSec** — TTL (sec) for presigned Upload (PUT) URLs. Default: `900`.
- **downloadTtlSec** — TTL (sec) for presigned Download (GET) URLs. Default: `600`.
- **deleteTtlSec** — TTL (sec) for presigned Delete URLs. Default: `300`.
- **previewEngine** — File preview engine: `built_in` | `native`. Default: `native`.
- **previewOpenMode** — Where to open preview: `tab` | `popup`. Default: `popup`.
- **filesListView** — Default list view in widget: `grid` | `list`. Default: `grid`.
- **filesListViewSwitchThreshold** — Auto‑switch/list behavior threshold. Default: `20`.

## Usage
- Open an issue to use the attachments widget.
- **Attach files**: Drag & drop or click "Attach" button.
- **Browse files**: Switch between Grid and List views; sort by date, type, or name.
- **Preview files**: Double-click to open preview (supports images, PDF, text, XLSX, DOCX, ODT).
  - Built-in preview loads viewer libraries from CDN and renders in a popup/tab window.
  - Preview limits: XLSX (1000×50), DOCX/ODT (30s timeout, 5MB warning).
  - Use "Download" button for full file access if preview is truncated or slow.
- **Download/Delete**: Hover over file for action buttons.
---

### Notes

* Settings are managed via the YouTrack App Settings and stored in YouTrack storage in production. 
* For MinIO you typically need `s3ForcePathStyle=true` and a valid `region` (commonly `us-east-1`).
* Ensure required permissions are configured on your S3 provider side according to your organization’s policy. Configure only the settings visible in the app Admin page.
* S3 Healthcheck is triggered from the Admin page. It performs real presigned operations server‑side; browser CORS does not apply.
* **Privacy**: This app does not collect, transmit, or store any telemetry, analytics, or personal data. All processing occurs locally within your YouTrack instance and your configured S3 storage. External services contacted: (1) your configured S3 endpoint, (2) public CDN for preview libraries (docx-preview, SheetJS, WebODF) only when using built-in preview feature. To avoid any CDN requests, set Preview Engine to `native` in Admin Settings (default).

#### Runtime preview libraries (CDN)

- Built‑in preview dynamically loads viewer libraries from CDN and they are not bundled with the app:
  - `docx-preview` (MIT)
  - `SheetJS/xlsx` (Apache‑2.0)
  - `WebODF` (AGPL‑3.0)
  See `THIRD_PARTY_NOTICES.md` → Runtime CDN Dependencies.

### Troubleshooting

* **"S3 configuration is incomplete"** — fill all required fields in Admin Settings and click "Save", then run the Healthcheck.
* **403 errors**: verify bucket policy and credentials, check region/endpoint, and ensure MinIO path‑style is enabled when required.
  - The app includes automatic clock skew detection and retry logic for expired signatures.
* **Time skew**: if you see RequestTimeTooSkewed, the app will auto-detect and compensate. You can also manually set `clockSkewAdjustSec` in settings.
* **Preview issues**:
  - "Rendering timeout" for DOCX/ODT: file is too large (>5MB recommended limit). Use Download button.
  - "Showing X of Y rows" for XLSX: spreadsheet exceeds 1000×50 limit. Use Download for full file.
  - Preview libraries fail to load: check browser console for CDN blocking (CSP/firewall). Switch to `native` preview engine in settings.
* **CORS Configuration**: YouTrack plugins run in iframe and send `Origin: null` header. S3/MinIO also requires CORS for browser file access. See complete setup guide: [SETUP_CORS.md](./SETUP_CORS.md).

Preview metadata (size/date) priority used by the app:

- Parameters passed from the file list (sizeBytes/lastModified) — highest priority.
- Response headers (Content-Length/Last-Modified) — used to fill missing fields when available via CORS ExposeHeaders.
- As a fallback, size is derived from the downloaded payload (ArrayBuffer/Blob) when headers/parameters are unavailable.

---

## Support

For questions and support, contact:

- Email: apps@mag1.cc

## License

This is proprietary software. See `LICENSE` and `EULA.md`.

- Distribution via JetBrains Marketplace — subscription/activation, entitlement scope (seats/instance) and billing are governed by JetBrains Marketplace. Our EULA also applies: `EULA.md`.
- Internal use by the licensor’s employer — allowed free of charge for the duration of employment and ends automatically upon its termination (see the simplified clause in `EULA.md`, details in `LICENSE`).
- This project is not affiliated with JetBrains. YouTrack is a trademark of JetBrains s.r.o. See `NOTICE`.
- Third‑party open‑source components are used under their respective licenses. See `THIRD_PARTY_NOTICES.md`.
