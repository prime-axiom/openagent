# OAuth-/Login-Vergleich: ClaudeCode vs. OpenAgent

> Basis: ausschließlich tatsächlich vorhandener Code in `/workspace/ClaudeCode` und `/workspace/openagent`.
> Keine Spekulationen ohne Kennzeichnung.

## Kurzfazit

- **ClaudeCode** hat eine umfangreiche, produktionsreife OAuth-Implementierung für einen nativen/CLI-zentrierten Client mit lokalem Callback-Server, manuellem Fallback, PKCE, `state`, Refresh-Mechanik, Cross-Process-Cache-Invalidierung, dedupliziertem 401-Handling, sicherer Persistenz und vielen Sonderfällen.
- **OpenAgent** hat **zwei klar getrennte Auth-Welten**:
  1. **eigene App-Authentifizierung** via Username/Password + JWT/Refresh-JWT (`/api/auth/login`, `/api/auth/refresh`) für Web-Frontend ↔ Web-Backend.
  2. **Provider-OAuth** für externe Modellanbieter in den Admin-Provider-Routen (`/api/providers/oauth/*`), delegiert an `@mariozechner/pi-ai/oauth`.
- Für den gefragten OAuth-/Login-Vergleich ist wichtig: **OpenAgent implementiert keinen vollständigen eigenen OAuth-Authorization-Code-Flow mit PKCE/State im Repository selbst**, sondern orchestriert einen externen OAuth-Provider-Adapter und speichert das Ergebnis. Dagegen implementiert **ClaudeCode den Flow selbst im Codebestand**.

---

## 1. Einstiegspunkte / Architektur

## ClaudeCode

### Relevante Einstiegspunkte

- CLI-Login-Command:
  - `ClaudeCode/src/main.tsx`
    - `program.command('auth')...`
    - `auth login`, `auth status`, `auth logout`
- Handler:
  - `ClaudeCode/src/cli/handlers/auth.ts`
    - `authLogin()`
    - `installOAuthTokens()`
    - `authStatus()`
    - `authLogout()`
- TUI-/Konsole-Flow:
  - `ClaudeCode/src/components/ConsoleOAuthFlow.tsx`
- OAuth-Service:
  - `ClaudeCode/src/services/oauth/index.ts`
  - `ClaudeCode/src/services/oauth/client.ts`
  - `ClaudeCode/src/services/oauth/auth-code-listener.ts`
- Laufzeit-/Tokenverwaltung:
  - `ClaudeCode/src/utils/auth.ts`

### Architektur

ClaudeCode trennt die OAuth-Implementierung in mehrere Schichten:

1. **UI-/CLI-Schicht**
   - startet Login,
   - zeigt URL/Fortschritt/Fehler,
   - erlaubt manuelle Code-Eingabe.

2. **OAuth-Flow-Service (`OAuthService`)**
   - erstellt PKCE-Werte,
   - startet localhost Callback-Listener,
   - erzeugt Auth-URLs,
   - wartet auf automatischen oder manuellen Authorization Code,
   - tauscht Code gegen Token.

3. **OAuth-Client-Schicht**
   - URL-Building,
   - Token-Exchange,
   - Token-Refresh,
   - Profil-/Rollen-/API-Key-Fetch,
   - Account-Metadaten-Update.

4. **Auth-/Persistenz-/Refresh-Schicht**
   - liest/speichert Tokens,
   - cached und invalidiert,
   - behandelt 401,
   - koordiniert Refresh via Locking.

Das ist eine relativ vollständige Native-App-/CLI-OAuth-Architektur.

---

## OpenAgent

### Relevante Einstiegspunkte

#### A. OpenAgent-App-Login
- Backend:
  - `openagent/packages/web-backend/src/routes/auth.ts`
  - `openagent/packages/web-backend/src/auth.ts`
- Frontend:
  - `openagent/packages/web-frontend/composables/useAuth.ts`
  - `openagent/packages/web-frontend/composables/useApi.ts`
  - `openagent/packages/web-frontend/middleware/auth.global.ts`
  - `openagent/packages/web-frontend/pages/login.vue`

#### B. Provider-OAuth für externe Modellanbieter
- Backend:
  - `openagent/packages/web-backend/src/routes/providers.ts`
- Frontend:
  - `openagent/packages/web-frontend/composables/useProviders.ts`
  - `openagent/packages/web-frontend/components/ProviderFormDialog.vue`
