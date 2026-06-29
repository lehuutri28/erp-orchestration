# Skill: LMS — Đào tạo Pha chế

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS camelCase, RBAC dấu `.`. Xem docs/ARCHITECTURE_CONTRACT.md.

> Khi nào load: Build module course, enrollment, payment, certificate

---

## Domain context

WECHA cung cấp 2 loại đào tạo:
1. **Public courses** (có phí): khoá pha chế, kinh doanh trà sữa cho người ngoài → thanh toán online + cấp chứng chỉ điện tử
2. **Franchise training** (miễn phí): đào tạo cho đối tác nhượng quyền → enroll qua hợp đồng franchise

---

## Schema

```typescript
// Khoá học
export const courses = pgTable('courses', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  code: text('code').notNull(),              // 'PHACHE-CB-001'
  title: text('title').notNull(),            // 'Pha chế trà sữa cơ bản'
  slug: text('slug').notNull(),
  description: text('description'),
  thumbnailUrl: text('thumbnail_url'),
  trailerVideoUrl: text('trailer_video_url'),
  
  type: text('type').notNull(),              // 'public', 'franchise', 'internal'
  category: text('category'),                // 'beverage', 'business', 'management'
  level: text('level').notNull(),            // 'beginner', 'intermediate', 'advanced'
  
  durationHours: decimal('duration_hours', { precision: 6, scale: 2 }),
  totalLessons: integer('total_lessons').notNull().default(0),
  
  // Pricing
  price: decimal('price', { precision: 15, scale: 2 }).notNull().default('0'),
  discountPrice: decimal('discount_price', { precision: 15, scale: 2 }),
  currency: text('currency').notNull().default('VND'),
  
  // Certificate
  hasCertificate: boolean('has_certificate').notNull().default(true),
  passingScore: integer('passing_score').notNull().default(70),  // % điểm đậu
  certificateTemplateId: text('certificate_template_id'),
  
  // Material kit (nguyên liệu thực hành)
  materialKitId: text('material_kit_id'),  // FK to course_kits
  
  // Status
  status: text('status').notNull().default('draft'),  // 'draft', 'published', 'archived'
  publishedAt: timestamp('published_at', { withTimezone: true }),
  
  ...auditFields,
}, (t) => ({
  uniqCode: uniqueIndex('courses_tenant_code_unique').on(t.tenantId, t.code),
  uniqSlug: uniqueIndex('courses_tenant_slug_unique').on(t.tenantId, t.slug),
  publishedIdx: index('courses_published_idx').on(t.tenantId, t.status, t.publishedAt),
}));

// Bài học
export const lessons = pgTable('lessons', {
  id: uuid('id').primaryKey().defaultRandom(),
  courseId: uuid('course_id').notNull().references(() => courses.id),
  moduleId: uuid('module_id'),              // Group lessons into modules
  
  orderIndex: integer('order_index').notNull(),
  title: text('title').notNull(),
  description: text('description'),
  
  type: text('type').notNull(),              // 'video', 'text', 'quiz', 'practical', 'live_session'
  durationMinutes: integer('duration_minutes'),
  
  videoUrl: text('video_url'),              // Cloudflare Stream URL
  content: text('content'),                  // Markdown
  attachments: jsonb('attachments'),
  
  isPreview: boolean('is_preview').notNull().default(false),
  isFree: boolean('is_free').notNull().default(false),
  
  ...auditFields,
});

// Quiz/Test
export const quizzes = pgTable('quizzes', {
  id: uuid('id').primaryKey().defaultRandom(),
  lessonId: uuid('lesson_id').notNull().references(() => lessons.id),
  
  title: text('title').notNull(),
  totalQuestions: integer('total_questions').notNull(),
  timeLimitMinutes: integer('time_limit_minutes'),
  passingScore: integer('passing_score').notNull().default(70),
  maxAttempts: integer('max_attempts').default(3),
  
  ...auditFields,
});

export const quizQuestions = pgTable('quiz_questions', {
  id: uuid('id').primaryKey().defaultRandom(),
  quizId: uuid('quiz_id').notNull().references(() => quizzes.id),
  
  orderIndex: integer('order_index').notNull(),
  questionText: text('question_text').notNull(),
  questionType: text('question_type').notNull(),  // 'single', 'multi', 'text', 'true_false'
  
  options: jsonb('options'),                 // [{ id, text, is_correct }]
  correctAnswer: text('correct_answer'),    // For text-type
  explanation: text('explanation'),
  points: integer('points').notNull().default(1),
});

// Enrollment
export const enrollments = pgTable('enrollments', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  courseId: uuid('course_id').notNull().references(() => courses.id),
  studentId: uuid('student_id').notNull(),  // user_id hoặc customer_id
  
  enrollmentType: text('enrollment_type').notNull(),  // 'paid', 'franchise_free', 'gift'
  enrolledAt: timestamp('enrolled_at', { withTimezone: true }).notNull().defaultNow(),
  
  // Payment (nếu paid)
  orderId: uuid('order_id'),                // Link to orders table
  paymentId: uuid('payment_id'),
  amountPaid: decimal('amount_paid', { precision: 15, scale: 2 }).default('0'),
  
  // Progress
  progressPercent: decimal('progress_percent', { precision: 5, scale: 2 }).notNull().default('0'),
  completedLessons: integer('completed_lessons').notNull().default(0),
  lastAccessedLessonId: uuid('last_accessed_lesson_id'),
  lastAccessedAt: timestamp('last_accessed_at', { withTimezone: true }),
  
  // Completion
  status: text('status').notNull().default('active'),  
  // 'active', 'completed', 'expired', 'cancelled', 'refunded'
  completedAt: timestamp('completed_at', { withTimezone: true }),
  finalScore: decimal('final_score', { precision: 5, scale: 2 }),
  
  // Certificate
  certificateId: uuid('certificate_id'),
  certificateIssuedAt: timestamp('certificate_issued_at', { withTimezone: true }),
  
  // Access control
  expiresAt: timestamp('expires_at', { withTimezone: true }),  // Time-limited access
  
  ...auditFields,
}, (t) => ({
  uniqEnrollment: uniqueIndex('enrollment_unique').on(t.courseId, t.studentId),
  studentIdx: index('enrollment_student_idx').on(t.tenantId, t.studentId, t.enrolledAt),
}));

// Lesson progress tracking
export const lessonProgress = pgTable('lesson_progress', {
  id: uuid('id').primaryKey().defaultRandom(),
  enrollmentId: uuid('enrollment_id').notNull().references(() => enrollments.id),
  lessonId: uuid('lesson_id').notNull().references(() => lessons.id),
  
  status: text('status').notNull().default('not_started'),  // 'not_started', 'in_progress', 'completed'
  watchedSeconds: integer('watched_seconds').notNull().default(0),
  totalSeconds: integer('total_seconds'),
  
  startedAt: timestamp('started_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  
  // Quiz result (if applicable)
  quizScore: decimal('quiz_score', { precision: 5, scale: 2 }),
  quizAttempts: integer('quiz_attempts').notNull().default(0),
}, (t) => ({
  uniqProgress: uniqueIndex('lesson_progress_unique').on(t.enrollmentId, t.lessonId),
}));

// Certificate (chứng chỉ điện tử)
export const certificates = pgTable('certificates', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull(),
  
  certificateNumber: text('certificate_number').notNull(),  // 'PL-20260529-0001' (theo CONTRACT mục 8)
  enrollmentId: uuid('enrollment_id').notNull().references(() => enrollments.id),
  
  studentName: text('student_name').notNull(),    // Snapshot
  studentIdCard: text('student_id_card'),        // CCCD (encrypted)
  courseTitle: text('course_title').notNull(),
  
  issuedAt: timestamp('issued_at', { withTimezone: true }).notNull(),
  validUntil: date('valid_until'),                // Nullable = vô thời hạn
  
  finalScore: decimal('final_score', { precision: 5, scale: 2 }),
  grade: text('grade'),                            // 'Pass', 'Distinction', 'High Distinction'
  
  pdfUrl: text('pdf_url'),                        // Cloudflare R2 URL
  qrVerifyUrl: text('qr_verify_url'),            // Public verification URL
  
  // Verification
  verificationHash: text('verification_hash').notNull(),  // SHA-256 of cert content
  blockchainTx: text('blockchain_tx'),                    // Optional: blockchain notary
  
  ...auditFields,
}, (t) => ({
  uniqNumber: uniqueIndex('cert_number_unique').on(t.certificateNumber),
  studentIdx: index('cert_student_idx').on(t.tenantId, t.studentName),
}));
```

