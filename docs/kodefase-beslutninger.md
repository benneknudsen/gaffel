# Gaffel — tekniske beslutninger for kodefasen

Status: beslutningsgrundlag, skrevet før kodestart. Verificeret **2026-11-24** (versionsnumre fra npm, dokumentation fra udgiverne).
Designkilden er fortsat OpenDesign-projektet `8cc24855-1ec8-45b3-8f21-f516b28fa604` — dette dokument ændrer ikke designet, kun hvordan det bygges.

---

## 0. Faste rammer (arvet fra design-fasen)

| Rammesætning | Beslutning |
|---|---|
| Stack | Next.js 16.4.6, React 19.3.1, TypeScript strict |
| Framework-funktioner | Next 16.4 giver **Turbopack som standard** (2–5× hurtigere builds), **Cache Components med `use cache`** og `proxy.ts` i stedet for `middleware.ts`. Vi bruger `use cache` på opskriftsbank og statiske data — **aldrig** på priser eller tilbud uden tidsstempel (en forkert pris i butikken er værre end ingen). |
| Farver | `design/tokens.css` er **eneste** farvekilde. Ingen hex-koder i komponenter. |
| Visuel sandhed | `assets/gaffel.css` fra prototypen er referencen; én screenshot-baseline pr. skærm før den regnes som færdig |
| Test | Intet nyt test-framework i første iteration. Gate = `next build` + `tsc --noEmit` + manuel a11y-tjekliste (afsnit 1.9) |
| AI | Vercel AI SDK er **kerne fra start**, ikke et senere tilvalg (afsnit 2.1). Copilot er en hovedfunktion, ikke en detalje. |
| CSS | Håndskrevet CSS med design-tokens + CSS Modules. **Ingen Tailwind** (afsnit 0.1). |
| Sprog i UI | Dansk. Kodekommentarer og commits på engelsk. |

### 0.1 CSS-arkitektur — beslutning: uden Tailwind

Målt på prototypen (`assets/gaffel.css`, 2026-11-24):

| Tal | Værdi |
|---|---|
| Linjer / regelblokke | 703 / 473 |
| Navngivning | BEM hele vejen: `.nav__pill`, `.btn--ink`, `.od-load__ch` |
| Egne CSS-variabler | 57 |
| Media queries | 45 |
| Keyframes | 17 |

Det er afgørende for valget: 473 formålsbestemte regler med BEM og 17 keyframe-koreografier **kan ikke porteres** til utilities — de skal skrives om, og hver omskrivning er en lejlighed til at ramme ved siden af designet. De 45 breakpoints er redaktionelle (fx loaderens `clamp(52px, 11vw, 104px)` og navets 1560 px-kollaps), ikke de fire skærmstørrelser et utility-system arbejder i.

**Beslutning:**

- `app/globals.css` porterer prototypens CSS uændret, med `@layer tokens, base, components;` forrest og tokens i egen fil.
- Nye komponenter får **CSS Modules** (`recipe-search.module.css`), så nye navne ikke kan kollidere med de globale BEM-klasser. Ported CSS bliver global, fordi den allerede er namespaced.
- Styling af ui-sdk-dele sker på `[data-slot]` + `data-state` (afsnit 1.4), ikke via utilities.
- Moderne CSS frem for et framework: `clamp()` til typografi, `@layer` til rækkefølge, `color-mix()` til tilstande, container queries i komponenter hvor bredden afgør layoutet — ikke viewporten.

**CV-begrundelse for at vælge det fra:** Tailwind er allerede dækket af portfolio-repoet (`benjaminschou.dk`, Next 16 + TS + Tailwind). Gaffel bliver den anden, sjældnere historie: *design-tokens som eneste farvekilde, BEM + CSS Modules, moderne CSS og et designsystem porteret 1:1 fra et designtool*. To forskellige historier er mere værd end den samme to gange.

**Hvis du alligevel vil have Tailwind** (og det er et legitimt ønske): så kun som layout-lag i ny, ikke-designet krom — `/dev/ui`, indstillingssider, fejltilstande — aldrig på de skærme OpenDesign har leveret. Rækkefølgen skal da være præcis som i 1.4, fordi Tailwinds preflight ellers nulstiller ui-sdk's basals.

