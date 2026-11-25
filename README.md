# gaffel

Dansk madplan-, tilbuds- og indkøbsapp til én husstand. Fire aftener om ugen, madplan lagt ud fra ugens tilbudsaviser, indkøbsliste med "har hjemme"-lager, og kvitteringsafstemning hvor `Varer − Rabatter = SUM` skal stemme på øre.

**Status: ikke kodet endnu.** Designet færdiggøres i OpenDesign; denne repo indeholder indtil videre beslutningsgrundlaget for kodefasen.

## Læs først

- [`docs/kodefase-beslutninger.md`](docs/kodefase-beslutninger.md) — stack, UI-lag, CSS-arkitektur, AI-integration, kvitteringsparser og de valg der blev truffet fra, med begrundelser og målte tal.

## Planlagt stack

| Lag | Valg |
|---|---|
| App | Next.js 16 (App Router, Turbopack), React 19, TypeScript strict |
| UI-adfærd | [`@ui-sdk/core`](https://ui-sdk.dev) — behaviors frem for komponenter (pinnet beta) |
| Styling | Håndskrevet CSS + `design/tokens.css` som eneste farvekilde + CSS Modules |
| Data | Lokal SQLite via Drizzle + `better-sqlite3`; PWA-cache via Serwist |
| AI | Vercel AI SDK 7, multi-provider BYOK, læseværktøjer mod lokal database |

## Licens

Ikke besluttet endnu.
