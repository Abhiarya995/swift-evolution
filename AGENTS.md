# Working in this repository (Base44 sandbox notes)

## What this project is
The Swift Evolution status page: a **static site with no build step and no
backend**. `index.html` + `index.css` + `index.js` render a list of proposals,
and `proposals/*.md` holds the proposals themselves. Everything is read directly
by the browser at runtime, so there is no bundler, package manager, or
compilation to run.

## Running it
```
docker compose -f docker-compose.base44.yml up -d
```
`docker-compose.base44.yml` serves the repository root on host port 3000 with
`python3 -m http.server` from a `python:3.12-slim` image, bind-mounting the repo
at `/app`. Because files are read per request, **edits appear on page reload —
do not rebuild the image after a source change** (a rebuild is only needed if
the compose file itself changes).

Verify:
```
docker compose -f docker-compose.base44.yml ps
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/
```

## No credentials required
The app needs no environment variables, secrets, or local services. `/run/base44/app.env`
is not wired into the compose file on purpose.

## Where the proposal metadata comes from (important)
`index.js` renders from a JSON document fetched at runtime, not from the
markdown files in `proposals/`:

* `PROPOSALS_DATA_URL` (in `index.js`) points at the **swift.org published
  metadata**: `https://download.swift.org/swift-evolution/v1/evolution.json`.
  This response is sent with `Access-Control-Allow-Origin: *`, so the browser
  may fetch it cross-origin.
* The older endpoint this page used, `https://data.swift.org/swift-evolution/proposals`,
  was **retired with the swift.org migration and now returns S3 `403 AccessDenied`**
  (as does `https://data.swift.org/swift-evolution/swift.svg`, the header logo,
  which now points at `https://www.swift.org/assets/images/icon-swift.svg`).
  That is why these URLs differ from older revisions of this repository.
* The published payload is nested (`{"proposals": [...], "implementationVersions": [...]}`)
  and shaped slightly differently from what the page renders: undotted status
  states (`implemented` vs `.implemented`), an array of `reviewManagers`, and
  tracking bugs without `assignee`/`status`. `normalizeProposals()` in `index.js`
  adapts the payload once on load and also refreshes `languageVersions` from
  `implementationVersions`. Keep the rest of the file expecting the adapted shape.

## How to verify it actually works
A healthy page is **not** just HTTP 200: `index.html` loads without data and can
show "Proposal data failed to load." Load `http://localhost:3000/` in a browser
and confirm the proposal list renders (hundreds of entries, `N proposals` count
in the header area, status pills such as *Implemented* / *Accepted*), the logo
shows in the header, and the search box plus the status/version filter panel
narrow the list. Failure modes to look for in the console:

* `Proposal data failed to load.` → the metadata fetch failed (network/CORS).
* `undefined` shown in a bug's assignee/status text → the payload shape changed
  again and `normalizeProposals()` needs updating.

## Layout notes
* `proposals/` is content only (362 markdown files); it is not read by the page.
* `_site/` is gitignored built output and is not used by this setup.
