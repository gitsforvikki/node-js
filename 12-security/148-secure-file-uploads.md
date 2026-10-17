# Lesson 148 — Secure File Uploads in Node.js

## 1. Why are file uploads risky?

A profile-photo upload seems simple: the user chooses a file and your Express server saves it. But an attacker can submit a huge file, disguise its type, use a dangerous filename, or repeatedly upload files until storage is full.

**Security goal:** Accept only files your application actually needs, enforce limits, inspect content, store safely, and control who can access them.

## 2. How multipart uploads work

A browser commonly sends files as `multipart/form-data`, not JSON.

```text
Browser <input type="file">
   -> POST /api/avatar (multipart/form-data)
   -> authentication
   -> upload parser + size limit
   -> type and content checks
   -> safe storage
   -> database stores file ID/path
   -> response
```

`express.json()` does **not** parse multipart files. Middleware such as Multer handles multipart forms.

## 3. Main threats

| Threat | Example | Defense |
| --- | --- | --- |
| Oversized file | 500 MB avatar | Strict byte-size limit |
| Wrong file type | Executable renamed to .png | Inspect actual file content |
| Dangerous filename/path | Path traversal or confusing extension | Generate server-side names |
| Malware | Malicious document | Scan/quarantine where appropriate |
| Storage exhaustion | Thousands of uploads | Quotas, rate limits, cleanup |
| Public private files | Anyone guesses a document URL | Authorization before download |
| SVG/HTML active content | Browser executes active content | Disallow risky formats or serve safely |
| Image parser bugs | Crafted image exploits library | Updated image-processing libraries and resource limits |

## 4. Multer example: upload avatar to memory

Install:

```bash
npm install express multer file-type sharp
```

`file-type` is an ESM package. Multer handles the multipart body, while `file-type` checks known file signatures. Sharp decodes and rewrites accepted images.

```js
import express from "express";
import multer from "multer";
import { fileTypeFromBuffer } from "file-type";
import sharp from "sharp";
import { randomUUID } from "node:crypto";
import { mkdir, writeFile } from "node:fs/promises";
import path from "node:path";

const app = express();
const MAX_BYTES = 2 * 1024 * 1024; // 2 MB

const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: MAX_BYTES,
    files: 1,
    fields: 0,
    parts: 1,
  },
});

app.post("/api/avatar", requireAuth, upload.single("avatar"), async (req, res) => {
  if (!req.file) return res.status(400).json({ message: "Avatar required" });

  const detected = await fileTypeFromBuffer(req.file.buffer);
  if (!detected || !["image/png", "image/jpeg"].includes(detected.mime)) {
    return res.status(415).json({ message: "Only PNG/JPEG allowed" });
  }

  // Decode and re-encode to a known format. Sharp may reject invalid images.
  const output = await sharp(req.file.buffer, { limitInputPixels: 20_000_000 })
    .rotate()
    .resize(512, 512, { fit: "cover" })
    .jpeg({ quality: 80 })
    .toBuffer();

  const filename = randomUUID() + ".jpg"; // Never use client filename
  const directory = path.resolve("private-uploads");
  await mkdir(directory, { recursive: true });
  await writeFile(path.join(directory, filename), output, { flag: "wx" });

  // Store filename + owner ID in DB; implement authorization on download.
  res.status(201).json({ fileId: filename });
});
```

**Assumptions:** `requireAuth` is implemented, this route uses a **single Node.js instance** with a private local directory, and uploads are small. For multi-instance/cloud deployment, use private object storage rather than relying on local disk.

**Important:** `memoryStorage()` buffers the entire allowed file in RAM. Even a 2 MB limit can consume significant memory under many concurrent uploads. Add rate/concurrency limits and infrastructure request-size limits. For large files use streaming to controlled temporary storage and scan before promotion.

## 5. How this example works

1. `requireAuth` identifies the logged-in user.
2. Multer accepts one `avatar` field and enforces a byte limit.
3. `fileTypeFromBuffer` examines file signatures instead of trusting `file.mimetype` or the extension.
4. Sharp decodes and re-encodes the image, helping remove unwanted metadata and rejecting many invalid inputs.
5. The server generates a random filename, not one supplied by the browser.
6. Store file metadata with its owner; serve it only after authorization.

File signatures alone do not prove a file is harmless. Re-encoding is useful for images but does not replace all malware scanning or sandboxing requirements.

## 6. Handle Multer errors properly

```js
app.use((err, req, res, next) => {
  if (err instanceof multer.MulterError) {
    return res.status(400).json({
      message: err.code === "LIMIT_FILE_SIZE"
        ? "File too large"
        : "Invalid upload",
    });
  }
  next(err);
});

// Keep a final error handler after this one.
```

Do not expose internal filesystem paths or parser stack traces to clients.

## 7. Private file download with ownership check

Never expose sensitive files using `app.use("/uploads", express.static("private-uploads"))`.

Instead:

```js
app.get("/api/files/:id", requireAuth, async (req, res) => {
  const file = await File.findOne({
    publicId: req.params.id,
    ownerId: req.user.id,
  });
  if (!file) return res.sendStatus(404);

  // Resolve the path from trusted DB metadata, not arbitrary request path.
  res.type("application/octet-stream");
  res.set("Content-Disposition", 'attachment; filename="download.bin"');
  res.sendFile(file.storedName, { root: path.resolve("private-uploads") });
});
```

A real implementation should validate stored names, choose the correct download content type, and use a suitable file ID. For cloud storage, issue short-lived signed URLs **only after** authorization.

## 8. Browser upload example

```jsx
async function uploadAvatar(file) {
  const form = new FormData();
  form.append("avatar", file);

  const response = await fetch("/api/avatar", {
    method: "POST",
    body: form,
    credentials: "include",
  });

  if (!response.ok) throw new Error("Upload failed");
  return response.json();
}
```

Don't manually set `Content-Type: multipart/form-data`; the browser must add the boundary. Cookie-authenticated uploads need CSRF protection where applicable.

## 9. Additional production controls

- **Storage:** private bucket, server-generated object keys, no execute permission.
- **Authorization:** check owner/role before upload and download.
- **Quotas:** limits per user, request, and time window.
- **Scanning:** quarantine and scan risky file formats before use.
- **Content-Disposition:** use `attachment` for potentially active content.
- **Cleanup:** remove abandoned temporary files and orphaned objects.
- **Monitoring:** log file ID, size, and result without logging private content.
- **Dependency updates:** patch Multer, Sharp, and parsers.

## 10. Common mistakes and practice

Mistakes: trusting `file.originalname`, checking only extensions, using a public uploads directory for private files, unbounded memory uploads, and returning raw filesystem errors.

Practice: (1) Upload a valid small JPEG. (2) Rename a text file to `.jpg` and confirm rejection. (3) Upload over 2 MB and check error handling. (4) Try downloading another user's file and confirm denial. (5) Explain why object storage is useful with multiple servers.

**Interview answer:** Secure uploads require authentication, strict size/count/type validation, server-generated filenames, safe storage, scanning/processing where appropriate, quotas, and authorization for retrieval. Trust neither filenames nor client-supplied MIME types.
