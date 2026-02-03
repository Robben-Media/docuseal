# Authentication and Sessions

This document describes how authentication, sessions, and related security flows work in Docuseal.

## Overview

Docuseal uses three primary authentication modes:

- Web UI: cookie-based sessions via Devise.
- API: token-based authentication via `X-Auth-Token` header.
- Console SSO: short-lived JWT used for redirecting into the console app.

Key files:

- Web UI sessions: `app/controllers/sessions_controller.rb`, `app/controllers/application_controller.rb`, `app/models/user.rb`, `config/initializers/devise.rb`
- MFA setup: `app/controllers/mfa_setup_controller.rb`, `app/controllers/dashboard_controller.rb`
- Password reset: `app/controllers/passwords_controller.rb`, `app/controllers/users_send_reset_password_controller.rb`
- API auth: `app/controllers/api/api_base_controller.rb`, `app/models/access_token.rb`
- Console JWT: `app/controllers/console_redirect_controller.rb`, `lib/json_web_token.rb`

## Web UI Authentication (Devise)

### Session entry points

Routes are configured via Devise to handle sessions and passwords:

- `config/routes.rb` sets `devise_for :users` with sessions and passwords only.

### Sign-in flow

`SessionsController` customizes Devise sign-in:

- Email is lowercased before checks.
- In multitenant mode, if the email does not exist, users are redirected to registration.
- If the account requires OTP and no `otp_attempt` is provided, the OTP form is rendered.
- `otp_attempt` is permitted in Devise parameter sanitizer.
- `after_sign_in_path_for` honors a `redir` param and supports console redirects.

Files:

- `app/controllers/sessions_controller.rb`
- `config/routes.rb`

### Global auth enforcement

`ApplicationController` enforces authentication for all non-Devise controllers:

- `before_action :authenticate_user!, unless: :devise_controller?`
- A demo mode auto-signs in a random user.
- If no users exist, requests are redirected to setup.

File:

- `app/controllers/application_controller.rb`

### User model behavior

The user model includes Devise modules and custom behavior:

- Devise modules: `two_factor_authenticatable`, `recoverable`, `rememberable`, `validatable`, `trackable`, `lockable`.
- Archived users or archived accounts cannot authenticate.
- `remember_me` always returns true (persistent sessions).
- After password reset, auto-sign-in is disabled if OTP is required.

File:

- `app/models/user.rb`

### Devise settings

Important configuration in `config/initializers/devise.rb`:

- Two-factor strategy is inserted into the default Warden stack.
- Remember-me duration is configurable via `SESSION_REMEMBER_DAYS` (default 730 days).
- Remember tokens are invalidated on sign-out.
- Password length is 6..128.
- Reset password tokens are valid for 6 hours.
- Sign-out uses the `DELETE` HTTP verb.

File:

- `config/initializers/devise.rb`

## MFA (TOTP)

### Setup and enable

`MfaSetupController` drives the OTP setup flow:

- Generates `otp_secret` if missing.
- Builds provisioning URI for authenticator apps.
- `validate_and_consume_otp!` enables MFA by setting `otp_required_for_login`.

### Disable

- Valid OTP disables MFA, clears `otp_required_for_login`, and removes `otp_secret`.

Files:

- `app/controllers/mfa_setup_controller.rb`

### Forced MFA policy

If account policy requires MFA and the user has not enabled it, dashboard requests are redirected to setup:

- `AccountConfig::FORCE_MFA` is checked in `DashboardController`.

File:

- `app/controllers/dashboard_controller.rb`

## Password Reset

### Standard reset

`PasswordsController` customizes Devise reset behavior:

- In non-multitenant mode, reset errors are cleared after request.
- `PasswordsController::Current.user` is set during reset to support conditional sign-in.
- After reset, users are redirected to the sign-in page.

Files:

- `app/controllers/passwords_controller.rb`
- `app/models/user.rb` (sign-in-after-reset logic)

### Admin-triggered reset

Admins can trigger reset emails for users, with a 10-minute send limit:

- `UsersSendResetPasswordController` checks `reset_password_sent_at` before sending.

File:

- `app/controllers/users_send_reset_password_controller.rb`

## API Authentication (X-Auth-Token)

API controllers inherit from `ApiBaseController` and require authentication by default:

- Reads `X-Auth-Token` header.
- Hashes the token with SHA-256 and looks it up against `access_tokens.sha256`.
- Returns `401` if no user is resolved.

Files:

- `app/controllers/api/api_base_controller.rb`
- `app/models/access_token.rb`

### Token storage and rotation

- Tokens are encrypted at rest using `encrypts :token`.
- A SHA-256 hash is stored for lookup.
- Token rotation is handled in `ApiSettingsController`.
- Revealing the token in UI requires a password check.

Files:

- `app/models/access_token.rb`
- `app/controllers/api_settings_controller.rb`
- `app/controllers/reveal_access_token_controller.rb`

### API docs

Client examples use the `X-Auth-Token` header in the API docs.

Files:

- `docs/openapi.json`
- `docs/api/*`

## Console SSO (JWT Redirect)

`ConsoleRedirectController` issues short-lived JWTs for console redirects:

- Payload includes `uuid`, `scope: :console`, and 1-minute expiry.
- JWT is signed using `secret_key_base`.

Files:

- `app/controllers/console_redirect_controller.rb`
- `lib/json_web_token.rb`

## Impersonation

Impersonation is powered by Pretender:

- `ApplicationController` and `ApiBaseController` both support `impersonates :user`.
- Testing account switching uses impersonation and stores `impersonated_user_id` in session.

Files:

- `app/controllers/application_controller.rb`
- `app/controllers/api/api_base_controller.rb`
- `app/controllers/testing_accounts_controller.rb`

## Security Notes

- Sensitive parameters are filtered in logs (passwords, tokens, OTPs).
- OTP allowed drift is 60 seconds.

Files:

- `config/initializers/filter_parameter_logging.rb`
- `config/initializers/devise.rb`