---

## Payment flow (VNPay/MoMo)

```typescript
// Service: enroll vào public course (có phí)
async enrollPaidCourse(courseId: string, userId: string, paymentMethod: 'vnpay' | 'momo') {
  return this.db.transaction(async (tx) => {
    const course = await this.coursesRepo.findById(courseId, tx);
    
    if (course.type !== 'public' || course.status !== 'published') {
      throw new BadRequestException('Khoá học không khả dụng');
    }
    
    // Check duplicate enrollment
    const existing = await this.enrollmentsRepo.findByCourseStudent(courseId, userId, tx);
    if (existing && existing.status === 'active') {
      throw new ConflictException('Bạn đã đăng ký khoá học này');
    }
    
    // Create order
    const price = course.discount_price ?? course.price;
    const order = await this.ordersService.create({
      customer_id: userId,
      type: 'course',
      items: [{ product_id: course.id, quantity: 1, unit_price: price }],
      payment_method: paymentMethod,
    }, tx);
    
    // Create enrollment in 'pending_payment' state
    const enrollment = await tx.insert(enrollments).values({
      tenant_id: this.tenantId,
      course_id: courseId,
      student_id: userId,
      enrollment_type: 'paid',
      order_id: order.id,
      amount_paid: price,
      status: 'pending_payment',
    }).returning();
    
    // Create payment URL
    const paymentUrl = await this.paymentService.createCheckoutUrl({
      order_id: order.id,
      amount: price,
      method: paymentMethod,
      return_url: `https://wecha.vn/courses/${course.slug}/enrollment-success`,
      ipn_url: 'https://api.wecha.vn/api/v1/payments/ipn',
    });
    
    return { enrollment: enrollment[0], paymentUrl };
  });
}

