# Code references for the video

Exact files and lines for each `[CODE:]` cue. Verified against the current source.

## Frontend

**Hand-built infinite scroll hook**
`client/src/components/usePageFetch.ts:7-39`
Generic `usePageFetch<T>(url, pageNumber)`. Fetches a page, appends results, cancels stale
requests with an axios CancelToken. Paired with an IntersectionObserver in
`client/src/containers/Moments.tsx` that watches the last card and bumps the page number.

Supporting files worth showing briefly:
- `client/src/models/Store.ts` - single global MobX store.
- `client/src/components/Boxes.ts`, `client/src/components/Headings.ts` - styled-components primitives.
- `client/src/App.css`, `client/src/index.css` - CSS custom-property design tokens (`var(--highlight)`, `var(--bg)`, `var(--border-radius)`).
- `client/src/components/Image.tsx` - Framer Motion image, builds `/api/assets/{size}/{src}`.

## Backend - image pipeline

**EXIF extraction + SHA-256 filename**
`server/services/asset-handler.js:44-70`
`ExtractMetadata` reads GPS latitude/longitude and capture date via exifr.
`HashFileName` SHA-256 hashes `originalname-userId` so filenames are opaque.

**Upload route (the full pipeline in one place)**
`server/routes/assets-route.js:26-87`
Multer writes to `/tmp` (line 14), EXIF + hash, Sharp fan-out to three sizes, upload each to S3,
upsert tags, create a Timeline entry, always unlink the temp file in `finally`.

## Backend - S3 signed URLs

**5-minute pre-signed GET URL**
`server/services/aws-s3.js:62-69`
`getSignedUrl(asset)` with `Expires: 60 * 5`. Bucket stays private; the server is the only
accessor. Note at lines 16-23: on Lambda the S3 client uses the execution role, no static keys.

Routes that serve images (both keep the bucket private):
- `server/routes/assets-route.js:158-172` - streams the object through the server.
- `server/routes/assets-route.js:177-183` - returns a 5-minute signed URL.

## Backend - Google OAuth

**Offline ID-token verification + session**
`server/routes/auth-route.js:12-33`
`client.verifyIdToken({ idToken, audience: client_id })` validates Google's signature locally.
`Authentication.GoogleUser(...)` enforces the invite allowlist in
`server/services/authentication.js`. Identity is stored in `req.session.user`
(Mongo-backed cookie session), not a JWT.

RBAC ladder: `server/services/user-roles.js` (`none < guest < contributor < admin`),
enforced by `AuthRole(...)` in `server/services/middlewares.js`.

## Infrastructure

**Topology + cost + the design decisions, in prose**
`terraform/README.md:1-15` (topology diagram), `187-191` (cost notes).

- Lambda container + Web Adapter: `server/Dockerfile`, `server/README.md:137-149`.
- Same-origin proxy (no CORS): `terraform/spa_proxy_example/functions/api/[[path]].js` and
  `client/functions/api/[[path]].js`.
- Multi-cloud providers: `terraform/providers.tf`, `terraform/versions.tf`.
- Keyless OIDC deploy role: `terraform/github_oidc.tf`.
- Secrets set-once pattern: `terraform/ssm.tf`.
