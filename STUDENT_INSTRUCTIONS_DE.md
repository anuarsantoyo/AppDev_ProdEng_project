# House Hunt Agent — Projektanleitung

## 1. Was du baust

Ein Full-Stack-KI-Agent, der einem Hauskäufer hilft zu entscheiden, ob ein Haus fair bepreist ist. Ein Nutzer registriert sich nur mit einer E-Mail, erhält einen API-Key und chattet mit einem Agenten. Der Agent entscheidet selbstständig, ob er weitere Informationen benötigt, einen Preis vorhersagen oder etwas nachschlagen soll.

Ein einzelner Nutzer-Zug sieht so aus:

    Nutzer: "Ich habe ein Haus mit 3 Schlafzimmern, 2 Bädern und 1.820 sqft Wohnfläche gefunden, Baujahr 1995, angeboten für 250.000 $."
    Agent:  (es fehlen Neighborhood, Quality, Lot Size ...) -> fragt alle auf einmal ab
    Nutzer: "Neighborhood OldTown, Quality 7/10, 7.000 sqft Grundstück, Garage für 2 Autos,
             800 sqft Keller, Central Air, 2 Stockwerke, 1 Kamin."
    Agent:  predict_price(...) -> 170.490 $ -> "Für 250.000 $ ist es überteuert; das Modell
            schätzt den Wert auf rund 170.000 $."

## 2. Was du dabei übst

Jedes Stück ist eine Fertigkeit aus dem Kurs, zusammengesetzt zu einem Produkt:

- Admission Control — Authentifizierung per API-Key, Rate Limiting pro Aufrufer, Credits + Aufladung
- Secrets — OpenRouter-Key in .env halten, niemals committen
- Die ML-Pipeline — Daten aus SQLite laden, vorverarbeiten, trainieren, mit joblib speichern
- Model Serving — das joblib einmal laden, aus wenigen Nutzereingaben vorhersagen
- MCP — deine Tools über einen eigenständigen MCP-Server bereitstellen
- RAG — Dokumente chunk-en, einbetten, indizieren, abrufen, cachen
- Der Agent-Loop — ein LLM, das überlegt, Tools aufruft und Ergebnisse beobachtet
- Serving — FastAPI mit den Security-Gates über Depends
- Der Client — ein Streamlit-UI

## 3. Die Anforderungen

### 3.1 Das ML-Modell (predict_price)

Trainiere ein Hauspreis-Modell auf dem Ames-Datensatz, den du als SQLite erhalten hast.

- Lade die SQLite-Datei in einen pandas-DataFrame — übe das Auslesen mit
  `pd.read_sql_query`; nicht einfach ein CSV fest verdrahten.
- Baue eine sklearn-Pipeline: Imputation (ein Sicherheitsnetz — die gelieferten Daten sind
  vollständig), One-Hot-Encoding der drei kategorialen Spalten (Neighborhood, HouseStyle,
  CentralAir), Skalieren der numerischen Spalten und Log-Transformation des Zielwerts
  (Preise sind rechts-schief).
- Trainiere eine lineare Regression auf den Log-Preis. Das ist das zu verwendende Modell.
  (Andere Modelle zu vergleichen oder Hyperparameter-Tuning ist optional — nicht der Fokus.)
- Speichere die gesamte Pipeline (Vorverarbeitung + Modell) als `house_price_model.joblib`.

Hinweise für das Serving (die brauchst du später für das Tool):

- Das Modell hat genau 13 Features, und der Käufer liefert alle — es muss nichts imputiert
  werden: Schlafzimmer, volle Bäder, halbe Bäder, Wohnfläche (sqft), Baujahr,
  Grundstücksgröße, Neighborhood, Gesamtqualität (1-10), Stellplätze, Kellerfläche (sqft),
  Central Air, Hausstil, Kamine.
- Die Neighborhood muss eine der 25 Ames-Neighborhoods sein; lehne alles andere mit einer
  klaren Meldung ab, die die gültigen Werte auflistet.

