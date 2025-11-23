<link rel="stylesheet" href="../assets/css/custom.css">
<link rel="icon" type="image/x-icon" href="../favicon.ico">
<script src="../assets/js/snowstorm-min.js"></script>
<script>
  snowStorm.flakesMaxActive = 96;
  snowStorm.useTwinkleEffect = true;
  snowStorm.zIndex = 9999;
</script>


# 💡 Technische Hinweise & Hilfen
*Hier findest du Hilfestellungen und Ideen falls du mit den Aufgaben Schwierigkeiten hast.*

*Die folgenden Informationen sind nur als Hilfestellung für dich gedacht. Du kannst sie natürlich auch ignorieren und andere Technologien verwenden, wenn du möchtest.*

## 🎄 Gesamtübersicht & Tech-Stack Vorschlag

**Ziel:**
Ein Cloud-basiertes System, das:
- Wünsche sammelt
- Wichtel & Produktion verwaltet
- Trends auswertet
- Empfehlungen & Forecasts macht
- Rentiere trackt & Routen berechnet
- AI-Storytelling & Weihnachtskarten generiert
- alles über ein Dashboard sichtbar macht

**Backend (Core-API & Services):**
  - AWS Lambda (Node.js / TypeScript)
  - AWS API Gateway
  - AWS DynamoDB (NoSQL)
  - AWS EventBridge (für Cron/Jobs)
  - Optional: AWS SQS (Async-Jobs), AWS SNS (Benachrichtigungen)

**Frontend (Dashboard & Story-Seite):**

  - Next.js (React, TypeScript)
  - Deployment auf Vercel oder Netlify

**AI & Data:**
  - OpenAI API (LLM für Texte, evtl. Bilder)
  - Kleines eigenes Recommender-Modul (Node/TS oder Python-Microservice)
  - Einfaches Forecasting in Python (z. B. in einem separaten Service, per Lambda)

**Infra / DevOps:**
  - Terraform oder Serverless Framework
  - GitHub Actions für CI/CD
  - Monitoring über CloudWatch

Du kannst natürlich GCP/Azure statt AWS nutzen.

## 🧩 Datenmodell

**🎁 Wunschliste**

```json
{
  "wishlistId": "uuid",
  "childName": "Emma",
  "age": 7,
  "wishes": ["Puppe", "Schlitten"],
  "region": "DE",
  "createdAt": "2025-12-01T10:30:00Z"
}
```

**🧝‍♂️ Elf/Wichtel**
```json
{
  "elfId": "uuid",
  "name": "Filbert",
  "skill": "Baking",
  "performanceScore": 0.87,
  "createdAt": "2025-12-02T09:00:00Z"
}
```

**🔍 Production Log**
```json
{
  "logId": "uuid",
  "item": "Plüsch-Pinguin",
  "count": 12,
  "status": "completed",
  "timestamp": "2025-12-10T15:12:00Z",
  "elfId": "uuid",
  "notes": "Glitzer-Upgrade"
}
``` 

**🌎 Trend Record**
```json
{
  "trendId": "uuid",
  "keyword": "Rainbow Slime",
  "score": 87,
  "region": "EU",
  "capturedAt": "2025-12-11T08:00:00Z"
}
```

**🦌 Rentier Position**
```json
{
  "reindeerId": "Donner",
  "timestamp": "2025-12-17T18:42:10Z",
  "lat": 87.1123,
  "lon": -42.9921,
  "speed": 35.4,
  "status": "training"
}
```

## 📦 Backend Services

**1. Wunschliste**
Endpoints:
- `POST /wishlist`
- `GET /wishlist?childName=...`
- `GET /wishlist/{wishlistId}`

**2. Elf/Wichtel**
Endpoints:
- `POST /elves`
- `GET /elves`
- später `GET /elves/performance`  

  Hier kannst du später für Tag 8 einfach Performance-Metriken hinzufügen (Tasks pro Tag, Fehlerquote etc.).

**3. Production Logging + ETL**
- `POST /production/log` schreibt erstmal Rohdaten in DynamoDB oder S3.
- Ein EventBridge Scheduled Event (z. B. alle Stunde) triggert eine ETL-Lambda:
  - liest neue Logs
  - aggregiert Stats (per Item, per Elf, per Stunde/Tag)
  - schreibt in production_stats Table

