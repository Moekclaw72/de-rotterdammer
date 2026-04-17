# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## CURRENT CODEBASE STATE

The project currently contains one deliverable: `rotterdammer-v4.html` — a fully self-contained single-file PWA for Brasserie De Rotterdammer.

**No build process.** Open the HTML file directly in a browser or serve it via any static file server (e.g. `python -m http.server 8080`). There are no dependencies, npm packages, or compilation steps.

### Configuration (top of script block, ~line 452)
Four constants must be filled in before deployment:
```js
const IBAN        = 'NL00INGB0000000000'; // ← klant IBAN
const WERO_PHONE  = '+31600000000';        // ← klant Wero telefoonnummer
const TIKKIE_URL  = 'https://tikkie.me/pay/JOUWLINK'; // ← klant Tikkie link
const GEMINI_API_KEY = ''; // ← gratis key van aistudio.google.com/apikey
```

### Architecture of rotterdammer-v4.html

**Menu data** lives in the `DATA` object (line ~465) — a plain JS object keyed by category (`uitsmijter`, `brood`, `hoofd`, `fris`, `bier`, etc.). Each item: `{n: naam, p: prijs, d: omschrijving}`. Price `0` renders as "Op aanvraag" and disables add-to-cart.

**Rendering** — `buildGrid(id, items)` dynamically creates `.item-card` elements into `<div id="grid-{id}">` placeholders in the HTML. Called once at load for every key in `DATA`.

**State** — `cart[]` array in memory. `orders[]` persisted in `localStorage` key `rdm2_orders`. No backend; no server. Orders survive page refresh but are device-local.

**Admin panel** — click the ⚙ button (bottom-right). No password. Shows live order queue with status buttons (Wacht / Klaar / Uitgeleverd). Polls `localStorage` every 3 seconds to detect new orders from other browser tabs/devices on the same device.

**Chatbot** — Gemini 2.5 Flash called directly from the browser (`fetch` to `generativelanguage.googleapis.com`). The API key is visible in page source. Acceptable for MVP; not suitable for public production without a backend proxy. Builds `menuContext` from `DATA` at query time.

**PWA** — Service worker and manifest are injected inline at runtime via `Blob` URLs. The app installs as a home screen app on mobile.

### Adding a new menu item
Add an entry to the relevant array in `DATA`. If a new category is needed, also add a `<div id="grid-{key}">` placeholder in the HTML and a label in `buildMenuContext()`.

### Adapting this for a new client
1. Replace all occurrences of `De Rotterdammer` / `Brasserie` / `NAAM`
2. Update `DATA` with the new menu
3. Fill in the four config constants
4. Adjust CSS variables in `:root` for brand colours if needed

---

# MKB AGENT BUSINESS - CLAUDE.MD

**Project:** Automated agent services voor kleine bedrijven (eetcafés, kappers, rijscholen, etc.)
**Eigenaar:** Moek (Mukesh Jagroep)
**Doel:** Schaalbaar verdienmodel via gepersonaliseerde AI-agents voor Nederlands MKB

---

## HOOFDARCHITECTUUR

Dit project bestaat uit **3 lagen**:

1. **Template Layer** - Herbruikbare basis-agents die je voor elke klant customized
2. **Client Layer** - Per-klant configuratie en data
3. **Outreach Layer** - Automatische acquisitie en lead management

---

## AGENT 1: ORDER AGENT
**Verantwoordelijkheid:** Bestellingen afhandelen voor eetcafés, restaurants, cafés

### Skills
- WhatsApp/Telegram integratie via officiële API's
- Menu parsing en beschikbaarheid check
- Bestelling opnemen met validatie
- Allergie-checks en aanpassingen verwerken
- Betalingsopties aanbieden (pin, iDEAL, contant)
- Order doorsturen naar keuken/bar (via webhook of notificatie)
- Multiple talen ondersteunen (NL, EN, DE, PL - belangrijk voor truckers)