- Core-/Persistenzseite:
  - `openagent/packages/core/src/provider-config.ts`

### Architektur

OpenAgent hat **keine einheitliche OAuth-Login-Architektur für die App selbst** wie ClaudeCode, sondern:

1. **App-Login**
   - klassisch per Username/Password gegen eigenes Backend,
   - Ausgabe von Access- und Refresh-JWT,
   - Frontend speichert Token in `localStorage`.

2. **Provider-OAuth**
   - nur für das Hinzufügen externer Provider,
   - Backend startet OAuth-Flow über `oauthProvider.login(...)` aus externer Library,
   - Backend verwaltet temporären In-Memory-Loginstatus,
   - Frontend pollt Status,
   - Abschluss führt zur dauerhaften Speicherung der Provider-Credentials.

Wichtig: **Der eigentliche OAuth-Flow steckt nicht sichtbar in OpenAgent selbst, sondern in `@mariozechner/pi-ai/oauth`.** Im Repository sieht man nur die Orchestrierung.

---

## 2. OAuth-Flows

## ClaudeCode

## Unterstützte Flow-Varianten

Aus dem Code sichtbar:

1. **Automatischer Browser-Flow mit localhost-Callback**
   - `OAuthService.startOAuthFlow()` startet `AuthCodeListener`
   - Redirect URI: `http://localhost:${port}/callback`
   - Code wird automatisch abgefangen.

2. **Manueller Fallback-Flow**
   - `buildAuthUrl(... isManual: true)` nutzt `MANUAL_REDIRECT_URL`
   - UI fordert Benutzer auf, den Code bzw. `authorizationCode#state` manuell einzufügen
   - `handleManualAuthCodeInput()` injiziert den Code in den laufenden Flow.

3. **Setup-Token-Flow**
   - in `ConsoleOAuthFlow.tsx` mit `mode === 'setup-token'`
   - nutzt `inferenceOnly` und optional langes `expiresIn`
   - zeigt den erzeugten Token anstatt Standard-Loginabschluss.

4. **Headless-/SDK-Control-Flow**
   - `src/cli/print.ts` behandelt `claude_authenticate`, `claude_oauth_callback`, `claude_oauth_wait_for_completion`
   - URLs werden über Kontrollkanal zurückgegeben
   - Browsersteuerung liegt beim Client.

## PKCE / State

Klar im Code vorhanden:

- `OAuthService`:
  - `this.codeVerifier = crypto.generateCodeVerifier()`
  - `const codeChallenge = crypto.generateCodeChallenge(this.codeVerifier)`
  - `const state = crypto.generateState()`
- `buildAuthUrl()` setzt:
  - `code_challenge`
  - `code_challenge_method=S256`
  - `state`
- `exchangeCodeForTokens()` sendet zusätzlich:
  - `code_verifier`
  - `state`

## Redirect-/Callback-Handling

- `AuthCodeListener` startet lokalen HTTP-Server.
- Er lauscht auf `/callback`.
- Er validiert:
  - dass `code` vorhanden ist,
  - dass `state === expectedState` ist.
- Bei Erfolg speichert er `pendingResponse`, damit danach noch eine Browser-Weiterleitung auf eine Success-Page gesendet werden kann.
- Bei Fehler:
  - HTTP 400,
  - Abbruch mit `Invalid state parameter` oder `No authorization code received`.

## Tokenaustausch

- `exchangeCodeForTokens()` POSTet auf `getOauthConfig().TOKEN_URL`
- sendet:
  - `grant_type=authorization_code`
  - `code`
  - `redirect_uri`
  - `client_id`
  - `code_verifier`
  - `state`
- Antwort wird in `OAuthTokens` umgeformt.

## Persistenz / Speicherung

- `installOAuthTokens()` in `cli/handlers/auth.ts`:
  - führt zuerst `performLogout({ clearOnboarding: false })` aus,
  - speichert Profilinfos,
  - ruft `saveOAuthTokensIfNeeded(tokens)` auf,
  - lädt Rollen,
  - erstellt ggf. API-Key für Console-User,
  - invalidiert Caches.
- `saveOAuthTokensIfNeeded()` in `utils/auth.ts`:
  - speichert Tokens in `secureStorage` unter `claudeAiOauth`
  - enthält `accessToken`, `refreshToken`, `expiresAt`, `scopes`, `subscriptionType`, `rateLimitTier`.

## Refresh-Strategie

ClaudeCode hat eine deutlich ausgebaute Refresh-Logik:

