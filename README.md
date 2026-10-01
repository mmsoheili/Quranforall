# Quran for All

A PHP 8.4 / MySQL 8.4 application for private Quran lessons. The original static mockup remains in `recitera-apple-style-mockup.html`. The working platform is `index.php`.

## Run locally

The isolated MySQL runtime is in `storage/runtime/`, listens only on `127.0.0.1:3307`, and stores its data in `storage/mysql-data/`. It does not alter the machine's existing MariaDB service.

```powershell
powershell -File scripts/start.ps1
```

Open http://127.0.0.1:8000/. Always use `router.php` with the PHP development server: it prevents access to private configuration, data, scripts, and tests. Apache uses the included `.htaccess` and requires `mod_rewrite` / `AllowOverride`.

The local administrator credentials are in `storage/admin-access.local.txt`. Database credentials are in `config.local.php`; both are ignored by Git. Do not publish the workspace or `storage/` as static files.

## Workflow

1. A teacher registers, completes their profile, selects their timezone, and adds individual available 30-, 45-, or 60-minute slots.
2. The administrator signs in, opens **Manage teachers**, reviews the profile, assigns an HTTPS classroom link, and publishes the teacher.
3. A student registers, finds a teacher by name/language/subject, and books a future slot. Their optional learning goal is visible to that teacher.
4. Both participants see the booking in their dashboards. The teacher's classroom link is shown only to participants of confirmed upcoming or ongoing bookings and to administrators. It is never included in public profile queries.
5. Either participant can cancel before the lesson starts, reopening the slot. The teacher can mark an ended lesson complete. Cancellation history is retained.

The link is assigned per teacher, as requested. Its current value is used for upcoming bookings, so an administrator's update applies immediately. A permanent external link can be retained by participants after viewing it; use the video provider's waiting room / host admission controls to keep consecutive lessons private.

## Database setup on another host

Copy `config.example.php` to `config.local.php`, enter your MySQL connection settings, and run:

```text
php scripts/setup.php
php scripts/create-admin.php your-email@example.com
php -S 127.0.0.1:8000 router.php
```

Alternatively set `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` as environment variables. `setup.php` is additive and creates the schema; it never drops tables. The provisioning script is exclusively for an unconfigured isolated local server, not an existing production database. The generated admin credentials are written to the private local file, not the terminal.

## Decisions and research

- Keep the existing PHP project, use server-rendered pages and PDO prepared statements, and use MySQL as the only durable data store. This delivers a small functional application without a separate JavaScript backend. Laravel is a strong choice when the team needs framework-managed verification, password recovery, queues, billing, and a larger codebase; its [authentication documentation](https://laravel.com/docs/12.x/authentication) describes the available framework facilities. This implementation deliberately has fewer dependencies but owns its authentication code.
- Use InnoDB transactions, a row lock on the slot, a unique generated `active_slot` key, and a student row lock to serialize conflicting reservations. A teacher row lock prevents overlapping availability creation. See [MySQL locking reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html).
- Store dates as UTC and convert them with each user's IANA timezone; reject past and overlapping slots. Local times skipped by DST are rejected by round-trip validation. Repeated times when clocks move backward use PHP's timezone resolution; the current UI does not offer a first/second occurrence selector.
- Use PHP's [password hashing API](https://www.php.net/manual/en/function.password-hash.php), session ID regeneration, HttpOnly/SameSite cookies, CSRF tokens, escaped output, role checks, and database-backed login throttling. Follow [PHP session security guidance](https://www.php.net/manual/en/features.session.security.management.php).
- Administrator publication separates registration from public listing. Public users cannot register as an administrator. No invented teachers, ratings, or bookings are inserted into the live database.

## Verification

```text
php tests/integration.php
php tests/http.php
```

Tests create uniquely named fixture users and remove only their own records in `finally` blocks. HTTP tests expect the local router at port 8000. Integration tests cover actual MySQL transactions, overlapping times, double booking, student schedule conflicts, cancellation ownership, rebooking, and timezone conversion.

## Imported academy teachers

Seven public profiles from https://academy.recitera.com/ are stored in `database/teachers.recitera.json`, with locally stored portraits in `assets/teachers/`. Biographies are concise paraphrases; names, languages, countries, experience labels, specialties and teaching restrictions come from each linked profile. Original source URLs are visible on the detail pages. Abozar's experience badge says 16+ years while his biography says more than 15 years of teaching; these source values are retained rather than inventing a new figure.

Run `php scripts/import-teachers.php` to apply the additive detail migration and import missing profiles. Re-running preserves existing profiles, classroom links, schedules and edits. `php tests/imported-teachers.php` checks all seven live detail pages and images. `scripts/prepare-teachers.php` can rebuild the curated import manifest from the source HTML snapshots under the ignored `storage/imports/` directory.

Imported profiles are published, but do not have verified teacher email addresses or working teacher logins. Their internal `@imported.invalid` identifiers are not contact details, and their password hashes deliberately cannot authenticate. No availability or classroom link is fabricated. The administrator can assign links immediately; verified account onboarding and actual teacher schedules must be arranged before these profiles can accept bookings.

## Administrator center and sample availability

The existing administrator account now controls the full academy at `/index.php?page=admin`. Its credentials remain in `storage/admin-access.local.txt`.

- Overview: teacher/student counts and upcoming classes.
- Teachers: edit biography, subjects, languages, country, experience, teaching restrictions, photo, publication and classroom link.
- Users: create teacher/student accounts, edit name/email/timezone, reset passwords and disable accounts. Password resets and deactivation invalidate existing sessions. Imported profiles can be enabled by assigning a verified email and a new password here. Administrators cannot disable themselves or modify another administrator through this interface.
- Availability: add times for any active teacher, edit open times, close unwanted times and filter by teacher.
- Classes: see all bookings, assign a student to an available lesson, reschedule upcoming classes atomically, cancel upcoming classes and complete ended classes. History is retained rather than deleted.
- Activity log: timestamped events, actor, target and relevant change information, with event filtering and pagination. Passwords and private classroom URLs are excluded. Logging begins with this update; past events are not reconstructed. This is an application audit trail, not a tamper-proof external logging service.

For an existing database run `php scripts/migrate-admin.php` before using this update. Fresh setup includes these fields and tables. `php scripts/seed-sample-times.php` creates three labelled, bookable 30-minute sample slots per imported active teacher, based on Asia/Tehran time over the next three days. Re-running does not duplicate a teacher's existing upcoming samples. The administrator can edit a sample to make it a normal slot, or close it. No student bookings or meeting URLs are fabricated.

## Deployment boundaries

This is a working local initial release, not a publicly deployed service. It needs PHP hosting with MySQL; the Sites/Cloudflare Workers host cannot execute this PHP application or its direct PDO connection. Before public launch, configure HTTPS and `secure_cookies=true`, disable PHP error display, restrict the application DB account to runtime privileges, back up MySQL, and keep storage/configuration inaccessible from the web server. Use a production web server rather than `php -S`.

Payments, automated reminders, email verification, password recovery, recurring availability, and embedded video are not included. Classes use the links the administrator provides. Add an email provider and its credentials when implementing verification/recovery. The interface remains English to match the original design.