### Tech Stack
- **AI Model:** Gemini 2.5 Flash (gratis tier: 250 req/dag, €0,10/M tokens betaald)
- **Messaging:** Meta WhatsApp Business API of Twilio
- **Database:** SQLite lokaal of Supabase voor multi-client
- **Deployment:** Vercel/Railway/Render (gratis tier mogelijk)

### Input/Output
**Input:** WhatsApp bericht van klant met vraag/bestelling
**Output:** Bevestigde bestelling + orderID + betalingsinstructies + ETA

### Template Variabelen (per klant)
```javascript
{
  "business_name": "De Rotterdammer",
  "business_type": "eetcafé",
  "menu": [/* menu items als JSON */],
  "payment_methods": ["pin", "contant", "ideal"],
  "languages": ["nl", "en", "de", "pl"],
  "whatsapp_number": "+31612345678",
  "notification_webhook": "https://..."
}
```

### Prompting Strategy
**System prompt template:**
```
Je bent een vriendelijke order-assistent voor {business_name}, een {business_type} in Nederland.

REGELS:
- Antwoord kort en duidelijk (max 3 zinnen)
- Antwoord in de taal van de klant
- Bevestig altijd de bestelling met items + totaalprijs
- Bij onduidelijkheid: stel 1 verduidelijkende vraag
- Bij allergieën: waarschuw én adviseer alternatief

MENU:
{menu_context}

BETALINGSMETHODEN:
{payment_methods}

Neem alleen bestellingen op die op het menu staan.
```

---

## AGENT 2: APPOINTMENT AGENT
**Verantwoordelijkheid:** Afspraken inplannen voor kappers, nagelstudio's, rijscholen, tandartsassistentes

### Skills
- Agenda-integratie (Google Calendar, iCal)
- Beschikbaarheid checken real-time
- Afspraak boeken met bevestiging
- Herinneringen sturen (24u + 2u van tevoren)
- No-show detectie en follow-up
- Verzetten/annuleren afhandelen

### Tech Stack
- **AI Model:** Gemini 2.5 Flash
- **Calendar:** Google Calendar API (gratis)
- **Scheduling:** Calendly webhooks of eigen logica
- **Messaging:** WhatsApp Business API

### Input/Output
**Input:** "Kan ik morgen om 14:00 terecht?" of "Ik wil mijn afspraak verzetten"
**Output:** Bevestigde afspraak + Google Calendar invite + herinneringen

### Template Variabelen
```javascript
{
  "business_name": "Salon Mooi",
  "services": [
    {"name": "Knippen", "duration": 30, "price": 25},
    {"name": "Kleuren", "duration": 90, "price": 75}
  ],
  "opening_hours": {
    "monday": ["09:00-17:00"],
    "tuesday": ["09:00-17:00"]
  },
  "calendar_id": "salon@example.com",
  "reminder_times": [1440, 120] // minuten voor afspraak
}
```

---

## AGENT 3: FAQ AGENT
**Verantwoordelijkheid:** Veelgestelde vragen beantwoorden voor elke branche

### Skills
- Custom knowledge base per klant
- Openingstijden, prijzen, locatie, parkeren
- Product/dienst uitleg
- Doorverwijzen naar mens bij complexe vragen
- Multi-channel (WhatsApp, website widget, Facebook Messenger)

### Tech Stack
- **AI Model:** Gemini 2.5 Flash
- **Knowledge Base:** Markdown files of JSON in repo
- **Embedding:** Niet nodig voor <50 FAQ's, wel voor >100
- **Deployment:** Cloudflare Workers (gratis) of Vercel

### Template Variabelen
```javascript
{
  "business_name": "Rijschool Safe Drive",
  "faqs": [
    {
      "q": "Wat zijn de kosten?",
      "a": "Een lesuur kost €45. Pakket van 10 lessen: €400."
    },
    {
      "q": "Waar parkeren?",
      "a": "Gratis parkeren achter het pand, ingang via steeg."
    }
  ],
  "escalation_contact": "+31612345678",
  "business_hours": "ma-vr 9-17u"
}
```

---

## AGENT 4: REVIEW AGENT
**Verantwoordelijkheid:** Google/Facebook reviews beheren en beantwoorden