### 3.2 RAG — die Wissensbasis (search_knowledge)

Baue ein Retrieval-System über den Wohneigentums-/Steuerdokumenten, die du erhalten hast (drei IRS-Publikationen: Pub 530, Pub 936, Pub 523).

- Extrahiere den Text aus den PDFs.
- Chunk den Text in 600-Wort-Abschnitte mit 100-Wort-Überlappung.
- Bette die Chunks mit einem lokalen Embedding-Modell ein: fastembed's `BAAI/bge-small-en-v1.5`
  (384-dim, kein API-Key, funktioniert offline). Nutze keine kostenpflichtige Embedding-API.
- Speichere die Embeddings in einem Vektor-Store (Chroma, wie im Kurs).
- `search_knowledge(query)`: Bette die Query ein, hole die ähnlichsten Chunks
  (Kosinus-Ähnlichkeit) und gib sie zusammen mit ihrem Quelldokument zurück.
- Füge einen Cache hinzu: Wenn eine sehr ähnliche Query schon beantwortet wurde, gib das
  gecachte Ergebnis zurück statt neu zu rechnen — ein semantischer Cache (vergleiche das
  Query-Embedding mit den gecachten per Kosinus-Ähnlichkeit und einem Schwellenwert).

(Warum lokale Embeddings: kostenlos, kein Key, und für alle Studierenden identisch.)

### 3.3 Der MCP-Server (deine Tools)

Baue einen eigenständigen MCP-Server, der deine zwei Tools über Streamable-HTTP-Transport bereitstellt (gleiches Muster wie der MCP-Sprint im Kurs):

- `predict_price(...)` — lädt dein gespeichertes Modell und gibt den vorhergesagten Preis
  zurück (gib auch den RMSE des Modells zurück, damit der Agent ehrlich über die Unsicherheit
  sein kann).
- `search_knowledge(query)` — ruft deine RAG-Suche auf und gibt die relevanten Passagen zurück.

Anforderungen:

- Verwende das `MCPServer` des mcp-SDKs und den `@mcp.tool`-Dekorator.
- Gib jedem Tool einen klaren Docstring — das LLM liest ihn, um zu entscheiden, wann und wie es
  das Tool aufruft, und die Parametertypen definieren die erforderlichen Eingaben.
- `predict_price` muss alle 13 Käufer-Eingaben als erforderliche Parameter deklarieren — genau
  das bringt den Agenten dazu, die fehlenden nachzufragen.
- Stelle ihn auf einem eigenen Port mit uvicorn bereit (`streamable_http_app`).

### 3.4 Der Agent-Loop

Das ist das Herzstück des Projekts: ein LLM, das überlegt, ein Tool aufruft, das Ergebnis beobachtet und das wiederholt, bis es antworten kann.

- Der Agent hat genau zwei Tools: `predict_price` (ruft dein trainiertes Modell auf) und
  `search_knowledge` (ruft dein RAG auf). Keine anderen Tools.
- Nutze OpenRouter für das LLM (Chat-Modell in .env gesetzt, z. B. `google/gemini-2.5-flash`).
- Agiere als MCP-Client: verbinde dich über Streamable-HTTP mit deinem MCP-Server, initialisiere,
  liste seine Tools und wandle sie in die Tool-Schemata um, die du dem LLM übergibst.
- Gib dem LLM einen System-Prompt, der ihm sagt, wie es sich verhalten soll:
  - Preisfrage -> sammle jede fehlende `predict_price`-Eingabe (frage alle in EINER Frage ab,
    rate niemals einen Wert) -> rufe `predict_price` auf -> vergleiche mit einem etwaigen
    Angebotspreis -> gib ein kurzes Urteil (fair / überteuert / unterbewertet).
  - allgemeine Frage -> rufe `search_knowledge` auf -> antworte aus dem, was zurückkommt, und
    nenne das Quelldokument.
  - halte Antworten kurz und in klarer, freundlicher Sprache; erfinde niemals Preise oder Fakten.