**4. Trend Service**
- Eine Lambda, die z. B. Google Trends API / Etsy / Fake-Daten anfragt.
- Ergebnis in trends-Tabelle oder S3.
- Endpoint `GET /trends` für das Dashboard.

## 📊 Recommendation & Forecasting
**1. Recommendation Engine**  
**Idee:** Content-based, ohne Overkill.
- Merkmale eines Geschenks: Alter, Kategorien (kreativ, mechanisch, plüschig, techy), Preisbereich.
- Wishlist → Interessenprofil.
- Algorithmus: 
  - Score = Summe (Interessen-Match, Altersrange, Trend-Score)
- Liefert Top-N-Vorschläge.  

Endpoint: `POST /recommend` mit Payload.
```json
{
  "age": 8,
  "interests": ["Weltraum", "Roboter", "laute Dinge"],
  "region": "DE"
}
```

Rückgabe: Liste von Geschenk-Optionen (z. B. aus einer statischen Gift-Catalog-Table).

**2. Forecasting Service**
- Separater Lambda oder Docker-Container mit Python (z. B. `prophet` oder `statsmodels`).
- Nimmt `production_stats` + Trends.
- Rechnet einfache Prognosen für die nächsten Tage.
- Ergebnis in `demand_forecast` Table:
```json
{
  "item": "Plüsch-Pinguin",
  "date": "2025-12-24",
  "expectedCount": 420
}
```
Endpoint: `GET /forecast/items?item=Plüsch-Pinguin`

## 📜 AI-Story & Weihnachtskarten
**1. North-Pole Chronicle**
- Cron-Lambda 1x pro Tag:
  - holt Logs & Stats (z. B. Anzahl Wishlists, Top-Item, Anzahl Wichtel-Tasks).
  - erstellt Prompt für OpenAI API:
    - „Schreibe eine kurze, humorvolle Zusammenfassung des Tages am Nordpol: …“
  - speichert generierten Text in `chronicle` Table.
  - Endpoint `GET /chronicle/today`.

**2. Karten-Texte**
- Endpoint `POST /cards/text`:
  - Input: Name, Alter, Interessen.
  - Backend generiert Prompt für LLM.
  - Rückgabe: generierter, weihnachtlicher Text.

Du kannst einen „Fallback-Generator“ bauen (Hardcoded Textvorlagen), falls API nicht verfügbar ist.

## 🦌 Rentier-Tracking Dashboard
**1. Reindeer GPS Generator**
- Cron-Lambda alle X Sekunden/Minuten, die Pseudo-GPS-Daten generiert.
- Speichern in `reindeer_positions` Table.
- Option: partitioniert pro `reindeerId` + `date`.

**2. Live-Map**
- Frontend ruft z. B. alle 5–10 Sekunden `GET /reindeer/positions` auf.
- Next.js nutzt Map-Bibliothek (Leaflet, Mapbox, Google Maps).

**3. Routing Engine**
- Lambda `POST /route/optimal`:
  - Input: Liste von Koordinaten/Adressen.
  - Konvertiert zu Koordinaten (falls nötig).
  - TSP-Heuristik (Nearest Neighbour + 2-Opt, o. ä.).
  - Rückgabe: sortierte Route + Distanz + Reihenfolge.

## 🎨 Frontend Struktur Vorschlag
Seiten-Ideen:
- `/` – Übersicht (KPIs: Anzahl Wishlists, Produktionsvolumen, Trends, Forecast).
- `/wishlists` – Liste + Filter.
- `/elves` – Wichtel & Performance.
- `/production` – Charts & Logs.
- `/trends` – Trenddiagramme.
- `/reindeer` – Live-Map.
- `/chronicle` – Story des Tages.
- `/admin` – technische Ansicht (Logs, Alarme, Healthchecks).

Technisch:
- `getServerSideProps`/`getStaticProps` oder clientseitiges `fetch`.
- Nutzung von `SWR` oder `React Query` für Data-Fetching.
- Komponenten-Bibliothek: z. B. shadcn/ui, Chakra UI oder Tailwind + eigene Komponenten.