### Skills
- Review monitoring via Google My Business API
- Automatisch bedankbericht op positieve reviews
- Voorgesteld antwoord op negatieve reviews (met menselijke check)
- Review-verzoek sturen na afspraak/bestelling
- Sentiment analyse en alerts bij negatieve trend

### Tech Stack
- **AI Model:** Gemini 2.5 Flash
- **APIs:** Google My Business API, Facebook Graph API
- **Scheduling:** Cron job via GitHub Actions (gratis)

### Template Variabelen
```javascript
{
  "business_name": "Kapsalon Chique",
  "google_place_id": "ChIJ...",
  "facebook_page_id": "123456789",
  "auto_respond": true,
  "manual_approval_threshold": 3, // sterren of lager
  "request_review_after_hours": 24
}
```

---

## AGENT 5: OUTREACH AGENT (voor jouw acquisitie)
**Verantwoordelijkheid:** Potentiële klanten vinden en benaderen

### Skills
- Scrapen van lokale bedrijven (Google Maps, Yelp)
- Filteren op doelgroep (eetcafés, kappers, etc.)
- Gepersonaliseerde cold outreach via email/LinkedIn
- Follow-up sequence (3x, dan stop)
- Lead tracking en CRM integratie

### Tech Stack
- **Scraping:** Bright Data, ScraperAPI, of Apify (betaald maar goedkoop)
- **Email:** Loops.so (gratis tot 2000/maand) of Resend
- **CRM:** Notion database of Google Sheets via API
- **AI:** Claude 3.5 Haiku voor personalisatie (goedkoop)

### Workflow
1. **Input:** Stad + branche (bijv. "Rotterdam eetcafés")
2. **Scrape:** Haal lijst met naam, adres, telefoon, website
3. **Filter:** Alleen bedrijven zonder online bestelsysteem
4. **Personaliseer:** Maak email per bedrijf met naam + specifieke pijnpunt
5. **Send:** Stuur via Loops/Resend
6. **Track:** Log respons in Notion/Sheets
7. **Follow-up:** Na 3 dagen + 7 dagen als geen reactie

### Email Template (personalisatie via AI)
```
Onderwerp: Klanten bestellen laten plaatsen terwijl je slaapt? ⚡

Hoi {owner_name},

Ik zag dat {business_name} nog geen online bestelsysteem heeft.
Dat betekent dat je elke bestelling handmatig moet afhandelen — en dat kost tijd.

Ik bouw WhatsApp-bots die dat automatisch doen voor €40/maand.
{specific_benefit_for_industry}

Interesse in een gratis demo?

Groet,
Moek
```

---

## AGENT 6: BROADCAST AGENT
**Verantwoordelijkheid:** Marketing broadcasts naar klantenlijst

### Skills
- Dagschotel/aanbieding versturen naar WhatsApp lijst
- Segmentatie (vaste klanten, nieuwe klanten, inactieven)
- A/B testing van berichten
- Opt-out management (GDPR compliant)
- Analytics: open rate, click rate, conversie

### Tech Stack
- **Messaging:** WhatsApp Business API (Meta) of Twilio
- **Scheduling:** Cron + database
- **Segmentation:** Tags in database per contact

### Template Variabelen
```javascript
{
  "business_name": "De Rotterdammer",
  "segments": {
    "vip": ["+316...", "+316..."],
    "regular": [...],
    "inactive": [...]
  },
  "broadcast_schedule": {
    "dagschotel": "daily 10:00",
    "weekend_special": "friday 15:00"
  },
  "opt_out_keyword": "STOP"
}
```

---

## TECHNISCHE ARCHITECTUUR

### File Structure (per project in Claude Code)
```
mkb-agents/
├── CLAUDE.md                 # Dit bestand
├── agents/
│   ├── order-agent/
│   │   ├── config.template.json
│   │   ├── prompt.template.txt
│   │   ├── main.py
│   │   └── test.py
│   ├── appointment-agent/
│   ├── faq-agent/
│   ├── review-agent/
│   ├── outreach-agent/
│   └── broadcast-agent/
├── clients/
│   ├── de-rotterdammer/
│   │   ├── config.json        # Klant-specifieke data
│   │   ├── menu.json
│   │   └── whatsapp_webhook.py
│   ├── salon-mooi/
│   └── rijschool-safe/
├── shared/
│   ├── gemini_client.py       # Herbruikbare Gemini wrapper
│   ├── whatsapp_client.py     # WhatsApp API wrapper
│   └── db.py                  # Database helpers
└── scripts/
    ├── deploy.sh
    └── test_agent.sh
```