- Der Loop: sende Konversation + Tools an das LLM. Wenn es einen Tool-Aufruf zurückgibt, führe ihn
  über den MCP-Client aus, lege das Ergebnis zurück in die Konversation und rufe das LLM erneut
  auf. Wenn es reinen Text zurückgibt, ist das deine Antwort.
- Begrenze die Anzahl der Tool-Aufruf-Runden (z. B. 4), damit es niemals endlos loopen kann.

### 3.5 Der FastAPI-Service (Admission Control + Chat)

Baue das FastAPI-Backend, das vor dem Agenten sitzt.

Endpoints:

- `POST /v1/signup` — `{email}` -> legt einen Nutzer an, erzeugt einen API-Key, speichert dessen
  Hash und gibt den Key EINMAL zurück. Nur E-Mail, kein Passwort.
- `POST /v1/chat` — `{message, history}` -> führt den Agenten aus, hinter den untenstehenden Gates.
- `GET /v1/usage` — die Credits des Aufrufers und was er verbraucht hat.
- `POST /v1/credits/topup` — `{amount}` -> lädt Credits auf.
- `GET /health`

Security-Gates, in dieser Reihenfolge mit FastAPI-`Depends` angewendet (wie in Class 1):

- 401 Authentifizierung — hashe den übermittelten `x-api-key` und schlage ihn nach; speichere nur
  den SHA-256-Hash, niemals den Key selbst.
- 429 Rate Limit — ein gleitendes 60-Sekunden-Fenster, max. N Requests/Minute pro Aufrufer, im
  Speicher gehalten.
- 402 Billing — ziehe Credits pro Aktion ab; ist der Saldo leer, lehne ab und sage "aufladen".

Credit-Modell:

- ein neuer Nutzer startet mit 100 Credits
- Chat-Nachricht = 1, `predict_price` = 3, `search_knowledge` = 2
- führe ein Nutzungs-Ledger: wer, welche Aktion, wie viele Credits, wann

Verdrahte den Agent-Loop in `/v1/chat`: ziehe den Chat-Credit im Voraus ab, führe den Loop aus und ziehe Credits für jeden Tool-Aufruf des Agenten ab. Gib Antwort + verbrauchte Credits + verbleibende Credits zurück.

Daten: SQLite (eine `users`-Tabelle + eine `usage`-Tabelle), wie in Class 1.

### 3.6 Der Client (Streamlit)

Eine einfache Streamlit-App genügt — der Punkt ist, dass du das Produkt wirklich nutzen kannst, nicht dass es schick aussieht.

- Registrieren (E-Mail -> API-Key erhalten) oder mit bestehendem Key einloggen.
- Eine Chatbox, um mit dem Agenten zu sprechen.
- Zeige deine verbleibenden Credits und eine Möglichkeit zum Aufladen.

Das war's. Halte es schlicht.

## 4. Was dir gegeben wird

Eine SQLite-Datenbank und drei PDFs. Alles andere baust du selbst.

- `data/app.db` — eine einzelne SQLite-Datei mit drei Tabellen:
  - `houses` — der Ames-Datensatz, vorab geladen: 1.460 Zeilen, 13 Feature-Spalten plus das
    `SalePrice`-Ziel (der volle ~75-Spalten-Ames-Satz wurde auf die 13 Features gekürzt, die ein
    Käufer realistisch angeben kann).
  - `users` — leer, nur Schema: `user_id` (E-Mail, Primärschlüssel), `key_hash`,
    `credits` (Standard 100), `created_at`. Du fügst bei der Registrierung eine Zeile ein.
  - `usage` — leer, nur Schema: `user_id`, `action`, `credits`, `ts`. Du fügst für jede
    abgerechnete Aktion eine Zeile ein.
- `rag/docs/` — drei IRS-Publikationen (Pub 530, Pub 936, Pub 523) als PDFs.