- `refreshOAuthToken()` in `services/oauth/client.ts`
- `checkAndRefreshOAuthTokenIfNeeded()` in `utils/auth.ts`
- Eigenschaften:
  - prüft lokale Ablaufzeit mit Buffer,
  - invalidiert Caches bei Dateisystemänderungen,
  - re-read aus Secure Storage,
  - nutzt Locking via `lockfile.lock(claudeDir)`,
  - retryt bei gesperrtem Lock,
  - dedupliziert parallele Refreshes.

## 401-/Fehlerbehandlung

Sehr stark ausgebaut:

- `handleOAuth401Error(failedAccessToken)`:
  - dedupliziert gleiche 401s per `pending401Handlers`
  - invalidiert Cache
  - liest Tokens neu
  - wenn Keychain schon neuen Token enthält: Recovery ohne weiteren Refresh
  - sonst force-refresh.
- `withOAuth401Retry()` / diverse API-Aufrufer verwenden diese Logik weiter.
- `ConsoleOAuthFlow` ergänzt UX-Hinweise für SSL/TLS-Intercept-Probleme via `getSSLErrorHint()`.

## Sicherheitsmaßnahmen

Im vorhandenen Code sichtbar:

- PKCE (`S256`)
- `state`-Validierung
- Loopback Redirect statt eingebettetem Secret-Flow
- manueller Fallback schließt Listener und verwendet laufenden Resolver
- Secure Storage statt einfachem Klartext-State im UI
- Locking bei Refresh
- Cache-Invalidierung über Dateisystemänderungen
- Org-Validierung via `validateForceLoginOrg()`
- getrennte Behandlung von inference-only Tokens ohne Refresh.

## UX / Fallbacks / Sonderfälle

Stark ausgebaut:

- Browser wird automatisch geöffnet
- URL wird angezeigt, falls Browser nicht aufging
- Copy-Hotkey (`c`) für URL
- manuelle Code-Eingabe
- spezielle Setup-Token-UX
- unterstützte Loginmethoden: `claudeai`, `console`, `sso`
- org-spezifischer Login per `orgUUID`
- Login-Hint (`login_hint`)
- Headless-Control-Channel-Unterstützung
- getrennte Success-Redirects für Claude.ai vs Console.

---

## OpenAgent

## Unterstützte Flows

### A. App-Login

Kein OAuth, sondern klassischer Credential-Login:

- `POST /api/auth/login` in `openagent/packages/web-backend/src/routes/auth.ts`
- validiert Username/Password gegen DB
- erzeugt:
  - Access Token (`1h`)
  - Refresh Token (`7d`)
- Frontend (`useAuth.ts`) speichert beide in `localStorage`.

### B. Provider-OAuth

In `openagent/packages/web-backend/src/routes/providers.ts`:

- `POST /api/providers/oauth/login`
  - validiert Provider-Typ,
  - holt Preset aus `PROVIDER_TYPE_PRESETS`,
  - prüft `preset.authMethod === 'oauth'`,
  - holt Adapter via `getOAuthProvider(preset.oauthProviderId)`.
- Dann:
  - erzeugt `loginId = crypto.randomUUID()`
  - legt `PendingOAuthLogin` in `pendingOAuthLogins` ab
  - startet `oauthProvider.login({...})` asynchron.

Der Rückgabevertrag des Adapters:

- `onAuth(info)` liefert `url` und `instructions`
- `onPrompt(prompt)` wird mit Defaults beantwortet
- `onProgress()` ist leer
- `onManualCodeInput` wird nur gesetzt, wenn `oauthProvider.usesCallbackServer` wahr ist
- bei Erfolg landen `credentials` in `loginState.credentials`
- bei Fehler `loginState.status = 'error'`.

## PKCE / State

### Im OpenAgent-Repository direkt sichtbar

**Nicht sichtbar.**

Der Backend-Code ruft nur `oauthProvider.login(...)` auf. Ob PKCE, `state`, Nonce oder Callback-Validierung verwendet werden, ist in diesem Repository **nicht direkt nachvollziehbar**.

Saubere Kennzeichnung:

- **Fakt:** OpenAgent selbst implementiert PKCE/`state` nicht im hier sichtbaren Code.
- **Unbekannt aus diesem Codebestand:** ob `@mariozechner/pi-ai/oauth` intern PKCE/`state` nutzt.

## Redirect-/Callback-Handling

### Sichtbares Verhalten