### Deployment Strategy
**Development:**
- Claude Code lokaal voor bouwen/testen
- ngrok voor WhatsApp webhook testing

**Production:**
- Railway.app (gratis tier + €5/maand per actieve agent)
- Of Vercel/Render voor stateless agents
- Supabase gratis tier voor database (500MB)

### Kosten Breakdown (per klant/maand)
```
Gemini API: €0 (binnen gratis tier van 250 req/dag)
              of €2-5 bij heavy usage
WhatsApp API: €0,005/bericht (100 berichten = €0,50)
Hosting: €0 (Railway gratis) of €5 (Railway paid)
Database: €0 (Supabase gratis tier)

TOTAAL: €0-10/maand kosten
PRIJS VOOR KLANT: €40-80/maand
MARGE: €30-80/maand per klant
```

---

## IMPLEMENTATIE ROADMAP

### Fase 1: MVP (Week 1-2)
- [ ] Order Agent bouwen met Gemini
- [ ] WhatsApp integratie testen (sandbox)
- [ ] Deploy voor De Rotterdammer (je vriend)
- [ ] 1 week live testen en feedback

### Fase 2: Template (Week 3)
- [ ] Order Agent omzetten naar template
- [ ] Config-driven menu systeem
- [ ] Deployment script maken
- [ ] Documentatie schrijven

### Fase 3: Uitbreiden (Week 4-6)
- [ ] Appointment Agent bouwen
- [ ] FAQ Agent bouwen
- [ ] 2e klant onboarden (test template)

### Fase 4: Acquisitie (Week 7-8)
- [ ] Outreach Agent bouwen
- [ ] 50 leads scrapen Rotterdam
- [ ] Eerste 5 deals sluiten

### Fase 5: Schalen (Week 9-12)
- [ ] Review Agent + Broadcast Agent
- [ ] 10 actieve klanten
- [ ] €400-800/maand recurring

---

## CHECKLIST VOOR NIEUWE KLANT ONBOARDING

**Stap 1: Discovery**
- [ ] Bedrijfstype: _______
- [ ] Welke agent(s) nodig: Order / Appointment / FAQ / Review / Broadcast
- [ ] WhatsApp nummer beschikbaar? Ja / Nee
- [ ] Website/menu beschikbaar? Ja / Nee

**Stap 2: Setup (2-4 uur werk)**
- [ ] Config.json aanmaken met bedrijfsdata
- [ ] Menu/diensten importeren
- [ ] WhatsApp nummer koppelen aan Meta Business
- [ ] Webhook deployen op Railway/Vercel
- [ ] Testen met 10 test-berichten

**Stap 3: Training**
- [ ] Demo sessie met klant (30 min)
- [ ] Admin panel uitleg (5 min)
- [ ] Eerste week monitoren

**Stap 4: Go-Live**
- [ ] QR-code printen voor in zaak
- [ ] Social media post klaarmaken
- [ ] Monitoring dashboard openen
- [ ] Factuur sturen (€X eenmalig + €Y/maand)

---

## SECURITY & COMPLIANCE

### GDPR (belangrijk voor NL markt)
- [ ] Opt-in vereist voor WhatsApp broadcasts
- [ ] Privacy policy link in eerste bericht
- [ ] Data bewaren max 2 jaar
- [ ] Recht op verwijdering implementeren (STOP keyword)
- [ ] Geen gevoelige data opslaan (alleen naam + nummer + orderhistorie)

### API Keys & Secrets
**Nooit in code:**
- Gemini API key
- WhatsApp API tokens
- Database credentials

**Opslag via environment variables:**
```bash
# .env.example
GEMINI_API_KEY=your_key_here
WHATSAPP_TOKEN=your_token_here
DATABASE_URL=your_db_url_here
```

