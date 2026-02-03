# Document Upload + Sealing Flow

This document traces the template upload path and the end-to-end sealing pipeline for completed submissions.

## Entry Points

Template upload UI
- `app/views/templates/_dropzone.html.erb:1`
- `app/views/templates/_upload_button.html.erb:1`
- `app/views/templates_uploads/show.html.erb:12`

Template upload routes
- `config/routes.rb:86`
- `config/routes.rb:88`

Template documents API (add/replace docs)
- `config/routes.rb:98`
- `app/controllers/template_documents_controller.rb:11`

Submitter form routes
- `config/routes.rb:144`
- `config/routes.rb:152`

Submitter form controller
- `app/controllers/submit_form_controller.rb:16`
- `app/controllers/submit_form_controller.rb:50`

Attachments upload API (signatures/files/images)
- `app/controllers/api/attachments_controller.rb:10`

Completion processing job
- `app/jobs/process_submitter_completion_job.rb:6`

## Document Upload Flow (Templates)

1. User selects or drops files in template UI. The custom element toggles loading and submits the form.
   - `app/views/templates/_dropzone.html.erb:1`
   - `app/javascript/elements/file_dropzone.js:12`
   - `app/javascript/elements/submit_form.js:1`

2. `TemplatesUploadsController#create`:
   - creates the template record
   - uploads attachments
   - extracts/normalizes fields (if template has no fields yet)
   - updates schema
   - enqueues webhook + search reindex
   - redirects to template editor
   - `app/controllers/templates_uploads_controller.rb:8`

3. `Templates::CreateAttachments.call`:
   - extracts ZIP entries
   - filters by content type
   - creates ActiveStorage blobs
   - passes PDFs/images to `Templates::ProcessDocument`
   - `lib/templates/create_attachments.rb:21`

4. `Templates::ProcessDocument`:
   - renders preview images for PDFs/images (Pdfium + Vips)
   - extracts AcroForm fields when enabled
   - writes metadata on attachment
   - `lib/templates/process_document.rb:17`
   - `lib/templates/find_acro_fields.rb:35`

5. For existing templates, `TemplateDocumentsController#create` reuses `CreateAttachments` and returns
   updated schema/fields/previews in JSON.
   - `app/controllers/template_documents_controller.rb:11`

## Submitter Flow + Sealing

1. Submitter opens `/s/:slug`; Vue `submission-form` is mounted with schema/values/attachments.
   - `app/controllers/submit_form_controller.rb:16`
   - `app/javascript/form.js:19`

2. Signature or file upload uses `/api/attachments`:
   - UI uses `dropzone.vue` for file uploads and `draw.js` for drawn signatures.
   - server validates image content and persists to `submitter.attachments`.
   - `app/javascript/submission_form/dropzone.vue:133`
   - `app/javascript/draw.js:115`
   - `app/controllers/api/attachments_controller.rb:10`
   - `lib/submitters.rb:123`

3. Submitter submits:
   - `SubmitFormController#update` calls `Submitters::SubmitValues.call`.
   - On completion it enqueues `ProcessSubmitterCompletionJob`.
   - `app/controllers/submit_form_controller.rb:73`
   - `lib/submitters/submit_values.rb:21`
   - `lib/submitters/submit_values.rb:34`

4. `ProcessSubmitterCompletionJob`:
   - generates result attachments
   - if all submitters completed, generates combined document + audit trail
   - sends completion notifications and webhooks
   - `app/jobs/process_submitter_completion_job.rb:13`
   - `app/jobs/process_submitter_completion_job.rb:16`
   - `app/jobs/process_submitter_completion_job.rb:20`

5. `Submissions::EnsureResultGenerated`:
   - lock-based coordination to prevent duplicate generation
   - calls `GenerateResultAttachments`
   - `lib/submissions/ensure_result_generated.rb:14`

6. `Submissions::GenerateResultAttachments`:
   - builds PDFs from schema documents
   - overlays field values and signature images
   - applies signature ID/reason blocks
   - optionally signs the PDF (PKCS + TSA)
   - `lib/submissions/generate_result_attachments.rb:84`
   - `lib/submissions/generate_result_attachments.rb:201`
   - `lib/submissions/generate_result_attachments.rb:302`
   - `lib/submissions/generate_result_attachments.rb:721`

7. Combined document and audit trail:
   - combined PDF generated via `GenerateCombinedAttachment`
   - audit trail PDF generated via `GenerateAuditTrail`
   - both can be digitally signed
   - `lib/submissions/ensure_combined_generated.rb:15`
   - `lib/submissions/generate_combined_attachment.rb:7`
   - `lib/submissions/ensure_audit_generated.rb:15`
   - `lib/submissions/generate_audit_trail.rb:33`