## 5. Architektur

    Streamlit (Browser)
        |  HTTP + x-api-key
        v
    FastAPI  :8000     Auth, Rate Limit, Credits, /v1/chat
        |                 (hier läuft der Agent-Loop)
        |  MCP-Client (Streamable HTTP)
        v
    MCP-Server :8001   predict_price, search_knowledge
        |                    |
        v                    v
    house_price_model.joblib   Chroma-Index (RAG)

Eine SQLite-Datenbank hält alles Relationale: `houses` (Training), `users` + `usage`
(Auth/Credits). Die RAG-Vektoren liegen in Chroma.

## 6. Empfohlene Build-Reihenfolge

Baue in dieser Reihenfolge — jeder Schritt ist für sich testbar, bevor du weitermachst:

1. Das ML-Modell (3.1) — zuerst, weil es die 13 Features festlegt, von denen alles andere abhängt.
2. Die RAG-Suche (3.2) — unabhängig vom Modell; als Zweites (oder parallel).
3. Den MCP-Server (3.3) — wickle 1 und 2 in deine zwei Tools.
4. Den FastAPI-Service (3.5) — zuerst Auth/Rate-Limit/Credits (das Class-1-Muster), dann
   den Agent-Loop (3.4), der sich mit deinem MCP-Server verbindet.
5. Den Streamlit-Client (3.6) — zuletzt, wenn die API funktioniert.

Abhängigkeitskette: ML + RAG -> MCP -> FastAPI (Agent) -> Streamlit. Das Modell ist der
Schlussstein — baue es zuerst, dann fügt sich der Rest.

Prüfe währenddessen: sage ein bekanntes Haus vorher, frage eine Steuerfrage, rufe jedes Tool auf,
curle die Endpoints, klicke dann durch das UI.

## 7. Setup und Ausführen

Voraussetzungen: Python 3.11+ und ein OpenRouter-API-Key.

- Erstelle eine virtuelle Umgebung und schreibe deine eigene `requirements.txt`, während du
  installierst, was du brauchst (fastapi, uvicorn, mcp, openai, chromadb, fastembed, scikit-learn,
  pandas, joblib, pypdf, streamlit, ...).
- Lege deinen Key in `.env` ab (gitignored — niemals committen):
  `OPENROUTER_API_KEY=...` und `OPENROUTER_MODEL=...` (ein Modell, das Tool-Calling
  unterstützt, z. B. `google/gemini-2.5-flash`).

Einmalige Datenvorbereitung (deine eigenen Skripte):

    ML-Modell bauen   houses aus app.db laden -> vorverarbeiten -> trainieren -> .joblib speichern
    RAG-Index bauen   PDFs extrahieren -> chunk-en (600/100) -> einbetten -> in Chroma indizieren

Ausführen (drei Terminals, vom Projekt-Root):

    uvicorn your_mcp_server:app --port 8001        # der MCP-Server
    uvicorn your_app:app --port 8000               # der FastAPI-Service
    streamlit run your_client.py                   # das UI

(Kein Datenbank-Erstellungsschritt — die gelieferte `app.db` hat die drei Tabellen bereits.)

## 8. Abnahmekriterien

Dein Projekt ist fertig, wenn jedes hiervon funktioniert:

- Signup gibt einen API-Key einmal zurück; dieselbe E-Mail erneut zu registrieren schlägt fehl.
- Ein fehlender oder falscher Key -> 401.
- Mehr als N Requests in einer Minute -> 429.
- Leere Credits -> 402; Aufladen lässt dich weitermachen.
- Frage nach einem Haus mit nur einigen Details -> der Agent fragt die fehlenden Eingaben ab
  (in einer Frage).
- Gib ihm alle 13 Details -> er sagt einen Preis vorher und vergleicht ihn mit dem Angebotspreis
  samt Urteil.
- Stelle eine Steuer-/Hauskauf-Frage -> er antwortet aus den RAG-Dokumenten und nennt die Quelle.
- Credits werden korrekt abgezogen: Chat = 1, predict_price = 3, search_knowledge = 2.
- Der Usage-Endpoint zeigt deinen Verbrauch; eine wiederholte RAG-Frage trifft den Cache.
