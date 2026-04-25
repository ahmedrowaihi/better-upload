# Better Upload — resumable fork

Adds resumable multipart uploads on top of `@better-upload/{client,server}`.

PR: [Nic13Gamer/better-upload#126](https://github.com/Nic13Gamer/better-upload/pull/126) · not merging upstream. Use the prebuilt artifacts from [pkg.pr.new](https://pkg.pr.new):

```bash
npm i https://pkg.pr.new/@better-upload/client@126
npm i https://pkg.pr.new/@better-upload/server@126
```

## What's new

- Client: `getResumeState?: (file) => { uploadId, key } | undefined` on all four entry points. `uploadId` exposed on `FileUploadInfo`. Progress counts parts the server already has.
- Server: route handler accepts `resume: { uploadId, key }`, calls `ListParts`, signs only missing parts. Falls back to fresh upload on `NoSuchUpload`. `file.resume` surfaced in `onBeforeUpload`. New `listParts` helper in `@better-upload/server/helpers`.

## Use it

Identify the file with a content-derived fingerprint, not just `name + size` (renames and re-saves collide). A SHA-256 over `[size][first 64KB][last 64KB]` is sub-millisecond even for huge files:

```ts
const sigCache = new WeakMap<File, Promise<string>>();

export function fileSig(file: File): Promise<string> {
  let p = sigCache.get(file);
  if (!p) p = (async () => {
    const SAMPLE = 64 * 1024;
    const headEnd = Math.min(SAMPLE, file.size);
    const tailStart = Math.max(headEnd, file.size - SAMPLE);
    const head = await file.slice(0, headEnd).arrayBuffer();
    const tail = tailStart < file.size ? await file.slice(tailStart).arrayBuffer() : new ArrayBuffer(0);
    const size = new Uint8Array(8);
    new DataView(size.buffer).setBigUint64(0, BigInt(file.size));
    const buf = new Uint8Array(8 + head.byteLength + tail.byteLength);
    buf.set(size, 0); buf.set(new Uint8Array(head), 8); buf.set(new Uint8Array(tail), 8 + head.byteLength);
    const digest = await crypto.subtle.digest('SHA-256', buf);
    return Array.from(new Uint8Array(digest)).map(b => b.toString(16).padStart(2, '0')).join('').slice(0, 32);
  })();
  sigCache.set(file, p);
  return p;
}
```

Wire the lifecycle:

```tsx
useUploadFile({
  route: 'videos',
  getResumeState: async (file) => {
    const raw = localStorage.getItem(`upload:${await fileSig(file)}`);
    return raw ? JSON.parse(raw) : undefined;
  },
  onUploadBegin: async ({ file }) => {
    if (file.uploadId) {
      localStorage.setItem(
        `upload:${await fileSig(file.raw)}`,
        JSON.stringify({ uploadId: file.uploadId, key: file.objectInfo.key }),
      );
    }
  },
  onUploadComplete: async ({ file }) => {
    localStorage.removeItem(`upload:${await fileSig(file.raw)}`);
  },
});
```

```ts
route({
  multipart: true,
  onBeforeUpload({ file }) {
    if (file.resume && !file.resume.key.startsWith(`users/${userId}/`)) {
      throw new RejectUpload('Not yours.');
    }
  },
});
```

Multi-file, direct utility (`uploadFile`/`uploadFiles`), TTL pruning, and a one-call `withResumePersistence` wrapper: see [`apps/docs/content/docs/routes-multiple.mdx`](./apps/docs/content/docs/routes-multiple.mdx) → **Resume**.

## Upstream

[better-upload.com](https://better-upload.com)

## License

MIT