## API Examples

Attachment upload (signature/image/file) to `/api/attachments`:
```bash
curl -X POST \
  -F "file=@/path/to/signature.png" \
  -F "submitter_slug=SUBMITTER_SLUG" \
  -F "name=attachments" \
  -F "remember_signature=true" \
  https://YOUR_HOST/api/attachments
```

Example success response:
```json
{
  "uuid": "att-uuid",
  "created_at": "2026-02-03T20:31:00Z",
  "url": "https://YOUR_HOST/file/signed_uuid/filename.png",
  "filename": "signature.png",
  "content_type": "image/png"
}
```

Submit form step (or completion) to `/s/:slug`:
```bash
curl -X POST \
  -F "values[FIELD_UUID]=John Doe" \
  -F "values[SIGNATURE_UUID]=ATTACHMENT_UUID" \
  -F "completed=true" \
  -F "timezone=America/New_York" \
  https://YOUR_HOST/s/SUBMITTER_SLUG
```

Notes:
- `completed=true` triggers finalization and enqueues `ProcessSubmitterCompletionJob`.
- `timezone` is captured for audit metadata.
- Optional `touch_attachment_uuid` updates the attachment timestamp when reusing a signature.
- Optional `with_reason` adds a required reason field linked to the signature field.

## Digital Signing + Timestamping

Signing certs and TSA:
- PKCS loaded from account config or defaults. `lib/accounts.rb:108`
- TSA URL from account config. `lib/accounts.rb:134`
- Timestamp handler posts RFC3161 requests. `lib/submissions/timestamp_handler.rb:25`

PDF signing integration points:
- Result PDFs: `lib/submissions/generate_result_attachments.rb:721`
- Combined PDF: `lib/submissions/generate_combined_attachment.rb:25`
- Audit trail: `lib/submissions/generate_audit_trail.rb:48`

## Data Model Touchpoints

Templates
- attachments on template: `app/models/template.rb:65`
- schema + fields: `app/models/template.rb:52`

Submissions
- attachments: `app/models/submission.rb:64`
- schema documents resolution: `app/models/submission.rb:112`

Submitters
- attachments (signatures/files): `app/models/submitter.rb:58`
- completed state: `app/models/submitter.rb:68`

## Key Dependencies

PDF + imaging
- HexaPDF for PDF build/sign: `lib/submissions/generate_result_attachments.rb:711`
- Pdfium + Vips for previews and image work: `lib/templates/process_document.rb:100`

Storage + async
- ActiveStorage for all document persistence: `lib/templates/create_attachments.rb:41`
- Sidekiq job for completion processing: `app/jobs/process_submitter_completion_job.rb:3`

## Notable Behaviors and Risks

ZIP extraction accepts DOC/RTF/ODT/etc but `handle_file_types` only accepts PDF/images.
Non-PDF docs from ZIP will raise `InvalidFileType`.
- `lib/templates/create_attachments.rb:70`
- `lib/templates/create_attachments.rb:95`

Lock-event timeouts are capped at 90 seconds, which can surface as timeouts for large documents.
- `lib/submissions/ensure_result_generated.rb:7`
- `lib/submissions/ensure_combined_generated.rb:7`
- `lib/submissions/ensure_audit_generated.rb:7`

Signature sealing embeds signature metadata (reason, timestamp, ID) into PDF canvas, tied to attachment UUID and
`created_at`.
- `lib/submissions/generate_result_attachments.rb:316`
- `lib/submissions/generate_result_attachments.rb:355`

## Essential Files (Minimum Set)

- `config/routes.rb:86`
- `app/controllers/templates_uploads_controller.rb:8`
- `app/controllers/template_documents_controller.rb:11`
- `lib/templates/create_attachments.rb:21`
- `lib/templates/process_document.rb:17`
- `lib/templates/find_acro_fields.rb:35`
- `app/controllers/submit_form_controller.rb:16`
- `app/controllers/api/attachments_controller.rb:10`
- `lib/submitters/submit_values.rb:21`
- `app/jobs/process_submitter_completion_job.rb:13`
- `lib/submissions/ensure_result_generated.rb:14`
- `lib/submissions/generate_result_attachments.rb:84`
- `lib/submissions/generate_combined_attachment.rb:7`
- `lib/submissions/generate_audit_trail.rb:33`
- `lib/submissions/timestamp_handler.rb:12`
