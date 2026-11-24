# House Hunt Agent — Roadmap / Roadmap

---

## 🇬🇧 English

**The idea:** build the two "brains" first (a price predictor and a document search), wrap them as callable tools, prove the LLM works, then let the LLM drive those tools, and finally put the security gates in front. Each step is independently testable, and CI grows with every step.

1. **ML model → `predict_price()` function.** Turn the dataset into a saved model, and expose it as a simple function that takes the buyer's details and returns a price (and how uncertain it is).

2. **RAG → `search_knowledge()` function.** Build a search function that takes a question and returns the most relevant passages, plus which document they came from — with a memory so repeat questions are instant. Chunks are embedded via **OpenRouter embeddings** (using the OpenRouter key already in `.env`, no local model) and stored in the vector index.

3. **MCP server on `:8001`.** Wrap both functions as tools on a standalone tool server the agent can call. The price tool demands *every* house detail, which is what forces the agent to ask for what's missing.

4. **FastAPI + simple chatbot (OpenRouter smoke test).** A minimal chat endpoint that just talks to the model — no tools, no auth — so any problem is clearly the model connection, not the plumbing.

5. **Agent loop (the async step).** Connect the agent to the tools and let it reason: gather missing details in one question, call the price tool, compare to the listing, give a verdict; or search the documents and answer with a source. Bounded so it can never loop forever.

6. **Auth, rate limit, credits — then wire billing.** Add signup with API keys, rate limiting, and a credit balance — charge for chats and for each tool the agent uses, with a usage ledger and top-up.

---

## 🇩🇪 Deutsch

**Die Idee:** Zuerst die beiden „Gehirne" bauen (einen Preis-Vorhersager und eine Dokumentsuche), sie als aufrufbare Tools verpacken, beweisen, dass das LLM funktioniert, dann das LLM diese Tools steuern lassen und zum Schluss die Sicherheitsschranken davor setzen. Jeder Schritt ist eigenständig testbar, und CI wächst mit jedem Schritt mit.

1. **ML-Modell → `predict_price()`-Funktion.** Den Datensatz in ein gespeichertes Modell verwandeln und als einfache Funktion bereitstellen, die die Angaben des Käufers entgegennimmt und einen Preis zurückgibt (sowie die Unsicherheit dazu).

2. **RAG → `search_knowledge()`-Funktion.** Eine Suchfunktion bauen, die eine Frage entgegennimmt und die relevantesten Textstellen zurückgibt — inklusive des Dokuments, aus dem sie stammen — mit einem Gedächtnis, damit wiederholte Fragen sofort beantwortet werden. Die Chunks werden über **OpenRouter-Embeddings** eingebettet (mit dem OpenRouter-Key aus der `.env`, kein lokales Modell) und im Vektorindex gespeichert.

3. **MCP-Server auf `:8001`.** Beide Funktionen als Tools auf einem eigenständigen Tool-Server bereitstellen, den der Agent aufrufen kann. Das Preis-Tool verlangt *jede* Hausangabe — genau das zwingt den Agenten, nach fehlenden Angaben zu fragen.

4. **FastAPI + einfacher Chatbot (OpenRouter-Smoke-Test).** Ein minimaler Chat-Endpunkt, der nur mit dem Modell spricht — keine Tools, keine Authentifizierung — damit jedes Problem eindeutig an der Modellverbindung liegt und nicht an der Verkabelung.

5. **Agent-Loop (der Async-Schritt).** Den Agenten mit den Tools verbinden und ihn denken lassen: fehlende Angaben in einer einzigen Frage sammeln, das Preis-Tool aufrufen, mit dem Angebotspreis vergleichen, ein Urteil geben; oder die Dokumente durchsuchen und mit Quellenangabe antworten. Begrenzt, damit er nie endlos schleifen kann.

6. **Auth, Rate-Limit, Credits — dann Abrechnung verdrahten.** Registrierung mit API-Schlüsseln, Rate-Limiting und einem Credit-Guthaben ergänzen — Chats und jeder vom Agenten genutzte Tool-Aufruf werden berechnet, mit einem Nutzungs-Ledger und Aufladen.
