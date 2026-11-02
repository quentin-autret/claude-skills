# File upload, streaming and file storage

## Contents
- Uploads with interceptors
- Validation and limits
- Fastify multipart
- Streaming responses
- @nestjs/storage disks
- Uploading to a disk
- Serving and signed URLs
- Production and testing

## Uploads with interceptors

- Default storage is Multer `memoryStorage()`: the whole file is in `file.buffer` before the handler runs, and Multer sets no size limit. Always set `limits.fileSize`. https://docs.nestjs.com/http/file-upload#memory-usage-and-storage
  ```typescript
  @UseInterceptors(FileInterceptor('file', { limits: { fileSize: 5 * 1024 * 1024 } }))
  ```
- Prefer `limits.fileSize` over `MaxFileSizeValidator` alone: the limit stops the upload as it happens (413), the validator runs after the full file arrived. https://docs.nestjs.com/http/file-upload#file-size-limit
- `diskStorage()` keeps memory flat. Generate names yourself and delete files after processing, including on failure. From v12.2, when a pipe rejects the request, the stored files are removed once the error response has been sent. https://docs.nestjs.com/http/file-upload#disk-storage
- Interceptor options override `MulterModule.register()`. Since v12.0.4 the two `limits` objects merge key by key. https://docs.nestjs.com/http/file-upload#default-options
- v12.2 bounds field names: `fieldNestingDepth` 32, `fieldArrayIndexLimit` 1000, exceeding gives 400. https://docs.nestjs.com/http/file-upload#field-name-limits
- With the Node adapter, import interceptors from `@nestjs/platform-node`. https://docs.nestjs.com/http/file-upload

## Validation and limits

- `ParseFilePipe` answers 400 when any validator fails and requires a file unless `fileIsRequired: false`. `FileTypeValidator` checks magic numbers by default. https://docs.nestjs.com/http/file-upload#file-validation
  ```typescript
  @UploadedFile(new ParseFilePipeBuilder()
    .addFileTypeValidator({ fileType: 'jpeg' })
    .addMaxSizeValidator({ maxSize: 1000 })
    .build({ errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY })) file: Express.Multer.File
  ```

## Fastify multipart

- From v12.1 the same interceptors work on Fastify, imported from `@nestjs/platform-fastify/multipart` and backed by `@fastify/multipart`, which the adapter registers on first use. Files are typed `UploadedMultipartFile`. They fail to start on `ExpressAdapter`. https://docs.nestjs.com/http/file-upload#fastify
- Fastify limits are stricter: `fileSize` defaults to `bodyLimit` (1 MiB). Raise per route, per `MultipartModule`, or app-wide with `new FastifyAdapter({ multipart: { limits } })`. https://docs.nestjs.com/http/file-upload#limits
- Set the `multipart` adapter option to `true` when an interceptor is created outside DI or used only in lazy-loaded modules. https://docs.nestjs.com/http/file-upload#plugin-registration
- `FileStreamInterceptor()` (Fastify only) gives a `stream`: text fields must come before the file, only 1 file is accepted, in the given field (another field gives 400), `MaxFileSizeValidator` and the magic number check of `FileTypeValidator` do not apply. https://docs.nestjs.com/http/file-upload#streaming-uploads

## Streaming responses

- Return `new StreamableFile(stream, { type, disposition, length })` rather than piping into `@Res()`, so interceptors still run. Use `@Res({ passthrough: true })` for computed headers. https://docs.nestjs.com/http/file-upload#streaming-files
- Default `Content-Type` is `application/octet-stream`. Set `length` yourself for streams. https://docs.nestjs.com/http/file-upload#response-headers
- Since v12.2, an Express stream error no longer puts the message in the body. Customize with `setErrorHandler()` and `setErrorLogger()`. On Fastify the exceptions layer handles it. https://docs.nestjs.com/http/file-upload#stream-errors

## @nestjs/storage disks