- Backend verwaltet nur den Flow-Status in Memory.
- `startOAuthLogin()` liefert dem Frontend:
  - `loginId`
  - `authUrl`
  - `instructions`
  - `usesCallbackServer`
- Frontend (`ProviderFormDialog.vue`) öffnet `window.open(response.authUrl, '_blank')`.
- Danach pollt das Frontend `GET /api/providers/oauth/status/:loginId`.
- Falls `usesCallbackServer` aktiv ist, zeigt das Frontend ein Feld für manuellen Code/Redirect-Eingabe.
- `POST /api/providers/oauth/code/:loginId` reicht den eingegebenen String an `resolveManualCode(code.trim())` weiter.

### Wichtige Beobachtung

Das Frontend spricht von:
- „paste the redirect URL here“
- Placeholder „Paste redirect URL or code...“

Aber der Backend-Handler nimmt nur ein Feld `code` entgegen und reicht **nur `code.trim()`** weiter.

Es gibt im sichtbaren OpenAgent-Code **keine lokale URL-Validierung, keine Extraktion von `code` oder `state` aus einer Redirect-URL**.

Daher gilt:

- **Faktisch sichtbar:** OpenAgent unterstützt manuellen Fallback nur als opaque String-Weitergabe.
- **Nicht sichtbar:** ob der Adapter eine volle Redirect-URL erwartet oder nur einen Authorization Code.
- **Risiko:** UI-Text und Backend-Verhalten könnten inkonsistent sein.

## Tokenaustausch

Im Repository nicht selbst implementiert.

Sichtbar ist nur:
- `oauthProvider.login(...).then(credentials => { loginState.credentials = credentials })`

Das heißt:
- Code-gegen-Token-Tausch findet außerhalb des Repos im OAuth-Adapter statt.
- OpenAgent übernimmt erst das fertige `credentials`-Objekt.

## Persistenz / Speicherung

### App-Login

- Frontend speichert in `localStorage`:
  - `openagent_access_token`
  - `openagent_refresh_token`
  - `openagent_user`

### Provider-OAuth

- Nach erfolgreichem Statuspoll:
  - Backend ruft `addOAuthProvider({... oauthCredentials: loginState.credentials })` auf.
- In `packages/core/src/provider-config.ts` ist sichtbar:
  - `oauthCredentials` werden „encrypted at rest“ gespeichert,
  - es gibt `encryptOAuthCredentials(...)` / `decryptOAuthCredentials(...)`-Nutzung.

Damit ist bei Provider-Credentials die Persistenz deutlich besser als beim Frontend-App-Login.

## Refresh-Strategie

### App-Login

- `useApi.ts`:
  - wenn API-Aufruf `401` zurückgibt und ein Token vorhanden ist,
  - ruft Frontend `refreshAccessToken()` auf,
  - wiederholt den Request genau einmal mit neuem Access Token,
  - wenn Refresh fehlschlägt: `logout()`.
- `useAuth.ts`:
  - `refreshAccessToken()` ruft `/api/auth/refresh` auf,
  - setzt neue Access-/Refresh-Tokens.

### Provider-OAuth

- In `packages/core/src/provider-config.ts` sichtbar:
  - `getApiKeyForProvider()` arbeitet bei OAuth-Providern mit
    `getOAuthApiKey(...)`
  - wenn `newCredentials.access !== oauthCreds.access || newCredentials.expires !== oauthCreds.expires`,
    werden Credentials aktualisiert.

Das heißt:
- Es gibt eine Refresh-/Credential-Rotation auf Provider-Ebene,
- aber die eigentliche Refresh-Logik steckt wieder im externen OAuth-/Provider-Layer.

## 401-/Fehlerbehandlung

### App-Login

Sichtbar und solide, aber einfach:

- `jwtMiddleware` gibt bei fehlendem/ungültigem Bearer direkt `401` zurück.
- Frontend `apiFetch()`:
  - bei `401`: versucht Refresh,
  - danach Retry,
  - sonst Logout + Fehler `Session expired`.

### Provider-OAuth

- Statusfehler werden als `status: 'error'` oder HTTP 500/400/404 transportiert.
- Kein sichtbares dedupliziertes 401-Handling auf Provider-OAuth-Orchestrierungsseite.
- Kein sichtbares Retry- oder Backoff-System für OAuth-Statuspolling außer dem simplen Frontend-Poll-Loop.

## Sicherheitsmaßnahmen