// Webhook IPN từ VNPay/MoMo
async handlePaymentIpn(payload: any, signature: string, gateway: string) {
  // Verify signature
  if (!this.paymentService.verifySignature(payload, signature, gateway)) {
    throw new UnauthorizedException('Invalid signature');
  }
  
  if (payload.status !== 'success') return;
  
  return this.db.transaction(async (tx) => {
    const order = await this.ordersRepo.findById(payload.order_id, tx);
    const enrollment = await this.enrollmentsRepo.findByOrderId(order.id, tx);
    
    if (enrollment.status !== 'pending_payment') return;  // Idempotent
    
    // Activate enrollment
    await tx.update(enrollments).set({
      status: 'active',
      payment_id: payload.transaction_id,
    }).where(eq(enrollments.id, enrollment.id));
    
    // Mark order paid
    await tx.update(orders).set({ status: 'paid' }).where(eq(orders.id, order.id));
    
    // Issue e-invoice (NĐ 70/2025)
    await this.eInvoiceService.issueForOrder(order.id, tx);
    
    // Send welcome email + access info
    await this.emailService.sendCourseWelcome(enrollment.student_id, order.id);
  });
}
```

---

## Certificate generation

```typescript
async issueCertificate(enrollmentId: string) {
  return this.db.transaction(async (tx) => {
    const enrollment = await this.enrollmentsRepo.findById(enrollmentId, tx);
    
    // Validate
    if (enrollment.status !== 'completed') {
      throw new BadRequestException('Chưa hoàn thành khoá học');
    }
    
    const course = await this.coursesRepo.findById(enrollment.course_id, tx);
    if (!course.has_certificate) {
      throw new BadRequestException('Khoá học không có chứng chỉ');
    }
    
    if (Number(enrollment.final_score) < course.passing_score) {
      throw new BadRequestException(`Điểm chưa đạt (yêu cầu >= ${course.passing_score}%)`);
    }
    
    if (enrollment.certificate_id) {
      throw new ConflictException('Chứng chỉ đã được cấp');
    }
    
    // Generate certificate number
    const certNumber = await this.generateCertNumber(tx);  // CER-WECHA-2026-001234
    
    const student = await this.usersService.findById(enrollment.student_id);
    
    const certData = {
      certificate_number: certNumber,
      student_name: student.full_name,
      course_title: course.title,
      issued_at: new Date(),
      final_score: enrollment.final_score,
      grade: this.calculateGrade(Number(enrollment.final_score)),
    };
    
    // Generate verification hash (SHA-256)
    const verificationHash = createHash('sha256')
      .update(JSON.stringify(certData))
      .digest('hex');
    
    // Generate PDF
    const pdfBuffer = await this.pdfService.generateCertificatePDF({
      ...certData,
      qr_data: `https://verify.wecha.vn/cert/${certNumber}`,
    });
    
    // Upload to R2
    const pdfUrl = await this.storageService.uploadCertificate(certNumber, pdfBuffer);
    
    // Insert
    const [cert] = await tx.insert(certificates).values({
      tenant_id: this.tenantId,
      ...certData,
      enrollment_id: enrollmentId,
      pdf_url: pdfUrl,
      qr_verify_url: `https://verify.wecha.vn/cert/${certNumber}`,
      verification_hash: verificationHash,
    }).returning();
    
    // Link to enrollment
    await tx.update(enrollments).set({
      certificate_id: cert.id,
      certificate_issued_at: new Date(),
    }).where(eq(enrollments.id, enrollmentId));
    
    // Email cert to student
    await this.emailService.sendCertificate(student.email, pdfUrl);
    
    return cert;
  });
}
```

---

## Public verification endpoint

```typescript
@Get('/cert/:certNumber')
@Public()  // No auth required
async verifyCertificate(@Param('certNumber') certNumber: string) {
  const cert = await this.db.query.certificates.findFirst({
    where: eq(certificates.certificate_number, certNumber),
  });
  
  if (!cert) {
    return { valid: false, message: 'Chứng chỉ không tồn tại' };
  }
  
  // Recompute hash to verify
  const recomputedHash = createHash('sha256')
    .update(JSON.stringify({
      certificate_number: cert.certificate_number,
      student_name: cert.student_name,
      course_title: cert.course_title,
      issued_at: cert.issued_at,
      final_score: cert.final_score,
      grade: cert.grade,
    }))
    .digest('hex');
  
  const isValid = recomputedHash === cert.verification_hash;
  
  if (cert.valid_until && new Date() > cert.valid_until) {
    return {
      valid: false,
      message: 'Chứng chỉ đã hết hạn',
      issued_at: cert.issued_at,
      valid_until: cert.valid_until,
    };
  }
  
  return {
    valid: isValid,
    certificate_number: cert.certificate_number,
    student_name: cert.student_name,
    course_title: cert.course_title,
    issued_at: cert.issued_at,
    grade: cert.grade,
    issuer: 'CÔNG TY TNHH VUA AN TOÀN (WECHA)',
  };
}
```

---

## Anti-patterns

- ❌ Cho phép access lesson nếu chưa active enrollment
- ❌ Cấp chứng chỉ tự động khi chưa đủ score
- ❌ Số chứng chỉ trùng nhau (không có unique constraint)
- ❌ PDF chứng chỉ store local server (mất nếu server fail)
- ❌ Verification không có hash → fake được dễ
- ❌ Refund không xử lý revoke certificate
- ❌ Material kit không track inventory → ship rồi mới biết hết hàng

---

**END Skill: LMS Courses**
