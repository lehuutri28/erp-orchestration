# Rule 17 — File Upload Security

> Mức độ: 🔴 CRITICAL | Áp dụng: Mọi endpoint nhận file (ảnh, tài liệu, Excel, CSV, PDF)

---

## 1. Threat model khi nhận file

| Threat | Ví dụ | Mitigation |
|--------|-------|-----------|
| Path traversal | `../../etc/passwd` | Sanitize filename, dùng UUID làm key |
| MIME spoofing | PNG extension nhưng PHP bên trong | Magic byte check (không tin Content-Type) |
| Zip bomb | 1MB nén → 10GB giải nén | Giới hạn uncompressed size |
| Malware trong binary | PDF chứa macro | Virus scan (ClamAV hoặc cloud API) |
| Storage leak | Upload không xóa temp | Cleanup cron + lifecycle rule R2 |
| Cross-tenant isolation | Tenant A đọc file tenant B | Prefix path với `tenant_id/` |

---

## 2. Pipeline upload bắt buộc (thứ tự không đảo)

```
[Client] → [Multipart parse] → [Size check] → [MIME magic byte check]
         → [Filename sanitize] → [Virus scan (async)] → [Store R2]
         → [Metadata lưu DB] → [Response URL]
```

**KHÔNG** trả về URL public ngay nếu virus scan chưa xong. Dùng trạng thái `pending_scan`.

---

## 3. Giới hạn size theo loại file

| Loại | Max size | Lý do |
|------|----------|-------|
| Ảnh sản phẩm | 5 MB | Thumbnail resize, không cần 4K gốc |
| Ảnh CCCD / giấy tờ | 10 MB | Scanner quality |
| Excel import | 20 MB | ~100k rows |
| PDF hợp đồng | 30 MB | Bản scan nhiều trang |
| Video (LMS) | 500 MB | Giới hạn upload trực tiếp, >500 MB dùng presigned multipart |

```typescript
// Multer config BẮT BUỘC
const upload = multer({
  limits: {
    fileSize: 20 * 1024 * 1024,  // 20 MB default — override per endpoint
    files: 10,                    // Tối đa 10 file / request
  },
  fileFilter: (_req, file, cb) => {
    const ALLOWED_MIME = new Set([
      'image/jpeg', 'image/png', 'image/webp',
      'application/pdf',
      'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
      'application/vnd.ms-excel',
      'text/csv',
    ]);
    if (!ALLOWED_MIME.has(file.mimetype)) {
      return cb(new UnsupportedFileTypeError(file.mimetype));
    }
    cb(null, true);
  },
});
```

---

## 4. Magic byte validation (OWASP A05)

Không tin `Content-Type` header hoặc file extension — client có thể giả. Đọc magic byte thực tế:

```typescript
import { fileTypeFromBuffer } from 'file-type';  // package: file-type

export async function validateMagicBytes(
  buffer: Buffer,
  declaredMime: string,
): Promise<void> {
  const result = await fileTypeFromBuffer(buffer);
  
  if (!result) {
    throw new InvalidFileError('Không đọc được magic bytes — file có thể bị corrupt');
  }
  
  if (result.mime !== declaredMime) {
    throw new InvalidFileError(
      `MIME thực tế (${result.mime}) khác MIME khai báo (${declaredMime})`,
    );
  }
  
  // Blacklist double extension: file.php.jpg
  const DANGEROUS_EXTENSIONS = /\.(php|exe|sh|bat|cmd|ps1|phtml|asp|aspx|jsp)$/i;
  if (DANGEROUS_EXTENSIONS.test(declaredMime)) {
    throw new InvalidFileError('Extension nguy hiểm bị từ chối');
  }
}
```

---

## 5. Filename sanitization — ngăn path traversal

```typescript
import { randomUUID } from 'crypto';
import path from 'path';

export function sanitizeUploadKey(
  originalName: string,
  tenantId: string,
  folder: string,
): string {
  // Lấy extension gốc (tối đa 10 ký tự để tránh path traversal dạng .tar.gz.php)
  const ext = path.extname(originalName).slice(0, 10).toLowerCase();
  
  // UUID làm tên file — không dùng originalName làm path
  const uuid = randomUUID();
  
  // Prefix tenant_id để isolate storage
  return `${tenantId}/${folder}/${uuid}${ext}`;
}

// ❌ CẤM: dùng tên gốc trực tiếp
const key = `uploads/${originalName}`;  // Path traversal nếu name = '../../etc/passwd'

// ✅ ĐÚNG
const key = sanitizeUploadKey(file.originalname, user.tenantId, 'products');
```

---

## 6. Drizzle schema — lưu metadata, KHÔNG lưu binary trong DB

```typescript
// Lưu metadata vào DB, binary lên R2
export const mediaFiles = pgTable('media_files', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenant_id: uuid('tenant_id').notNull().references(() => tenants.id),
  
  original_name: text('original_name').notNull(),    // Tên gốc để hiển thị
  storage_key: text('storage_key').notNull().unique(), // Key trong R2 (UUID-based)
  mime_type: text('mime_type').notNull(),
  size_bytes: integer('size_bytes').notNull(),
  
  scan_status: text('scan_status', {
    enum: ['pending', 'clean', 'infected', 'error'],
  }).notNull().default('pending'),
  
  uploaded_by: uuid('uploaded_by').references(() => users.id),
  uploaded_at: timestamp('uploaded_at').defaultNow().notNull(),
  deleted_at: timestamp('deleted_at'),               // Soft delete — xóa R2 riêng
  
  // Metadata tùy loại file
  width: integer('width'),     // Ảnh
  height: integer('height'),   // Ảnh
  page_count: integer('page_count'),  // PDF
});
```