### Sichtbar im OpenAgent-Code

#### App-Login
- JWT-basierte Auth mit Bearer-Header
- Middleware-basierte Route-Sicherung
- Rollenkontrolle für Admin-Routen in `providers.ts`
- `oauthCredentials` im Core verschlüsselt gespeichert

#### Provider-OAuth-Orchestrierung
- `loginId` per `crypto.randomUUID()`
- temporäre In-Memory-State-Map
- automatische Bereinigung alter Pending-Logins nach 10 Minuten

### Nicht sichtbar / nicht belegbar

- PKCE
- `state`-Validierung
- CSRF-Schutz des OAuth-Flows
- Redirect-URL-Validierung
- Callback-Port-/localhost-Listener-Sicherheit

Das kann im externen Adapter existieren, ist hier aber **nicht aus dem vorhandenen Code ableitbar**.

## UX / Fallbacks / Sonderfälle

### Vorhanden

- OAuth-Dialog in `ProviderFormDialog.vue`
- Browser wird in neuem Tab geöffnet
- Polling auf Abschluss
- optionales manuelles Code-Feld
- Timeout nach 120 Versuchen à 1 Sekunde = ca. 2 Minuten
- Fehleranzeige im Dialog
- Sperre des Dialogschließens während `oauthInProgress`

### Auffällige Sonderfälle / Grenzen

- Pending OAuth Logins leben **nur im Backend-Prozessspeicher**:
  - bei Server-Neustart gehen sie verloren.
- Es gibt keine sichtbare Benutzer-/Session-Bindung des `loginId` außer dem allgemeinen Admin-JWT auf den Routen.
- Manual-Code-Pfad validiert nicht sichtbar die Eingabe.
- Polling ist einfach und frontendgetrieben, ohne serverseitige Events/WebSocket.

---

## 3. Gemeinsamkeiten

1. **Beide unterstützen Refresh-basierte Sitzungsfortsetzung**
   - ClaudeCode: OAuth Refresh Token
   - OpenAgent App-Login: JWT Refresh Token
   - OpenAgent Provider-OAuth: indirekt via gespeicherte OAuth-Credentials im Core

2. **Beide haben einen manuellen Fallback-/Benutzerinteraktionspfad für OAuth-nahe Flows**
   - ClaudeCode: manuelle Code-Eingabe (`authorizationCode#state`)
   - OpenAgent: manuelle Code-/Redirect-Eingabe im ProviderDialog

3. **Beide trennen UI von eigentlicher Auth-Logik**
   - ClaudeCode: UI-Komponente ↔ `OAuthService` ↔ Client/Auth-Layer
   - OpenAgent: Vue-Dialog ↔ Provider-API ↔ Core/OAuth-Adapter

4. **Beide behandeln 401 nicht nur als finalen Fehler**
   - ClaudeCode: Cache-Invalidate + Refresh + Retry
   - OpenAgent: Refresh-Token verwenden und API-Request wiederholen

5. **Beide speichern Auth-/Credential-Zustand persistent**
   - ClaudeCode: Secure Storage
   - OpenAgent App-Login: `localStorage`
   - OpenAgent Provider-Credentials: verschlüsselt at-rest im Core

---

## 4. Unterschiede

## Fundamentaler Unterschied: Wer implementiert den OAuth-Flow?

### ClaudeCode
- implementiert den OAuth-Code-Flow im eigenen Repository selbst.
- PKCE, State, Redirect-Listener, Token-Exchange, Refresh, Cache-Management sind sichtbar.

### OpenAgent
- implementiert primär die **Orchestrierung** eines externen OAuth-Adapters.
- eigentliche OAuth-Sicherheitsdetails sind im Repo nicht nachvollziehbar.

## Login-Ziel

### ClaudeCode
- OAuth ist Kern des Produkt-Logins gegen Anthropic/Claude-Dienste.

### OpenAgent
- App-Login ist kein OAuth, sondern eigenes JWT-System.
- OAuth dient Provider-Anbindung, nicht primär Benutzer-Login in die App.

## Callback-Handling

### ClaudeCode
- robuster nativer Loopback-Listener (`/callback`),
- automatische und manuelle Pfade,
- klare State-Validierung.

### OpenAgent
- kein im Repo sichtbarer eigener Callback-Server,
- Frontend öffnet Browser und pollt,
- manual path reicht nur String weiter,
- Redirect-/State-Validierung im Repo nicht sichtbar.