---

## 1. UI-laget: UI SDK — all-in

### 1.1 Hvad det er, mekanisk

`@ui-sdk/core` er en framework-fri TypeScript-model for **adfærd**. En komponent beskrives som en *primitive*:

```ts
definePrimitive({
  name: "…",
  state: () => ({ … }),          // komponentens data
  api: (context) => ({ … }),     // funktioner der ændrer data
  slots: (slot) => ({ … }),      // DOM-delene, med attrs(state) og hændelser
})
```

`attrs` mapper state → HTML-attributter (herunder ARIA). `slots` beskriver delene; en adapter gør dem til rigtige elementer (`createParts(primitive, { root: "button" })` i React). Delene er **ustylede** og eksponerer `[data-slot="…"]`, `data-state` og CSS-variabler.

Ovenpå ligger de færdige *behaviors*, som er de usynlige dele man normalt skriver forkert:

| Behavior | Løser |
|---|---|
| `createPopup` | Placering, dismiss ved Escape/klik udenfor, fokus tilbage til trigger |
| `createListbox` | `aria-activedescendant`, piletaster, Home/End |
| `createTypeahead` | Fuzzy-match med rangering + skærmlæser-announcement ("3 retter matcher") |
| `createRovingFocus` | Ét Tab-stop for en hel liste/grid |
| `createSelection` | En/flere valg, Shift/Ctrl som filbrowser |
| `createReorder` | Omarrangering med pointer **og** tastatur |
| `createViewport` | Pan/zoom (fx beskæring af kvitteringsfoto) |
| `createVirtualizer` | Render kun synlige rækker |
| `createResize` | Træk i håndtag |

Shippede komponenter ovenpå samme behaviors: `Select`, `Combobox`, `Field`, `Tabs`, `Date Picker`, `Menu`.

**Agent-broen** er en separat funktion: komponenterne kan melde navne, værdier og handlinger til en agent, som så opererer UI'et uden screenshots (`act { steps }`), med `data-agent-confirm` som samtykke-markering og filtrering af skjult indhold. Relevant i fase 2, ikke fase 1 (se 1.7).

### 1.2 Hvad "all-in" betyder — fem regler

1. **Alt interaktivt med fokus, popup, valg eller omarrangering bygges på primitives.** Knapper og links forbliver native elementer.
2. **Ingen parallel UI-libs.** Radix, base-ui, shadcn/ui, Headless UI og Reach tilføjes ikke — ui-sdk er laget, ikke et supplement til et andet lag. Tilføjer vi begge, får vi to konkurrerende fokus- og portal-modeller.
3. **Én wrapper pr. komponent**, i `components/ui/<navn>.tsx`, med en header-kommentar der nævner hvilke behaviors den bruger.
4. **Eksakt version, ingen caret.** `--save-exact`. Beta'en udgiver flere gange om ugen; `^0.1.0-beta.x` er en tidsindstillet bombe.
5. **Sider importerer aldrig `@ui-sdk/*` direkte** — kun via `components/ui/*`. Det er den regel der gør en exit til ni filer i stedet for en refaktor.

### 1.3 Install (verificeret 2026-11-24)

```bash
npm i --save-exact @ui-sdk/react@0.1.0-beta.17 @ui-sdk/core@0.1.0-beta.17
npx skills@latest add vercel-labs/ui --skill ui-sdk
```

`@ui-sdk/react` kræver `react >= 19` og `@types/react >= 19.2.10` — begge opfyldt af React 19.3.1.

