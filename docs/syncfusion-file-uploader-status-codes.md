# Syncfusion File Uploader — All Status Codes

Complete reference for every `statusCode` value handled by the Syncfusion EJ2 File
Uploader (`@syncfusion/ej2-inputs` → `Uploader` component).

Source of truth: `controls/inputs/src/uploader/uploader.ts` in
`syncfusion/ej2-javascript-ui-controls` (verified against the shipped
`@syncfusion/ej2-inputs` build, v35.1.41).

---

## 1. `FileInfo.statusCode` — the file's lifecycle state

`statusCode` is a **string** (not a number) on every `FileInfo` object. It describes the
current state of a file in the uploader's list.

| Code | State | `status` text (default en-US) | Where it is set | UI state |
|---|---|---|---|---|
| `'0'` | **Failed / Invalid** | `File failed to upload`, `File Name is not allowed`, `File type is not allowed`, `File size is too small`, `File size is too large`, `Unable to remove file` | `uploadFailed()`, `removeFailed()`, `chunkUploadFailed()`, validation failures (min/max size, extension, file name) | `e-upload-fails` / `e-validation-fails` — retry button shown |
| `'1'` | **Ready to upload (Selected)** | `Ready to upload` | `updateInitialFileDetails()`, `mergeFileInfo()`, `reloadcanceledFile()`, `retryUpload()`, retry-icon click | Neutral — remove icon |
| `'2'` | **Uploaded successfully** | `File uploaded successfully` | `raiseSuccessEvent()`, `removeCompleted()`, preloaded files | `e-upload-success` — delete icon |
| `'3'` | **Uploading (in progress)** | `Uploading` | `uploadInProgress()`, `chunkUploadInProgress()`, `resumeUpload()`, last chunk received | `e-upload-progress` — abort icon + progress bar |
| `'4'` | **Paused** | `File upload paused` / `Pause` | `pauseUpload()`, pause-icon click | `e-file-play-btn` (resume) icon |
| `'5'` | **Canceled** | `File upload canceled` | `cancelUpload()`, `eventCancelByArgs()`, `removeFiles()` (abort icon), `removecanceledFile()` | `e-upload-fails` — retry button shown |

### State transitions

```
                    ┌──────────────────────────► '2' (Uploaded)
                    │                              │
'select' ─► '1' ────┼──► '3' (Uploading) ─────┬──► '0' (Failed)
  Ready              │      │                 │
                     │      ├──► '4' (Paused) ─┘──► '3' (resume)
                     │      │
                     │      └──► '5' (Canceled)
                     │
                     └──► '0' (validation failed: bad name/type/size)
```

Key rules from the source:

- **Retry** (`retryFailedFiles`) only acts on files whose code is `'5'` or `'0'`.
- **Pause** (`pauseUploading`) only acts on `'3'`; **Resume** (`resumeFiles`) only on `'4'`;
  **Cancel** (`cancelUpload`) only on `'3'`.
- **Upload** (`upload`) only picks up files whose code is `'1'`.
- On list render, `'2'` gets the success class; anything that is **not** `'1'` and **not** `'2'`
  gets the validation-fail class.
- `updateStatus(files, status, statusCode)` is the single funnel that writes both `status`
  and `statusCode`; it ignores empty/`null`/`undefined` values for either argument.

---

## 2. `ResponseEventArgs.statusCode` — the HTTP response code

A **separate, unrelated** `statusCode` exists on the response object exposed in the
`success`, `failure`, `canceling`, and `chunkSuccess` event args. This one is the raw HTTP
status number from the XHR.

Built by `getResponse(e)`:

```ts
{
    readyState:    target.readyState,      // e.g. 4
    statusCode:    target.status,          // e.g. 200, 404, 500
    statusText:    target.statusText,      // e.g. "OK"
    headers:       target.getAllResponseHeaders(),
    withCredentials: target.withCredentials
}
```

Note the differing types — `ResponseEventArgs.statusCode` is typed `object`, while
`FileInfo.statusCode` is typed `string`.

### HTTP handling rules

The uploader treats a request as successful only in the 2xx range:

| Condition | Source | Result |
|---|---|---|
| `readyState === 4 && status >= 200 && status <= 299` | `uploadComplete()` | `success` event, file → `'2'` |
| `readyState === 4 && status >= 200 && status <= 299` | `removeCompleted()` | `success` event, file → `'2'` |
| `readyState === 4 && response.status >= 200 && response.status < 300` | `chunkUploadComplete()` | `chunkSuccess` event |
| anything else | `uploadFailed()` / `removeFailed()` / `chunkUploadFailed()` | `failure` event, file → `'0'` |

So **HTTP 200–299 = success**, and every other status (including 3xx redirects not followed
by the browser, 4xx, 5xx) routes to the failure path and sets `FileInfo.statusCode = '0'`.

---

## 3. `status` vs `statusCode` vs `response.statusCode`

| Property | Type | Meaning |
|---|---|---|
| `file.status` | `string` | Human-readable, localized message shown in the file list |
| `file.statusCode` | `string` | Uploader-internal state machine (`'0'`–`'5'`) |
| `eventArgs.response.statusCode` | `object` / number | HTTP status returned by the server |

---

## 4. Practical usage

### Inspect a file's state

```js
uploader.getFilesData().forEach((file) => {
    switch (file.statusCode) {
        case '0': console.log('failed/invalid:', file.status); break;
        case '1': console.log('ready'); break;
        case '2': console.log('uploaded'); break;
        case '3': console.log('uploading'); break;
        case '4': console.log('paused'); break;
        case '5': console.log('canceled'); break;
    }
});
```

### React to a server error

```js
uploader.on('failure', (e) => {
    const httpStatus = e.response && e.response.statusCode; // 404, 500, ...
    console.error(`Upload failed with HTTP ${httpStatus}: ${e.file.status}`);
});
```

### Retry only the failed files

```js
uploader.retry(uploader.getFilesData().filter((f) => f.statusCode === '0'));
```

---

## 5. Reference

- Component: `Uploader` — `@syncfusion/ej2-inputs` (`src/uploader/uploader.ts`)
- Interfaces: `FileInfo` (`statusCode: string`), `ResponseEventArgs` (`statusCode?: object`)
- Default locale strings: `src/uploader/uploader.ts` → `preRender()` → `this.localeText`
- CSS classes: `e-upload-success`, `e-upload-fails`, `e-upload-progress`,
  `e-validation-fails`, `e-file-pause-btn`, `e-file-play-btn`, `e-file-reload-btn`