**In productie:** Railway/Vercel environment variables

---

## SUPPORT & MAINTENANCE

### Wat je belooft aan klant
- **Uptime:** 99% (downtime max 7 uur/maand)
- **Response tijd bugs:** binnen 24 uur
- **Updates:** maandelijks of bij Meta/Gemini API changes
- **Support:** WhatsApp support binnen 4 uur op werkdagen

### Monitoring
- [ ] UptimeRobot (gratis) voor endpoint checks
- [ ] Sentry (gratis tier) voor error tracking
- [ ] Weekly report naar klant: # berichten, # orders, uptime

---

## PRICING STRATEGY

### Voor klanten
| Pakket | Prijs | Wat zit erin |
|--------|-------|--------------|
| **Starter** | €40/m | 1 agent (Order of FAQ) + 500 berichten/maand |
| **Basis** | €60/m | 2 agents + 1000 berichten/maand |
| **Pro** | €100/m | Alle agents + onbeperkt + review management |

**Setup fee:** €200-400 eenmalig (afhankelijk van complexiteit)

### Jouw kosten
- Tijd per klant setup: 3-4 uur (€0)
- Tijd per klant onderhoud: 30 min/maand (€0)
- Technische kosten: €5-10/maand
- **Winstmarge: €30-90/maand per klant**

**Break-even:** 2 klanten = €150/maand inkomsten = Claude Max (€100) gedekt

---

## TOOLS DIE JE NODIG HEBT

### Development
- [ ] Claude Code (heb je al)
- [ ] ngrok voor webhook testing (gratis)
- [ ] Postman/Insomnia voor API testing (gratis)
- [ ] Git/GitHub voor version control (gratis)

### Production
- [ ] Railway.app account (gratis + paid)
- [ ] Meta Business account voor WhatsApp API (gratis)
- [ ] Google Cloud account voor Gemini API (gratis tier)
- [ ] Supabase account voor database (gratis tier)

### Acquisitie
- [ ] Bright Data of ScraperAPI account (vanaf €10/m)
- [ ] Loops.so of Resend voor emails (gratis tier)
- [ ] Notion of Google Sheets voor CRM (gratis)

---

## VEELGESTELDE VRAGEN

**Q: Kan ik dit echt allemaal zelf bouwen?**
A: Ja. Met Claude Code doe je het samen met mij. Ik schrijf de code, jij test en deploy.

**Q: Hoeveel tijd kost dit per week?**
A: Fase 1-3: 10-15 uur/week. Daarna: 2-3 uur/week onderhoud + acquisitie.

**Q: Wat als een klant stopt?**
A: Je stopt de agent, bewaart de config. Nul extra kosten.

**Q: Kan ik dit schalen naar 50+ klanten?**
A: Ja, dat is het doel. Bij 50 klanten × €50/m = €2.500/m bij €500/m kosten = €2.000/m netto.

**Q: Moet ik programmeren kunnen?**
A: Nee. Claude Code + ik doen het zware werk. Jij leert het onderweg.

---

## VOLGENDE STAPPEN

1. **Nu meteen:** Maak dit project aan in Claude Code
2. **Deze week:** Bouw Order Agent voor De Rotterdammer (je hebt al het HTML bestand)
3. **Over 2 weken:** Test met echte klanten in zijn zaak
4. **Over 1 maand:** Template maken + 2e klant zoeken
5. **Over 3 maanden:** 10 klanten, €400-800/maand recurring revenue

---

**Laatste update:** 17 april 2026
**Eigenaar:** Moek Jagroep
**Contact:** [je WhatsApp]

---

# LET OP: GEBRUIK DIT BESTAND ALS SINGLE SOURCE OF TRUTH

Elke keer als je in Claude Code werkt aan dit project:
1. Lees eerst deze CLAUDE.md
2. Volg de structuur en naming conventions
3. Update dit bestand als iets verandert
4. Commit naar Git na elke milestone

**Succes, Moek. We gaan dit bouwen. Stap voor stap. Samen.**