---

## 7. Cloudflare R2 integration (ADR 0003)

```typescript
// infrastructure/storage/r2.client.ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

@Injectable()
export class R2StorageClient {
  private readonly s3: S3Client;
  private readonly bucket = process.env.R2_BUCKET_NAME;

  constructor() {
    this.s3 = new S3Client({
      region: 'auto',
      endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
      credentials: {
        accessKeyId: process.env.R2_ACCESS_KEY_ID,
        secretAccessKey: process.env.R2_SECRET_ACCESS_KEY,
      },
    });
  }

  async upload(key: string, body: Buffer, contentType: string): Promise<void> {
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: body,
      ContentType: contentType,
      // Không set public ACL — dùng presigned URL
      ServerSideEncryption: 'AES256',
    }));
  }

  // Presigned URL — hết hạn 1 giờ, không expose public URL
  async getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
    return getSignedUrl(
      this.s3,
      new GetObjectCommand({ Bucket: this.bucket, Key: key }),
      { expiresIn },
    );
  }
}
```

**Không dùng public R2 bucket** — mọi file phải qua presigned URL để kiểm soát truy cập theo tenant.

---

## 8. Virus scan — ClamAV async

```typescript
// Scan chạy background sau khi upload lên R2 thành công
@Injectable()
export class VirusScanWorker {
  constructor(
    private readonly pgBoss: PgBoss,
    private readonly mediaRepo: MediaFilesRepository,
    private readonly alertService: AlertService,
  ) {}

  // Job enqueue ngay sau upload
  async enqueueScan(mediaId: string): Promise<void> {
    await this.pgBoss.send('virus-scan', { mediaId }, {
      retryLimit: 3,
      retryDelay: 30,  // seconds
    });
  }

  // Worker xử lý
  async processScan(job: PgBoss.Job<{ mediaId: string }>): Promise<void> {
    const { mediaId } = job.data;
    const media = await this.mediaRepo.findById(mediaId);

    // Download từ R2 → scan → update status
    const buffer = await this.r2.download(media.storage_key);
    const result = await this.clamAV.scan(buffer);

    if (result.isInfected) {
      await this.mediaRepo.markInfected(mediaId);
      await this.r2.delete(media.storage_key);  // Xóa ngay
      await this.alertService.telegram(`INFECTED file: ${mediaId} tenant=${media.tenant_id}`);
    } else {
      await this.mediaRepo.markClean(mediaId);
    }
  }
}
```

---

## 9. API endpoint chuẩn

```typescript
@Post('upload')
@UseGuards(JwtAuthGuard)
@UseInterceptors(FileInterceptor('file', multerConfig))
async uploadFile(
  @UploadedFile() file: Express.Multer.File,
  @CurrentUser() user: AuthUser,
  @Query('folder') folder: string,
): Promise<UploadResponse> {
  // 1. Magic byte check
  await this.uploadService.validateMagicBytes(file.buffer, file.mimetype);

  // 2. Sanitize key
  const storageKey = sanitizeUploadKey(file.originalname, user.tenantId, folder);

  // 3. Upload R2
  await this.r2.upload(storageKey, file.buffer, file.mimetype);

  // 4. Lưu metadata DB
  const media = await this.mediaRepo.create({
    tenant_id: user.tenantId,
    original_name: file.originalname,
    storage_key: storageKey,
    mime_type: file.mimetype,
    size_bytes: file.size,
    uploaded_by: user.id,
  });

  // 5. Enqueue virus scan (async, không block)
  await this.virusScanWorker.enqueueScan(media.id);

  return { mediaId: media.id, status: 'pending_scan' };
  // KHÔNG trả URL ngay — chờ scan xong
}
```

---

## 10. Anti-pattern ❌

| Anti-pattern | Vấn đề |
|-------------|--------|
| `trust file.mimetype` từ multer | Client bypass được bằng Content-Type header giả |
| `accept: '*/*'` không filter | Cho phép upload file nguy hiểm |
| Lưu `originalname` làm storage key | Path traversal: `../../etc/passwd` |
| Trả URL public ngay sau upload | File chưa scan, có thể phát tán malware |
| Không có size limit | DoS bằng file 100GB |
| Lưu binary BYTEA trong PostgreSQL | DB bloat, không scale, không CDN được |
| Public R2 bucket | Bất kỳ ai biết key đều đọc được |
| Không soft delete trước khi xóa R2 | Mất track, không audit được |

---

## 11. Checklist trước khi deploy endpoint upload

- [ ] `fileFilter` whitelist MIME type
- [ ] Magic byte validation bằng `file-type`
- [ ] `limits.fileSize` đặt phù hợp loại file
- [ ] Filename dùng UUID, KHÔNG dùng original name
- [ ] Storage key prefix `{tenant_id}/`
- [ ] Virus scan enqueued (async OK)
- [ ] Presigned URL thay vì public URL
- [ ] Metadata lưu `media_files` với `scan_status`
- [ ] Test: upload file PHP extension → bị reject
- [ ] Test: upload file lớn quá limit → 413
- [ ] Test: tenant A không lấy được presigned URL của tenant B

---

## Cross-ref

- `rules/01-security.md#A05` — Broken Access Control + security headers
- `rules/02-multi-tenant.md` — Storage isolation per tenant
- `rules/10-error-handling.md` — Error response format
- `skills/api-development/SKILL.md` — Workflow endpoint chuẩn
- `docs/adr/0003-*.md` — ADR chọn Cloudflare R2
- `skills/threat-modeling/SKILL.md` — STRIDE cho feature nhạy cảm
