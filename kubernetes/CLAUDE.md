# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Communication with the author and the thesis text are in Czech. Source code, comments, commit messages, all project documentation and these CLAUDE.md files are in English, because the project targets an international audience (the EU and Europe).

## This directory: kubernetes

Kubernetes manifests defining the workloads of every part except the native Android app (`server`, `api`, `web-app`, `crawler`, the database) for deploying the application on the canonical domain, which is meant to be publicly accessible from the internet. A data centre has not been secured yet; the school one will not suffice for this purpose. Sensitive data belongs in GitHub Secrets or Kubernetes Secrets, never in the repository.

## Thesis context

Litrivo is the practical part of the author's master's thesis (Ing. degree, part-time study, Faculty of Electrical Engineering and Computer Science, VSB – Technical University of Ostrava, FEI VŠB-TUO). Supervisor: prof. Miroslav Vozňák.

- **Working title of the thesis:** „Aplikace pro vyhledávání a porovnávání čerpacích stanic“ (An application for finding and comparing petrol stations)
- **Topic:** a full web and mobile application for comparing fuel prices by geographic location and a given driving range, accessible from anywhere on the internet. The author wants to keep running it after graduation (a native iOS app is not planned for now but not ruled out; iPhone users can install the PWA) and to develop it as open source.
- **Thesis goals:**
  - the frontend (Android + web as a PWA) talks to the server over a REST API,
  - use of location services (GPS, browser and system geolocation),
  - sorting: nearest, by price, optimal (a trade-off between price and distance),
  - map data and petrol stations shown on a map.
- **Predecessor:** PetrolScan, a proof of concept from the author's bachelor's thesis. It obtained prices by crawling websites, effectively supported only two petrol station chains and ran only inside the school network. Litrivo is meant to primarily **crowdsource** prices.

### Thesis materials

- **Thesis folder (school OneDrive):**
  `C:\Users\marpo\OneDrive - VSB-TUO\VŠB-TUO\Fakulta Elektrotechniky a Informatiky\Kombinované studium\Ing\Diplomová práce`
  - `informace.txt` — assignment, goals and a task checklist with current status. Read it before any planning.
  - `nazev-aplikace-zdroje.md` — research behind the app name: sources to cite, verification methodology, trademark status
  - `brandsearch.bib` — the same sources for biblatex (VŠB-TUO template, `style=iso-numeric`, biber). Just add `\addbibresource{brandsearch.bib}`; no custom `\usepackage{biblatex}`. Source provenance is in the `keywords` field: `kolo1-4` = from the ChatGPT research, `kolo5` = added during verification. Verification status is in `%` comments above the entries.
  - `priloha-konverzace-nazev.md` — transcript of the ChatGPT conversation (rounds 1–4) as a thesis appendix
- **Artifact „Rešerše značky Litrivo“ (Litrivo brand research):** https://claude.ai/artifact/CxLHCMTLY9Efo3ZAEp4yi5 — summary of the name research (domains, competition, trademarks). Read it with the Artifact tool (`action: read`), not WebFetch.
- The OneDrive files and the artifact belong to the author. Modify them only on the author's explicit request.

### Decisions so far (as of 2026-09-26)

- **Name:** Litrivo. Claim „Ceny paliv ve tvém dojezdu.“ / "Fuel prices within your range." Fallback name: Trasiva.
- **Canonical domain: `litrivo.com`.** The other three redirect to it with 301 and 1:1 subdomain mapping, i.e. `*.litrivo.{cz,eu,sk}` → `*.litrivo.com`. The application is served **only on the canonical host**; the other domains are redirects only. The API will run on `api.litrivo.com`. Rationale: `.cz` is the only one of the four locked to a country, and expanding into Europe would force a migration of the canonical domain, whereas the Czech brand loses nothing on `.com`.
- **Domains registered on 2026-09-26** with VEDOS for three years: `litrivo.cz` until 2029-09-25; `litrivo.sk`, `litrivo.eu` and `litrivo.com` until 2029-09-26. Price CZK 2,904 excl. VAT. All four are delegated to `ben.ns.cloudflare.com` and `hazel.ns.cloudflare.com`, i.e. into a single Cloudflare zone. `.app` is not registered.
- **Android package ID: `com.litrivo.app`**, following the canonical domain. It cannot be changed later on Google Play, so double-check it when creating the app.
- **Trademarks:** searched in TMview on 2026-09-26, **no conflict**. For `Litrivo` the only similar mark is ELITRIVOX (Canada, Mexico, USA, class 21 — household goods), which conflicts in neither sound, territory nor class. `Trasiva` returned no results. Additionally checked in business registers: ARES (CZ) and Register právnických osôb (SK) return zero entities. Relevant classes for a possible application: 9 (software), 35 (comparison, advertising), 42 (SaaS). A trademark registration is **not** a prerequisite for running the app.
- **Google Play:** developer account created, ID document uploaded, identity verification in progress; the phone number is verified only after that. The public contact e-mail `support@litrivo.com` (Cloudflare Email Routing) is verified. The author's legal name and address are not published on Play; only the developer name and contact e-mail are public — except for monetized apps, where the full address is always published. New personal accounts must pass **closed testing with 12 testers for 14 consecutive days** before a production release.
- **Logo:** the author will create the black-and-white version personally. Colour variants legible on both light and dark backgrounds will be added later, once the master is finished (ideally as SVG).
- **Open tasks** (per `informace.txt`): GitHub organization, 301 domain redirects, data centre and Kubernetes cluster (hosting with a friend who runs hosting services is being considered), development environment, initial database of petrol stations partly populated by the crawler (brands, stations, locations; prices are crowdsourced only), GitHub Secrets.
- **Direct competitors:** TankUp (formerly Tankuj, Droppy Technologies s.r.o., released 2026-09-09, markets CZ/AT/SK/HR/SI), Tank Navigator (prices from CCS transactions), FuelNook Tankora, Tanklio, FuelRadar (IE and AU), FuelScope UK, FuelScout, ZeFuel, NexFuel, Fuelza, Pump Scout. The list with sources is in the research.

### Writing the thesis

Conversations with AI (ChatGPT, Claude) are **not cited as a source of facts** in the thesis. Every claim needs its own primary source (a registry, an official API, a website). AI tools belong in the AI-use declaration. Citations carry a date `[cit. YYYY-MM-DD]` per ISO 690.

## Monorepo layout

The project is **a single monorepo** rooted at `D:\GITHUB\Litrivo` (repository `litrivo` in the Litrivo GitHub organization). This directory is a subdirectory of it, not a standalone repository — it commits into the shared history.

| Subdirectory | Purpose |
|---|---|
| `api` | backend: REST API service over the database, plus its OpenAPI specification — the single source of truth for the REST contract; clients are generated from it |
| `server` | public-facing reverse proxy (HTTP/HTTPS, ports 80 and 443) in front of `api` and `web-app` |
| `web-app` | web frontend (PWA) |
| `android-app` | Android mobile app |
| `crawler` | indexes petrol station brands, stations and their locations; does not collect prices |
| `db` | database: schema, migrations and seed data (users, brands, stations, crowdsourced prices) |
| `kubernetes` | deployment (Kubernetes manifests) |

All subdirectories are empty so far. Build, lint and test commands and an architecture description will be added once there is code. The project is licensed under MIT (`LICENSE` in the repository root). The rationale for the monorepo is in the root `CLAUDE.md`.

A REST contract change typically touches `api`, `web-app` and `android-app` at once — thanks to the monorepo it belongs in **one commit**, not several coordinated ones.
