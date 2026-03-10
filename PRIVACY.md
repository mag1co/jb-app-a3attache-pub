# Privacy Policy for A3 Attache

**Last Updated**: October 21, 2025

## Data Collection

**A3 Attache does not collect, transmit, or store any telemetry, analytics, or personal data.**

All processing occurs locally within your YouTrack instance and your configured S3-compatible storage.

## External Services

The plugin contacts only the following external services:

1. **Your configured S3 endpoint** — for file storage operations (upload, download, delete)
2. **Public CDN servers** — only when using the built-in preview feature, to load document viewer libraries (docx-preview, SheetJS, WebODF). No personal data is transmitted to these CDN servers.

To avoid any CDN requests, set Preview Engine to "native" in Admin Settings (this is the default).

## Data Processing

- All file operations use AWS Signature V4 presigned URLs generated on your YouTrack server
- S3 credentials are stored securely in YouTrack App Settings
- No credentials or personal data are logged or transmitted to third parties

## Contact

For privacy-related questions, contact: apps@mag1.cc
