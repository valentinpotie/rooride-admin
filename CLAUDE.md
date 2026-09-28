# rooride-admin

Static admin dashboard for Rooride, one file (`index.html`), served by GitHub Pages from `main`.
**Pushing to `main` deploys to admin.rooride.app within a couple of minutes.**

## Current state (28 September 2026)

- The dashboard was rebuilt on 27 and 28 September 2026 around five screens: Overview, Users, ID verification, Trips, Money. The old Statistics, Advanced search and Due diligence pages are gone.
- **The anonymised view (accounts carrying the `dd` claim) has not been re-tested since the rebuild. Marion is handling it on 28 September 2026. Please do not touch the `dd` paths in the meantime.**
- `adminUserAction` (Cloud Function, `australia-southeast1`) is deployed but its IAM policy is empty, so it answers 403. It needs `roles/cloudfunctions.invoker` for `allUsers`; the function checks the `staff` claim itself. Marion's editor role cannot set it.

## Rules

- No secrets in this repository: it is public.
- Firestore settings: keep `experimentalAutoDetectLongPolling`, never `experimentalForceLongPolling` (it breaks sign-in).
- Work locally with a no-cache server (`Cache-Control: no-store`), otherwise the browser serves a stale page.
