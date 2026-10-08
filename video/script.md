# Fizca - 5 minute portfolio video script

Audience: recruiters and portfolio viewers.
Target length: about 5 minutes, roughly 745 words at a calm pace.
Format: narration with on-screen cues. `[SHOW: ...]` is a diagram or the app, `[CODE: file:lines]` is a code file to put on screen.

Diagrams live in `video/diagrams/`. Code references with exact lines are in `video/code_references.md`.

---

## Beat 1 - What it is (0:00 - 0:30)

`[SHOW: the app, Fizca login / Moments feed]`

This is Fizca. It is a private, invite-only photo and memory journal.

You create a profile for someone you care about, a kid or a pet, and fill it over time with photos and journal entries. It all lands on one timeline you can scroll and filter by tag.

I built the whole thing. The React frontend, the CSS, the REST API, the image pipeline, the database model, and the cloud infrastructure. Every choice here was deliberate, and I will walk through the ones I am proud of.

`[SHOW: diagrams/1_system_topology.png]`

---

## Beat 2 - The frontend I designed (0:30 - 1:30)

Let's start where the user does. The frontend is React with strict TypeScript, built with Vite, and state managed with MobX. I picked MobX over Redux because it stays out of my way. I mutate plain objects and the UI reacts.

I did not use a component library for the look. The design system is hand-coded: CSS custom properties for the tokens, and styled-components for the pieces. The transitions and the lightbox use Framer Motion.

`[CODE: client/src/components/usePageFetch.ts:7-39]`

The feed is an infinite scroll I wrote myself, no React Query or SWR. A small hook fetches pages and cancels stale requests, and an IntersectionObserver triggers the next page as you reach the last card. A few files instead of a dependency, and I understand every line.

`[SHOW: diagrams/4_data_model.png]`

The data model was its own decision. Photos and journal moments are separate records, but I stitch them into one timeline, sorted by when they happened and filterable by tag.

---

## Beat 3 - The backend and the security choices (1:30 - 3:00)

`[SHOW: diagrams/2_image_pipeline.png]`

The backend is a plain Express REST API. The part I care most about is how images are handled, because that is where the security decisions live.

`[CODE: server/services/asset-handler.js:44-70]`

When you upload a photo, I pull the EXIF data off it, the GPS location and the date it was taken, so a photo knows where and when it happened. I hash the filename with SHA-256 so it is opaque and not guessable. Then Sharp resizes it into three sizes, and those go to S3.

`[CODE: server/routes/assets-route.js:158-183]`

Here is the deliberate part. The S3 bucket is private. Images are never public, and the server is the only thing with access to it. It either streams a photo back through an authenticated request, or it hands out a pre-signed URL that only lives for five minutes. Either way, nobody ever gets a permanent public link to your photos.

`[SHOW: diagrams/3_auth_flow.png]`
`[CODE: server/routes/auth-route.js:12-33]`

Auth is Google OAuth, and I offloaded it on purpose. I do not store passwords. When you sign in, the client hands me a Google token and I verify it offline against Google's public keys. No callback round-trip, and no client secret to leak. On top of that it is a closed system. Only pre-invited accounts can log in, and everyone else gets turned away. Identity lives in a server-side session, not a token sitting in the browser.

---

## Beat 4 - The infrastructure (3:00 - 4:30)

`[SHOW: diagrams/1_system_topology.png, full]`

Now the part that makes this cheap to run. The whole backend is serverless, and it costs me close to nothing until about a million requests a month.

The Express app ships as a container image on AWS Lambda, using the Lambda Web Adapter. That means the exact same image runs on my laptop and in production. My rule was simple: the contract is the image, not the language. If I ever rewrite the backend in Go, the infrastructure does not change. I just ship a new image.

The frontend is on Cloudflare Pages, and this is my favorite trick. The browser only ever talks to one origin. A tiny Cloudflare function proxies every `/api` call to AWS behind the scenes. So there is no CORS, because there is no cross-origin request to begin with. I designed the problem out instead of configuring around it. That proxy also hides the AWS backend behind a shared secret.

`[CODE: terraform/README.md:1-15]`

All of this is one Terraform codebase across three clouds: AWS, Cloudflare, and MongoDB Atlas. And the deploys are keyless. GitHub Actions assumes a role through OIDC, so there are no long-lived AWS keys sitting in my CI.

---

## Beat 5 - Tradeoffs and close (4:30 - 5:00)

A couple of honest tradeoffs. Serverless means cold starts, so I connect to the database in the background with a retry loop, and the app never hangs waiting on it. The Atlas allowlist is open by design, because Lambda's outbound IP keeps changing, so I lean on TLS and database auth instead.

That is Fizca. One person, one stack, from the CSS tokens down to the Terraform. Every layer was a decision, and you have just seen the ones that mattered most.

Thanks for watching.

---

### Delivery notes
- Pace: aim for about 150 words per minute. If you run long, Beat 2 is the easiest to trim.
- The `[CODE:]` cues are good moments to zoom the editor to those exact lines.
- Diagrams are 16:9 (1600x900) and drop straight into a 1080p timeline.
