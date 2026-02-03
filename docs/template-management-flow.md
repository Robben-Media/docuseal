# Template Management Flow (Create/Edit/Apply)

This document traces how templates are created, edited, and applied to submissions in the web UI and API.

## Entry Points

UI routes
- `config/routes.rb:86` (template upload form submit)
- `config/routes.rb:95` (templates CRUD)
- `config/routes.rb:98` (template documents add/replace)
- `config/routes.rb:103` (template submissions create)
- `config/routes.rb:132` (shared link start form)

Template edit UI
- `app/controllers/templates_controller.rb:29`
- `app/views/templates/edit.html.erb:9`
- `app/javascript/application.js:148`

API
- `app/controllers/api/templates_controller.rb:7`
- `app/controllers/api/submissions_controller.rb:52`

## Create Templates

1. Create empty template (name + folder).
   - Form: `app/views/templates/_file_form.html.erb:1`
   - Controller: `TemplatesController#create` assigns author/account/folder and saves.
   - `app/controllers/templates_controller.rb:46`

2. Create template from uploaded file(s).
   - Dropzone submits to `templates_upload_path`.
   - `app/views/templates/_dropzone.html.erb:1`
   - Upload handler creates template, attaches docs, extracts fields, and updates schema.
   - `app/controllers/templates_uploads_controller.rb:10`

3. Clone template.
   - Controller: `TemplatesCloneController#create`.
   - `app/controllers/templates_clone_controller.rb:12`
   - Clone logic duplicates submitters/fields/schema/preferences with new UUIDs.
   - `lib/templates/clone.rb:8`
   - Attachments are duplicated with remapped attachment UUIDs.
   - `lib/templates/clone_attachments.rb:7`

4. Clone and replace documents.
   - Controller: `TemplatesCloneAndReplaceController#create`.
   - `app/controllers/templates_clone_and_replace_controller.rb:6`
   - Document replacement updates schema + field areas.
   - `lib/templates/replace_attachments.rb:8`

## Edit Templates (Builder)

1. Edit page preloads schema documents + preview images and passes JSON to builder.
   - `app/controllers/templates_controller.rb:29`
   - `app/views/templates/edit.html.erb:9`

2. Builder mount and config.
   - `app/javascript/application.js:148`

3. Autosave / explicit save sends full template payload.
   - `app/javascript/template_builder/builder.vue:2862`
   - `TemplatesController#update` persists and emits webhook + reindex.
   - `app/controllers/templates_controller.rb:64`

4. Add documents in builder.
   - Upload component posts to `/templates/:id/documents`.
   - `app/javascript/template_builder/upload.vue:219`
   - Server adds documents and returns `schema`, `documents`, and optional `fields`/`submitters`.
   - `app/controllers/template_documents_controller.rb:12`
   - Builder merges response and saves.
   - `app/javascript/template_builder/builder.vue:2432`

5. Replace documents in builder.
   - Rewrites schema entry and rewires areas to new attachment UUID.
   - `app/javascript/template_builder/builder.vue:2512`

6. Detect fields (SSE).
   - `TemplatesDetectFieldsController#create` streams detected fields per page.
   - `app/controllers/templates_detect_fields_controller.rb:8`
   - Builder inserts non-overlapping fields and saves.
   - `app/javascript/template_builder/builder.vue:2674`

## Apply Templates (Create Submissions)

1. UI: send to recipients from a template.
   - `SubmissionsController#create` normalizes params, creates submissions, and sends.
   - `app/controllers/submissions_controller.rb:40`
   - Normalization may add new fields for submitter values.
   - `lib/submissions/normalize_param_utils.rb:7`
   - Submission captures snapshot of template fields/schema/submitters.
   - `lib/submissions/create_from_submitters.rb:10`
   - `app/models/submission.rb:52`

2. Shared link flow (`/d/:slug`).
   - Loads template by slug, initializes submission + submitter.
   - `app/controllers/start_form_controller.rb:18`
   - Submission uses template snapshot defaults.
   - `app/controllers/start_form_controller.rb:150`

3. API: create submissions for a template.
   - Validates template has fields, normalizes params, and creates submissions.
   - `app/controllers/api/submissions_controller.rb:52`
   - `app/controllers/api/submissions_controller.rb:146`

## Storage Model (Template vs Submission)

- Templates store live `schema`, `fields`, `submitters` in JSON columns.
  - `app/models/template.rb:52`
- Submissions store snapshots (`template_schema`, `template_fields`, `template_submitters`) at creation time.
  - `app/models/submission.rb:52`