## Persistenz

### ClaudeCode
- Secure Storage / Keychain-orientiert,
- Cache-Layer + Cross-Process-Invalidierung.

### OpenAgent
- App-Access-/Refresh-Tokens in `localStorage`,
- Provider-Credentials verschlüsselt gespeichert,
- laufende OAuth-Pending-State nur in Memory.

## Refresh-Robustheit

### ClaudeCode
- Locking,
- deduplizierte parallele 401-Behandlung,
- Race-Recovery,
- Dateisystem-basierte Cache-Invalidierung.

### OpenAgent
- einfacher 401 → Refresh → Retry,
- keine sichtbare Lock-/Dedupe-/Race-Schutzlogik im Frontend-App-Login,
- Provider-Refresh nur indirekt sichtbar.

## Fehler- und Sonderfalltiefe

### ClaudeCode
- viele explizite Sonderfälle:
  - SSL-Intercept,
  - org mismatch,
  - setup-token,
  - headless control channel,
  - stale cache,
  - inference-only token,
  - parallel refreshes.

### OpenAgent
- deutlich schlanker und funktional, aber mit weniger erkennbarer Tiefe.

---

## 5. Was ClaudeCode robuster / besser löst

## 5.1 Vollständige, auditierbare OAuth-Implementierung

ClaudeCode löst den kompletten Authorization-Code-Flow im eigenen Code:
- URL-Erzeugung,
- PKCE,
- `state`,
- Callback-Capture,
- Code Exchange,
- Refresh,
- Persistenz.

Bei OpenAgent ist das für Provider-OAuth im Repo nicht vollständig nachvollziehbar.

**Warum besser:**
- besser auditierbar,
- besser testbar,
- weniger Black-Box-Abhängigkeit.

## 5.2 State-/CSRF-Schutz sichtbar und strikt

ClaudeCode validiert `state` explizit im Callback-Listener.

OpenAgent zeigt im sichtbaren Code keine eigene `state`-Validierung.

**Warum besser:**
- klarer Schutz gegen falsche/unerwartete Redirects,
- Security-Eigenschaften sind direkt im Repo belegbar.

## 5.3 Automatic + Manual Callback Handling sauber kombiniert

ClaudeCode unterstützt:
- lokalen Callback,
- manuelle Eingabe,
- SDK-/Headless-Varianten.

OpenAgent hat zwar manuelle Eingabe und Browser-Öffnen, aber kein gleichwertig sichtbares automatisches Callback-System im Repo.

## 5.4 Refresh-Koordination unter Konkurrenz

ClaudeCode hat sehr ausgereifte Konkurrenzbehandlung:
- in-flight dedup,
- Locking,
- Race-Recovery,
- Keychain-Re-Read,
- Cross-Process-Invalidierung.

OpenAgent-App-Login ist funktional, aber viel einfacher.

**Warum besser:**
- reduziert 401-Stürme,
- verhindert doppelte Refreshes,
- besser in Multi-Tab-/Multi-Prozess-Szenarien.

## 5.5 Sicherere Token-Persistenz für Hauptauth

ClaudeCode speichert Haupt-OAuth-Zustand in Secure Storage.

OpenAgent speichert App-JWTs im Frontend in `localStorage`.

**Warum besser:**
- geringere Exposition gegenüber Frontend-/Browser-Kontext-Risiken.

## 5.6 Bessere Fehler-UX

ClaudeCode behandelt konkrete Probleme mit gezielten Hinweisen, z. B.:
- TLS-/SSL-Intercept,
- Re-Login-Hinweise,
- Org-Mismatch,
- differenzierte Erfolgs-/Retry-Stati.

OpenAgent zeigt Fehler im Dialog, aber deutlich generischer.

## 5.7 Robustere Cleanup-/Lifecycle-Strategie

ClaudeCode:
- `cleanup()` schließt Listener,
- ersetzt vorherige Flows,
- behandelt Headless-Control-Flows explizit.

OpenAgent:
- Pending-State wird zwar bereinigt,
- aber ist nur In-Memory und dadurch fragiler gegenüber Neustarts.

---

## 6. Konkrete Verbesserungsvorschläge für OpenAgent

Priorisierung nach **Impact** und **Aufwand**.

## Priorität 1 — Hoher Impact, geringer bis mittlerer Aufwand

### 1. Manual OAuth Input serverseitig eindeutig validieren