- `@nestjs/storage` gives one API over named disks: `LocalDisk`, `S3Disk` (S3, R2, MinIO, B2, GCS through S3), `InMemoryDisk`. It talks to S3 over `fetch`, no AWS SDK. Multer or Fastify multipart still parse the request. https://docs.nestjs.com/application/file-storage
- Keep public and private files on separate disks (separate buckets in production). There is no per-file public flag. https://docs.nestjs.com/application/file-storage#production-checklist
- `StorageModule` is global. Inject `StorageDisk` (default disk), `@InjectDisk('photos')`, or `Storage` with `storage.disk(name)`. `default` is required when there are several disks. https://docs.nestjs.com/application/file-storage#module-options
- Startup fails on a signing key under 32 characters, an unknown `default`, or a class injecting a disk name no config provides. The first key signs and all verify, so rotate by putting the new key first. https://docs.nestjs.com/application/file-storage#configure-the-disks
- Errors extend `StorageError` and are not `HttpException`s outside the HTTP helpers: map them (`StorageFileNotFoundError` 404, `StorageInvalidKeyError` 400 and so on). https://docs.nestjs.com/application/file-storage#errors

## Uploading to a disk

```typescript
@UseInterceptors(FileInterceptor('photo', {
  storage: uploadToDisk({ disk: 'photos', contentTypes: PHOTO_TYPES, cacheControl }),
  limits: { fileSize, files: 1 },
}))
upload(@UploadedFile() file: StoredUpload | undefined) {}
```
https://docs.nestjs.com/application/file-storage#upload-product-photos

- `uploadToDisk()` detects the type from the first bytes and refuses unlisted types with 415 before writing, enforces the size limit without partial files (413), and keys files by random UUID. Use `file.contentType`, `file.mimetype` is still the client's claim. https://docs.nestjs.com/application/file-storage#upload-product-photos
- `ParseFilePipe` runs after the file is already stored. Check type and size in the engine. The file is stored even if the handler then rejects (unknown product): delete it to avoid orphans. https://docs.nestjs.com/application/file-storage#upload-product-photos
- Formats without a signature (CSV) need a `detectContentType` function. Return `undefined` for unknown files rather than throwing (a throw becomes 500). Detection classifies, it does not validate the content. https://docs.nestjs.com/application/file-storage#accept-csv-and-other-text-formats
- Direct uploads with `signedUpload()` sign the declared type and size, but those are client claims: on completion accept only keys you handed out, re-check size with `stat()`, sniff the first bytes, then copy to the public disk. https://docs.nestjs.com/application/file-storage#upload-straight-to-storage-from-the-mobile-app
- On Fastify, register `addContentTypeParser('*', (req, payload, done) => done(null))` so signed upload bodies reach the handler unread. https://docs.nestjs.com/application/file-storage#upload-straight-to-storage-from-the-mobile-app

## Serving and signed URLs

- `serveFile(disk, key, { req, res })` with `@Res({ passthrough: true })` handles Range, `If-None-Match`, 404, `nosniff`, and forces `attachment` for HTML and SVG. https://docs.nestjs.com/application/file-storage#upload-product-photos
- `signedUrl(key, { expiresIn })` defaults to 15 minutes, max 7 days. A link cannot be revoked: keep it short (5 minutes) and check ownership before issuing it. https://docs.nestjs.com/application/file-storage#production-checklist https://docs.nestjs.com/application/file-storage#keep-invoices-private
- `LocalDisk` signed URLs point at your app: add a route calling `serveSignedUrl(disk, { req, res })`. https://docs.nestjs.com/application/file-storage#serve-signed-urls-in-development

## Production and testing

- Do not use `LocalDisk` in production (container disk vanishes, instances do not share it). https://docs.nestjs.com/application/file-storage#production-checklist
- S3 IAM needs `s3:ListBucket`, otherwise a missing file answers 403 and `exists()` throws. Add a lifecycle rule for abandoned `incoming/` uploads. https://docs.nestjs.com/application/file-storage#production-checklist
- Keys are validated so input cannot escape, but can still name any file: build keys from ids you control. https://docs.nestjs.com/application/file-storage#production-checklist
- Tests: `overrideProvider(STORAGE_MODULE_OPTIONS).useValue({ default, disks: { ... InMemoryDisk } })`. https://docs.nestjs.com/application/file-storage#testing
