# CLAUDE.md — frappe-blog (SWE-Pioneers fork)

Changes made in this fork on top of upstream `frappe/blog` (v16 / `version-16`).

## 1. Arabic i18n
- `blog/locale/ar.po` — completed to 100% of app-owned strings (UI + doctype/field labels from a
  `bench generate-pot-file` pass), Libyan MSA, tagged `ai-translated; needs-native-review`.
- Verified live in Arabic (RTL) on the VPS demo (`sanad-blog.swe.com.ly`): portal chrome (مدونة,
  الرئيسية, حسابي), dates (١٠ يوليو ٢٠٢٦), comment form (اسمك / إضافة تعليق / تعليق).

## 2. Visitor commenting + moderation
All comment UI/submission is Frappe **core** (`templates/includes/comments/comments.html`); the app
only feeds `context.comment_list` + `context.guest_allowed` and hooks `Comment` events.

- **Anonymous comments**: enable `Blog Settings → Allow Guest to comment` (per-site). Guests then
  comment with name+email, no account (replaces the "Please login to post a comment" wall).
- **Register-to-comment**: enable website signup (`Website Settings → disable_signup = 0`). New
  registrants are the built-in **Website User** type — no custom role needed for identity.
- **Account-level moderation** (`blog/doctype/blog_post/blog_post.py`):
  - `moderate_comment` (Comment `after_insert`): if the commenter is not *trusted*, force
    `published=0` (held). Trusted = post author, System Manager, or an account holding the
    **"Approved Commenter"** role.
  - `is_trusted_commenter()` decides the above; anonymous guests are always moderated.
  - `publish_comments_on_account_approval` (User `on_update`): when a supervisor grants
    "Approved Commenter" to an account, that account's held Blog Post comments are released live.
  - `ensure_commenter_role()` (hooks `after_install` + `after_migrate`): idempotently creates the
    desk-less "Approved Commenter" role.
  - `load_comments()` defensively hides `published=0` comments from the public list.
  - **Supervisor workflow**: Desk → User → grant "Approved Commenter" to confirm an account (its
    held comments then go live), or Desk → Comment → toggle `Published` to approve a single comment.

## 3. Email-safe likes/comments
`safe_sendmail()` (+ `has_outgoing_email_account()`) in `blog_post.py`, used by `send_email` and
`templates/includes/likes/likes.py`. Notifications are skipped/guarded when a site has no default
outgoing Email Account — fixes the crash "Please setup default outgoing Email Account" on like/comment.

## 4. CSS bundle (MIME fix)
`web_include_css` previously pointed at the raw `blog.scss`, served as `application/octet-stream` and
refused by strict-MIME browsers. Now a compiled bundle: `blog/public/scss/blog.bundle.scss`
(Bootstrap/Frappe SCSS vars `$gray-*`/`$font-size-4xl`/`$text-muted` → CSS `var(--…)` with fallbacks;
`@include media-breakpoint-up(xl)` → `@media (min-width: 1200px)`) referenced as
`web_include_css = "blog.bundle.css"`. `bench build` compiles it to `assets/blog/dist/css[-rtl]/…`
(served as `text/css`, RTL variant auto-built for Arabic).

## Deploy notes
Multi-tenant demo stack `/opt/apps/demo-blog` on the VPS (image `blog-swe:v16-swe1`, built from
`~/build/blog-custom/Containerfile`). Rebuild with `--no-cache`; recreate with `docker compose up -d
--force-recreate`. New settings (`allow_guest_to_comment`, signup) are per-site — enabled on
`sanad-blog` as the showcase.