**Ist-Zustand:**
- `POST /api/providers/oauth/code/:loginId` nimmt `code` entgegen und reicht String ungeprüft an `resolveManualCode(code.trim())` weiter.
- Frontend spricht aber von „redirect URL oder code“.

**Verbesserung:**
- explizit festlegen und validieren, ob erwartet wird:
  - nur Authorization Code, oder
  - volle Redirect-URL.
- Falls Redirect-URL erlaubt ist:
  - serverseitig parsen,
  - `code` extrahieren,
  - falls vorhanden auch `state` extrahieren,
  - ungültige Eingaben früh ablehnen.

**Impact:** hoch
**Aufwand:** niedrig bis mittel

### 2. Provider-OAuth-Flow an Benutzer/Session stärker binden

**Ist-Zustand:**
- `pendingOAuthLogins` ist eine globale In-Memory-Map.
- Sichtbar gespeichert werden Provider-Typ, Name, Modell, Credentials, aber kein expliziter `startedByUserId`.

**Verbesserung:**
- `PendingOAuthLogin` um `startedByUserId` erweitern,
- `status`-/`code`-Endpoints prüfen, dass derselbe Admin-User zugreift.

**Impact:** hoch
**Aufwand:** niedrig

### 3. Klare Trennung von App-Login und Provider-OAuth in Doku/UI

**Ist-Zustand:**
- OpenAgent hat JWT-App-Login plus Provider-OAuth.
- Das ist architektonisch sinnvoll, aber begrifflich leicht verwechselbar.

**Verbesserung:**
- Benennung im UI/Docs klarer machen:
  - „OpenAgent Sign-In“ vs.
  - „Connect Provider Account via OAuth“.

**Impact:** mittel-hoch
**Aufwand:** niedrig

### 4. Session-/Token-Speicherung der Web-App härten

**Ist-Zustand:**
- Access- und Refresh-Token liegen im `localStorage`.

**Verbesserung:**
- mittelfristig Umstellung auf HttpOnly Secure Cookies prüfen,
- mindestens klar dokumentieren, dass current design browser-storage-basiert ist.

**Impact:** hoch
**Aufwand:** mittel bis hoch

## Priorität 2 — Hoher Impact, mittlerer Aufwand

### 5. Pending OAuth Logins persistent oder restart-resilient machen

**Ist-Zustand:**
- `pendingOAuthLogins` liegt nur in Memory.
- Backend-Neustart zerstört laufende Flows.

**Verbesserung:**
- Pending-Status temporär persistent speichern,
  oder
- Flow-Neustart/Recovery explizit unterstützen.

**Impact:** hoch bei produktivem Betrieb
**Aufwand:** mittel

### 6. Polling robuster machen

**Ist-Zustand:**
- Frontend pollt 120-mal im 1-Sekunden-Takt.

**Verbesserung:**
- Backoff oder Server-Sent Events/WebSocket prüfen,
- differenzierte Statusinfos (`pending`, `awaiting-manual-input`, `token-exchange`, `completed`, `error`) anbieten.

**Impact:** mittel-hoch
**Aufwand:** mittel

### 7. 401-/Refresh-Dedupe im Frontend einführen

**Ist-Zustand:**
- `useApi.ts` refreshes bei 401 direkt pro Request.
- Es gibt keine sichtbare globale Dedup-Logik für parallele Requests.

**Verbesserung:**
- ein globales `refreshPromise` einführen,
- parallele 401er auf denselben laufenden Refresh warten lassen.

**Impact:** hoch bei parallelen Requests
**Aufwand:** mittel

## Priorität 3 — Mittlerer Impact, niedriger bis mittlerer Aufwand

### 8. Serverseitige Logout-/Revocation-Strategie für App-Refresh-Tokens verbessern

**Ist-Zustand:**
- `/api/auth/logout` ist laut Kommentar rein stateless; Client verwirft Token.
- Refresh Tokens bleiben bis Ablauf gültig, solange Signatur passt.

**Verbesserung:**
- Refresh-Token-Rotation / Revocation-Liste / Token-Versionierung prüfen.

**Impact:** mittel
**Aufwand:** mittel bis hoch

### 9. OAuth-Statusobjekt um technische Details ergänzen

**Ist-Zustand:**
- Statusroute liefert nur `pending | completed | error`.

**Verbesserung:**
- zusätzliche Felder wie:
  - `stage`,
  - `manualInputRequired`,
  - `expiresAt`,
  - `startedAt`.