Tilføj til `AGENTS.md` (så coding-agenten læser API'et i stedet for at gætte):

```md
Before using ui-sdk, follow the ui-sdk skill if it's installed, and read
node_modules/@ui-sdk/react/docs/llms.txt. To build a new component, also search
node_modules/@ui-sdk/react/docs/core.md by heading, and read rendering.md next to it.
```

### 1.4 CSS-lag

Uden Tailwind — `app/globals.css` starter med lag-rækkefølgen, så ui-sdk's base aldrig slår vores tokens:

```css
@layer ui-sdk, tokens, components;

@import "@ui-sdk/core/web.css";
@import "./tokens.css";
```

Med Tailwind (hvis det bliver aktuelt) er rækkefølgen den udgiverne kræver — Tailwinds preflight må ikke nulstille ui-sdk's basals:

```css
@layer ui-sdk;
@import "tailwindcss";
@import "@ui-sdk/core/web.css" layer(components);
```

Styling sker på `[data-slot]` + `data-state` og refererer **kun** tokens:

```css
[data-slot="combobox-trigger"] { border-bottom: 1px solid var(--line); }
[data-slot="combobox-trigger"][data-state="open"] { border-color: var(--terra); }
```

### 1.5 Klientgrænse

Primitives er stateful og kan ikke rendere i en server component. Konvention:

- Sider og layouts forbliver server components.
- Hver wrapper er `"use client"` og udgør en ø af interaktivitet.
- Data hentes i server components og sendes ind som props; wrappers laver ingen dataadgang selv.

### 1.6 Komponent-inventar: skærm → behaviors

| Gaffel-komponent | Bygges som | Behaviors | Fil |
|---|---|---|---|
| Opskriftsøgning ("burg" → burger) | `Combobox` (shipped) | popup + listbox + typeahead + selection | `components/ui/recipe-search.tsx` |
| Filterchips (Alle/Vegetar/…) | Toggle-primitiv, én valgt ad gangen | `createSelection` + `createRovingFocus` | `components/ui/chip-group.tsx` |
| Dag-vælger (`askDay`) | Dialog | `createPopup` (fokusfælde, Escape, retur til trigger) | `components/ui/day-picker.tsx` |
| Butiksskifter (Lidl/REMA/Netto) | `Select` eller `Menu` | popup + listbox/roving focus | `components/ui/store-picker.tsx` |
| Uge-skifter (‹ 45 ›) | To knapper + live tal | NumberFlow til tallet (se 2.5) | `components/week-switcher.tsx` |
| Nav-overløb på desktop | `Menu` | popup + roving focus | `components/ui/nav-overflow.tsx` |
| "Har hjemme"-lager | Multi-select-liste | `createSelection` | `components/ui/pantry-list.tsx` |
| Fire aftener / uge-bræt | Reorder-liste | `createReorder` | `components/ui/week-board.tsx` |
| Bon-upload + beskæring | Dropzone + viewport | `createViewport` | `components/ui/receipt-crop.tsx` |
| Toast | Sonner (2.0.8) | — (egen lib, ikke ui-sdk) | `components/ui/toast.tsx` |

Chips-rækken er ikke kosmetisk: i prototypen kunne `?protein=ingen` sætte **to** chips til `aria-pressed="true"` samtidig. `createSelection` i single-mode gør den tilstand umulig.

### 1.7 Agent-broen — fase 2, ikke fase 1

Slås først til når Copiloten faktisk bliver en agent (BYOK-model der skal *udføre* noget i UI'et). Indtil da er `data-agent-*` kun ekstra DOM-overflade og et sikkerhedsspørgsmål uden en modpart.

Når den slås til: `agent.tools` eksponeres bag `data-agent-confirm` på alt der skriver (lås uge, slet ret, køb), og `data-agent-hidden` på persondata. WebMCP er den browser-inbyggede variant af samme bro — samme beslutningspunkt.

### 1.8 Exit-plan (hvis beta'en brækker)

- **Berøringspunkt:** kun `components/ui/*`. Domænekode (`lib/`, `app/`) må ikke kende ui-sdk's typer.
- **Erstatninger:** base-ui (popup/listbox/selection), dnd-kit (`createReorder`), TanStack Virtual (`createVirtualizer`), `cmdk` (⌘K).
- **Trigger:** to brydende ændringer inden for en måned, eller en a11y-fejl vi ikke selv kan patche.
- **Kostnad ved exit:** ni små filer, ikke en refaktor. Det er hele grunden til regel 5 i 1.2.

### 1.9 Sådan ved vi at det virker (uden nyt framework)

Pr. wrapper, manuelt, før den regnes som færdig:

1. Tab igennem: komponenten fylder **ét** Tab-stop (ikke ét pr. element).
2. Piletaster flytter markering; Home/End virker.
3. Skriv 2–3 bogstaver: markeringen lander på bedste match, Enter vælger.
4. Escape lukker og **fokus er tilbage på triggeren**.
5. Klik udenfor lukker uden at fokus forsvinder.
6. Skærmlæser (VoiceOver) læser rolle, navn, tilstand — og antallet af match ved søgning.
7. `aria-pressed`/`aria-selected` kan ikke stå på to elementer samtidig i single-mode.
8. Reduced motion: ingen bevægelse, stadig funktionel.

Plus screenshot-baseline pr. skærm. Det er gate'en — ikke et testframework.

### 1.10 CV-referencen

Det her er en af de få ting i projektet der er **reelt differentierende** i en ansøgning, fordi biblioteket er nyt (første beta i november 2026) og fordi det kræver en forståelse for *hvorfor* fokus, ARIA og tastatur er svært — ikke bare at man kan kalde en komponent.

**Dansk — brugbar formulering:**

> Byggede UI-laget på UI SDK (`@ui-sdk/core`), et behaviors-bibliotek fra vercel-labs, hvor komponenter sammensættes af primitiver som `createPopup`, `createListbox`, `createTypeahead` og `createReorder`. Alle komponenter er ustylede og stylet udelukkende via design-tokens, så et eget designsystem kunne bevares 1:1. Fokus-styring, tastaturnavigation, ARIA-relationer og typeahead er dækket af biblioteket frem for håndskrevet kode.

**Engelsk:**

> Built the UI layer on UI SDK (`@ui-sdk/core`), a behaviors library from vercel-labs: components composed from primitives such as `createPopup`, `createListbox`, `createTypeahead` and `createReorder`. All parts are unstyled and styled exclusively through design tokens, preserving a custom design system 1:1. Focus management, keyboard navigation, ARIA relationships and typeahead behavior come from the library rather than hand-written code.

**Sådan taler du om det i en samtale** (det er her det bliver troværdigt):

- *Hvorfor:* "Interaktioner er den del af en UI man ikke ser, når den er rigtig. Jeg valgte et behaviors-lag frem for et komponentbibliotek, fordi designet var fastlagt først — jeg skulle bruge adfærd uden at få et look med."
- *Hvad der var svært:* "Typeahead plus listbox plus popup oveni hinanden: fokus skal både blive i inputfeltet, flytte markering via `aria-activedescendant` og tilbage til triggeren ved Escape. Det er tre forskellige fokusmodeller i samme komponent."
- *Hvad du ikke må påstå:* at biblioteket er stabilt (det er beta), eller at du har bygget behaviors selv. Sig at du sammensatte dem og styrede dem bag wrappers.

**Det du kan dokumentere i repoet** (det er CV'ets bevis): `components/ui/*` med behaviors nævnt i header, `tokens.css` som eneste farvekilde, a11y-tjeklisten i 1.9 og screenshot-baselines pr. skærm.

---

## 2. Andet guld — verificeret 2026-11-24

Kriteriet for at komme på listen: det løser et behov Gaffel faktisk har, det er nutidigt nok til at være værd at kunne tale om, og det har en klart defineret grænse.

| # | Værktøj | Version | Behov i Gaffel |
|---|---|---|---|
| 2.1 | Vercel AI SDK (`ai`, `@ai-sdk/react`, `@ai-sdk/openai-compatible`) | 7.0.181 / 4.0.186 / 3.0.94 | Copilot med BYOK, streaming |
| 2.2 | Serwist (`@serwist/next`) | 9.5.15 | PWA + offline indkøbsliste |
| 2.3 | Drizzle ORM + better-sqlite3 | 0.45.4 / 13.2.0 | Typet lokal SQLite + migrations |
| 2.4 | React `<ViewTransition>` (i React 19.3 / Next 16.4) | indbygget | Side- og kort-overgange |
| 2.5 | NumberFlow (`@number-flow/react`) | 0.6.2 | Levende priser (198,35 → 234,80) |
| 2.6 | `skills` CLI | 1.9.0 | Repoet leverer selv agent-skills |
| 2.7 | Server Actions + `useOptimistic` | indbygget | "Lås uge" uden ventetid |

### 2.1 AI SDK — kerne fra start

Copilot er en hovedfunktion i Gaffel, ikke et tilvalg: den skal læse ugen, foreslå retter ud fra tilbudsaviser og husstandens præferencer, og svare på "hvad kan jeg lave i denne uge?". Derfor er AI SDK med fra første commit — ikke tilføjet senere som pynt.

Besluttede versioner: `ai` 7.0.181, `@ai-sdk/react` 4.0.186, `@ai-sdk/openai-compatible` 3.0.94.

**Nous Portal som udbyder — verificeret 2026-11-24** i `~/.hermes/config.yaml`:

```
provider: nous
base_url: https://inference-api.nousresearch.com/v1
```

Den bruges som standardudbyder gennem `createOpenAICompatible({ baseURL })`. Dermed er samme kodevej også vejen til alle andre OpenAI-kompatible endpoints, og BYOK bliver et spørgsmål om konfiguration, ikke om kode.

**Arkitektur:**

- Copilot-samtalen kører i en route handler (`app/api/copilot/route.ts`) og streames til `useChat` i klienten. Nøgler forlader aldrig serveren.
- Udbyder og model vælges i en lille factory (`lib/ai/provider.ts`) ud fra husstandens indstillinger; ukendt udbyder = fejl, ikke stille fallback.
- Modellen får **læseværktøjer** over den lokale database (ugens retter, priser, tilbud, lager) — ikke skriveadgang. Alt der ændrer ugen kræver et klik fra brugeren.
- Prompten bærer husstandens kontekst (antal voksne/børn, butikker, "har hjemme", historik) som struktureret data, ikke som prosa der skal parses tilbage.

**Grænse:** ingen model i prisberegning, uge-validering eller bon-afstemning. Modellen må *foreslå*; aritmetikken er deterministisk kode, så tallene altid kan efterprøves. Nye nøgler lander i `.env.local` **og** Vercel-prod — aldrig kun det ene sted.

**AI SDK 7 er ikke kun `useChat`.** Versionen (juni 2026, nu 7.0.181) har tre ting der er værd at kende for Gaffel:

- **`WorkflowAgent` + `@ai-sdk/workflow`:** varig, genoptagelig agent-kørsel der overlever genstarter og afbrydelser, med indbyggede godkendelser. Det er præcis formen "planlæg ugen i tre trin → brugeren godkender → ugen låses" har, hvis Copilot senere skal gøre mere end at svare.
- **Typet tool-kontekst:** værktøjerne får delt, typet kontekst i stedet for at grave i global state — mindre overraskelse i et system hvor modellen kun må læse.
- **Telemetri:** indbygget sporing pr. kald, så token-forbrug og fejl kan måles i stedet for gættes. Relevant for BYOK-brugere der betaler per token.

Begynd med `useChat` + route handler; ryk til `WorkflowAgent` når Copilot får flertrinshandlinger med godkendelse.

**CV:** "streaming Copilot med multi-provider BYOK via OpenAI-compatible endpoints, værktøjskald mod lokal SQLite og nøgler der aldrig forlader serveren." AI SDK er branchens standard for streaming-chat i React, så det er både et ærligt og et genkendeligt punkt.

### 2.2 Serwist — PWA der virker i butikken

Indkøbslisten bruges i en SuperBrugsen-kælder med dårlig dækning. Serwist er den vedligeholdte afløser for `next-pwa` (som er forladt), med App Router-understøttelse og precache af app-skal.

**Grænse:** caches kun app-skal og statiske data. Priser og tilbud må aldrig serveres fra cache uden tidsstempel — en forkert pris i butikken er værre end ingen pris.

**CV:** "offline-first PWA med service worker-precaching og eksplicit cache-strategi pr. datatype" — det er et spørgsmål der kommer i næsten hver frontend-samtale.

### 2.3 Drizzle + better-sqlite3

Lokal SQLite er allerede besluttet. Drizzle giver typet schema, genererede migrations og SQL man kan læse — modsat en ORM der skjuler hvad der sker.

**Grænse:** ingen databaseadgang i client components; kun via server actions/route handlers. `better-sqlite3` er synkron og egner sig ikke til at blokere request-tråden med tunge joins — hold forespørgslerne små.

**CV:** "typet SQL-lag med migrations" slår "brugte en ORM".

### 2.4 React View Transitions

Ifølge Next.js-dokumentationen (opdateret 18. november 2026) virker view transitions i App Router **uden konfiguration**, og React 19.3 indeholder `<ViewTransition>` og `addTransitionType`. Det er præcis den slags overgang Gaffel mangler: opskriftskort → opskriftsdetalje som morph, uge-skift som retningsbestemt slide.

**Grænse:** kun hvor det kommunikerer noget (samme element, gå dybere; frem/tilbage). Ikke som pynt på hvert klik, og altid med reduced-motion-fallback. Browserstøtte varierer — uden støtte skal appen bare virke uden animation.

**CV:** en API der er måneder gammel og som de fleste ansøgere ikke har rørt. Der findes en agent-skill til den: `npx skills add vercel-labs/agent-skills --skill vercel-react-view-transitions`.

### 2.5 NumberFlow — levende priser

Butiksskift ændrer 234,80 → 198,35. NumberFlow animerer cifrene korrekt (inkl. `tabular-nums` og locale), hvilket er mærkbart bedre end at re-rendere teksten.

**Grænse:** kun tal der faktisk ændrer sig. Reduced motion respekteres.

**CV:** lille, men et konkret eksempel på "animation med et formål" frem for animation som dekoration — det er præcis den sondring design-interviews tester.

### 2.6 `skills` CLI — repoet lærer agenten sin egen stack

`npx skills add …` installerer skills som ui-sdk's og view-transitions ind i repoet, så coding-agenten læser den korrekte API-dokumentation frem for at gætte. Kombineret med `AGENTS.md`-noten i 1.3 er det en reproducerbar opskrift: klon repoet, agenten ved hvordan stacken bruges.

**CV:** "agent-ready repository" — skills, AGENTS.md, og en QA-gate der kan køres af en agent. Det er et 2026-kompetencesignal, ikke et 2020-et.

### 2.7 Server Actions + `useOptimistic`

"Lås uge" og "tilføj til uge" skal føles øjeblikkelige. Optimistisk UI med server action som sandhed, og rollback ved fejl. Ingen bibliotek — men et mønster der er værd at gøre rigtigt og omtale.

### 2.8 Kvitteringsparseren — danske kæder, ikke én universel regex

Gaffel kører **kun danske butikker** i v1. Det gør omfanget håndterbart, men det gør ikke én parser mulig: hver kæde har egen kolonneorden, egne rabatlinjer (`Rabat`, `Tilbud`, `-`), egne momsmarkeringer og egen footer. En universal-regex der "læser danske bonner" vil fejle stille på den tredje kæde.

**Arkitektur:**

- **Én konfiguration pr. kæde** (`lib/receipt/chains/netto.ts`, `rema1000.ts`, `lidl.ts`, …): layout-hints, kolonneorden, hvordan rabatlinjer ser ud, hvordan totalen markeres, og et lille sæt fixtures (rigtige boner, anonymiserede).
- **Fælles motor** der bruger konfigurationen; ny kæde = ny fil + fixtures, ikke ændret motor.
- **Aritmetisk gate:** `Varer − Rabatter = SUM` skal stemme på øre. Gør det ikke, afvises resultatet og brugeren får manuel indtastning — den regel er hele grunden til at feature'en kan bruges i virkeligheden. Prototypen har allerede tallene der skal reproducere: 176,45 + 70,55 = 247,00.
- **Usikkerhed vises:** linjer under en konfidenstærskel markeres til gennemsyn, ikke skjules.

**Kæder i v1:**

| Niveau | Kæder |
|---|---|
| Primære (fixtures + tests) | Netto, REMA 1000, Lidl, Føtex, Bilka, 365discount |
| Sekundære (konfiguration, lettere test) | SuperBrugsen, Kvickly, Brugsen/Dagli'Brugsen, MENY, SPAR, Min Købmand, ALDI |
| Long tail | Løvbjerg, ABC Lavpris, Fleggaard, Calle, Otto Duborg, Poetzsch — via generisk konfiguration, tilføjes når en rigtig bon foreligger |

**Privatliv:** boner indeholder butik, tidspunkt og hvad husstanden har købt. Alt bliver lokalt i SQLite; ingen bon-billeder eller linjer sendes til en model medmindre brugeren aktivt vælger det, og aldrig til et endpoint der logger dem.

---

## 3. Bevidst fravalgt

| Fravalgt | Hvorfor |
|---|---|
| TanStack Query | Server components + lokal SQLite gør den overflødig; den ville kun tilføje en cache ved siden af databasen |
| Tesseract.js (7.0.0, sidst opdateret dec. 2025) | Kvitterings-OCR på termopapir er svært, og pakken er et år gammel. Mål først: manuel indtastning + parser til `Varer − Rabatter = SUM`, og kun hvis det ikke er nok, en vision-model via AI SDK |
| shadcn/ui, Radix, base-ui | Konkurrerer med ui-sdk om samme lag. Vælges kun som exit (1.8) |
| Tailwind i første iteration | Se rammerne i afsnit 0 |
| Storybook | Endnu en byggeflade; wrappers + en `/dev/ui`-rute dækker behovet |
| WebMCP nu | Kræver en agent som modpart (1.7) |
| Nye test-frameworks | Gate er build + typecheck + a11y-tjekliste. Playwright kan komme senere, når flows er stabile |

---

## 4. Rækkefølge ved kodestart

1. `npx create-next-app@latest . --yes` (Next 16, React 19, TS, App Router).
2. Portér `assets/gaffel.css` → `app/globals.css` og udtræk `design/tokens.css` fra prototypens `:root`. **Gate:** ingen hex i komponenter, kun `var(--…)`.
3. Sæt lag-rækkefølgen (1.4) og verificér at ui-sdk's CSS ikke slår tokens.
4. Installér ui-sdk pinnet + skills (1.3). **Gate:** `next build` grøn.
5. Byg **én** wrapper: `recipe-search.tsx` (Combobox). Kør a11y-tjeklisten (1.9). **Gate:** alle 8 punkter.
6. Byg `chip-group.tsx` og `day-picker.tsx` samme vej. **Gate:** samme liste + screenshot-baseline.
7. Drizzle + schema, derefter **AI SDK som kerne** (provider-factory, `/api/copilot`, `useChat`), og til sidst Serwist. Copilot er en hovedfunktion: den skal stå færdig sammen med uge-brættet, ikke bagefter.

Hvis trin 5 ikke kan bestå tjeklisten, er det signalet om at bruge base-ui i stedet — og det er billigt at opdage på det tidspunkt, fordi der kun findes én wrapper.

---

## 5. Afgjort, og hvad der stadig er åbent

**Afgjort i denne runde:**

1. **CSS:** uden Tailwind — håndskrevet CSS med tokens + CSS Modules (afsnit 0.1).
2. **Nous Portal-baseURL:** `https://inference-api.nousresearch.com/v1`, verificeret i `~/.hermes/config.yaml` 2026-11-24 (afsnit 2.1).
3. **Butikker:** kun Danmark i v1; primære kæder med fixtures, sekundære med konfiguration, long tail når en rigtig bon foreligger (afsnit 2.8).

**Stadig åbent — og det skal afgøres før Trin 7, ikke efter:**

1. ~~Hosting af den lokale database~~ **Afgjort 2026-11-25:** vi vælger (a) — `better-sqlite3` og selvhost til udvikling og personlig brug. Turso/libSQL eller ren browserlagring bliver først aktuel den dag Gaffel skal deles med andre; Drizzle-skiftet er lille nok til at vente til da. Ingen deploy-arkitektur bygges på spekulation.
2. **Første rigtige bon-fixture.** Parserens kvalitet afhænger af rigtige boner, ikke opdigtede. Én anonymiseret bon fra Netto, REMA 1000 og Lidl er nok til at starte.
3. **Standardmodel for Copilot.** Vælges når provider-factory'en skrives; den skal ikke pinnes i dokumentationen, men i konfiguration — samme regel som for Hermes' egne modeller.

---

## 6. Repoet, offentligheden og agent-harnesset

### 6.1 Hvad der bliver offentligt

Repoet er tænkt offentligt, og **alt i git er læsbart for alle**. Det betyder:

| Fil | Offentlig? | Begrundelse |
|---|---|---|
| `docs/kodefase-beslutninger.md` (dette dokument) | **Ja** | Det er bevis på beslutningstagning — målinger, afvejninger, hvad der blev valgt fra og hvorfor. Det er den slags fil der gør et hobbyrepo troværdigt. |
| Afsnit 1.10 (CV-formuleringer og interview-svar) | **Nej** | Det er ansøgningsstrategi, ikke projektviden. Flyt til det private jobstack-repo (`~/code/ai-job-search`) eller en gitignored `notes/`. Resten af dokumentet bliver i repoet. |
| Bon-fixtures | **Kun anonymiserede** | Butik, dato og hvad husstanden har købt er persondata. Beløb og linjestruktur er nok til at teste parseren. |
| `.env.local`, nøgler | **Nej, aldrig** | Gælder også eksempel-filer: `.env.example` med tomme værdier, intet andet. |

Reglen er den samme som for de andre offentlige repoer: kode, beslutninger og anonymiserede fixtures ud; ansøgningsmateriale, persondata og nøgler ind.

### 6.2 Skills — i repoet og i Hermes

**I repoet** (committes, så alle agenter får samme kontekst):

```bash
npx skills add vercel-labs/ui --skill ui-sdk
npx skills add vercel-labs/agent-skills --skill vercel-react-view-transitions
```

`SKILL.md`-formatet blev open standard i slutningen af 2025 og er pr. marts 2026 adopteret af 32 værktøjer (Claude Code, Codex, Cursor, Copilot, Gemini CLI, Goose, Amp …). At committe skills betyder at OpenCode, Codex og en fremtidig agent læser den samme, korrekte API-dokumentation i stedet for at gætte.

**I Hermes** mangler der én skill: `gaffel-dev`. Den skal indeholde det der ellers skal genlæses hver session:

- Gates: `next build` + `tsc --noEmit` + a11y-tjeklisten (1.9) + screenshot-baseline.
- `tokens.css` som eneste farvekilde; ingen hex i komponenter.
- Wrapper-konventionen (1.2 regel 3 og 5) og hvad der må importeres hvor.
- Bon-invarianten `Varer − Rabatter = SUM` som hård gate.
- At ui-sdk er pinnet og hvor exit-planen står.

### 6.3 Agent-harnesset omkring projektet

2026-litteraturen er entydig på ét punkt: **en agent er model + harness**, og kvaliteten afgøres af harnesset — gates, kontekst, tilladelser, review — ikke af modellen alene. Gaffels harness er `/oc-issue`-pipelinen: brief med verbatim regler, kodeagent, uafhængig QA-model, grøn gate, derefter merge.

To ting der er værd at tilføje i praksis:

1. **En lille eval-suite af rigtige briefs.** Ikke 100 syntetiske tests, men 10–15 opgaver hentet fra det der faktisk går galt (fx "tilføj kategori-filter til opskrifter", "parse en Netto-bon", "ret fokus i day-picker"). Målet er at se om coder + QA leverer en grøn gate uden menneskelig indblanding — og at opdage når et model- eller harness-skift gør det dårligere. Sammenlign kun inden for samme harness-version; ellers måler man to ting på én gang.
2. **Gate proportional med blast radius.** Ren CSS og tekst: build + typecheck. Nyt library, schema eller auth: HermaGuard + en gennemlæsning af diffen før merge. Det er allerede reglen i den eksisterende pipeline — den skal bare følge med over i Gaffel.

### 6.4 WebMCP — hold øje, byg ikke nu

Chrome og Edge kører WebMCP som origin trial (Chrome 146+, Edge 153), og kun **Tools** er understøttet — ikke resources eller prompts. ui-sdk understøtter allerede broen, så den dag Chrome ruller det ud, er Gaffel klar hvis agent-broen (1.7) er slået til. Indtil da: ingen kode, ingen `data-agent-*` i produktion.
