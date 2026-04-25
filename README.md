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

```tsx
useUploadFile({
  route: 'videos',
  getResumeState: (file) => {
    const raw = localStorage.getItem(`upload:${file.name}:${file.size}`);
    return raw ? JSON.parse(raw) : undefined;
  },
  onUploadBegin: ({ file }) => {
    if (file.uploadId) {
      localStorage.setItem(
        `upload:${file.name}:${file.size}`,
        JSON.stringify({ uploadId: file.uploadId, key: file.objectInfo.key }),
      );
    }
  },
  onUploadComplete: ({ file }) => {
    localStorage.removeItem(`upload:${file.name}:${file.size}`);
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