**Impact:** mittel
**Aufwand:** niedrig bis mittel

### 10. Sicherheitsrelevante Eigenschaften des externen OAuth-Adapters dokumentieren

**Ist-Zustand:**
- PKCE/State/Nonce/Callback-Validierung sind im Repo nicht sichtbar.

**Verbesserung:**
- explizite Dokumentation nahe `routes/providers.ts` oder im Docs-Ordner:
  - verwendet der Adapter PKCE?
  - wie wird `state` validiert?
  - wie funktioniert manual callback input?

**Impact:** mittel
**Aufwand:** niedrig

---

## 7. Bewertung nach Themenfeld

| Themenfeld | ClaudeCode | OpenAgent |
|---|---|---|
| Architektur | Vollständige OAuth-Architektur im Repo | App-JWT + Provider-OAuth-Orchestrierung |
| Einstiegspunkte | CLI, TUI, Headless-Control, Service-Layer | Web-Login + Admin-Provider-Dialog |
| OAuth-Flow | Selbst implementierter Auth-Code-Flow | Delegation an externen Adapter |
| Redirect/Callback | Loopback-Listener + manuell | Adapter/Browser + Polling + manueller String-Fallback |
| PKCE/State | Sichtbar vorhanden | Im Repo nicht belegbar |
| Tokenaustausch | Sichtbar implementiert | Nicht im Repo sichtbar |
| Persistenz | Secure Storage + Cache-Management | localStorage (App), encrypted provider creds (Core) |
| Refresh | Sehr robust, dedupliziert, gelockt | Einfach, funktional, weniger robust sichtbar |
| 401-Handling | Sehr ausgereift | Standardmäßig, einfach |
| Sicherheitsmaßnahmen | Stark, sichtbar, auditierbar | Teilweise sichtbar, OAuth-Details Black Box |
| UX/Fallbacks | Sehr ausgeprägt | Solide, aber einfacher |
| Sonderfälle | Viele explizit behandelt | Weniger sichtbar behandelt |

---

## 8. Fundierte Schlussbewertung

## Wenn man "Login der Anwendung" vergleicht

- **OpenAgent** benutzt ein klassisches JWT-Login-System mit Refresh.
- **ClaudeCode** benutzt primär OAuth-basierte Auth gegen Anthropic.

Das ist kein 1:1 identischer Auth-Typ.

## Wenn man "OAuth-Implementierung" vergleicht

- **ClaudeCode** ist klar robuster und vollständiger innerhalb des eigenen Repos.
- **OpenAgent** ist für Provider-OAuth eher ein koordinierender Wrapper um eine externe OAuth-Library.

## Wichtigster belastbarer Unterschied

ClaudeCode zeigt im vorhandenen Code explizit:
- PKCE,
- `state`,
- Redirect-Handling,
- Token-Exchange,
- Refresh,
- Persistenz,
- 401-Recovery.

OpenAgent zeigt im vorhandenen Code explizit vor allem:
- Flow-Orchestrierung,
- Pending-Login-State,
- Frontend-Polling,
- Speicherung fertiger OAuth-Credentials,
- aber **nicht** die internen Sicherheits- und Protokolldetails des OAuth-Flows selbst.

---

## 9. Kompakte Zusammenfassung für Nutzererklärung

ClaudeCode hat eine deutlich umfassendere und robustere OAuth-Implementierung direkt im eigenen Code: mit PKCE, State-Validierung, lokalem Callback-Server, manuellem Fallback, sauberem Tokenaustausch, sicherer Speicherung, dedupliziertem Refresh und ausgefeilter 401-Fehlerbehandlung. OpenAgent dagegen nutzt für den eigentlichen App-Login kein OAuth, sondern ein eigenes JWT-/Refresh-Token-System. OAuth wird dort vor allem genutzt, um externe Modellanbieter anzubinden; dieser Flow wird im Repo selbst aber nur orchestriert und an eine externe Library delegiert. Dadurch ist OpenAgent funktional, aber in Sachen Auditierbarkeit, sichtbarer Sicherheitsmechanismen und Robustheit des OAuth-Flows klar schwächer als ClaudeCode. Die wichtigsten Verbesserungen für OpenAgent wären: manuellen OAuth-Callback sauber validieren, laufende OAuth-Logins an den konkreten User binden, 401-Refresh deduplizieren, `localStorage`-basierte Session-Speicherung härten und die Sicherheitsgarantien des externen OAuth-Adapters explizit dokumentieren.
