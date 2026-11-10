# Čarotéka — Frontend API & Integrační Příručka
## Obsah
1. [Autentizace a bezpečnostní architektura](#2-autentizace-a-bezpečnostní-architektura)
   - [2.1 JWT Access Token & Refresh Token rotace (SEC-08)](#21-jwt-access-token--refresh-token-rotace-sec-08)
   - [2.2 Proč `credentials: 'include'` je povinné](#22-proč-credentials-include-je-povinné)
   - [2.3 Uživatelské role (RBAC)](#23-uživatelské-role-rbac)
3. [Globální chování API, chyby a stránkování](#3-globální-chování-api-chyby-a-stránkování)
   - [3.1 Formát chyb (RFC 7807 / RFC 9457 ProblemDetails)](#31-formát-chyb-rfc-7807--rfc-9457-problemdetails)
   - [3.2 Omezení četnosti požadavků (Rate Limiting)](#32-omezení-četnosti-požadavků-rate-limiting)
   - [3.3 Standardní stránkování (PagedResult)](#33-standardní-stránkování-pagedresult)
4. [Práce s `dataJson` & 5e.tools kompatibilita](#4-práce-s-datajson--5etools-kompatibilita)
   - [4.1 Co je `dataJson` a jak s ním pracovat](#41-co-je-datajson-a-jak-s-ním-pracovat)
   - [4.2 Značkovací tagy 5e.tools a jejich parsování](#42-značkovací-tagy-5etools-a-jejich-parsování)
   - [4.3 Ukázková funkce parseru pro frontend](#43-ukázková-funkce-parseru-pro-frontend)
5. [Přehled endpointů a přesné datové modely](#5-přehled-endpointů-a-přesné-datové-modely)
   - [5.1 Autentizace (`/api/v1/auth`)](#51-autentizace-apiv1auth)
   - [5.2 Uživatelé a profily (`/api/v1/users`)](#52-uživatelé-a-profily-apiv1users)
   - [5.3 Postavy (`/api/v1/characters`)](#53-postavy-apiv1characters)
   - [5.4 Kampaně (`/api/v1/campaigns`)](#54-kampaně-apiv1campaigns)
   - [5.5 DM Poznámky ke kampani (`/api/v1/campaigns/{campaignId}/dm-notes`)](#55-dm-poznámky-ke-kampani-apiv1campaignscampaigniddm-notes)
   - [5.6 Databáze pravidel / Kompendium (10 kolekcí)](#56-databáze-pravidel--kompendium-10-kolekcí)
     - [5.6.1 Kouzla (`/api/v1/spells`)](#561-kouzla-apiv1spells)
     - [5.6.2 Bestiář nestvůr (`/api/v1/monsters`)](#562-bestiář-nestvůr-apiv1monsters)
     - [5.6.3 Předměty a vybavení (`/api/v1/items`)](#563-předměty-a-vybavení-apiv1items)
     - [5.6.4 Rasy (`/api/v1/races`)](#564-rasy-apiv1races)
     - [5.6.5 Povolání (`/api/v1/classes`)](#565-povolání-apiv1classes)
     - [5.6.6 Odbornosti (`/api/v1/feats`)](#566-odbornosti-apiv1feats)
     - [5.6.7 Zázemí (`/api/v1/backgrounds`)](#567-zázemí-apiv1backgrounds)
     - [5.6.8 Stavy (`/api/v1/conditions`)](#568-stavy-apiv1conditions)
     - [5.6.9 Dobrodružství (`/api/v1/adventures`)](#569-dobrodružství-apiv1adventures)
     - [5.6.10 Knihy pravidel (`/api/v1/books`)](#5610-knihy-pravidel-apiv1books)
   - [5.7 Globální vyhledávač & Filtry (`/api/v1/search`)](#57-globální-vyhledávač--filtry-apiv1search)
   - [5.8 Generátory obsahu (`/api/v1/generators`)](#58-generátory-obsahu-apiv1generators)
   - [5.9 Systémové informace & Stav (`/api/v1/changelog`, `/api/v1/status`)](#59-systémové-informace--stav-apiv1changelog-apiv1status)
1. [Kompletní TypeScript definice typů (Copy-Paste pro frontend)](#6-kompletní-typescript-definice-typů-copy-paste-pro-frontend)
## 2. Autentizace a bezpečnostní architektura

### 2.1 JWT Access Token & Refresh Token rotace (SEC-08)
Čarotéka využívá dvoustupňový bezpečnostní model:
1. **Access Token (JWT):**
   - Vrací se v JSON těle odpovědi při registraci, přihlášení a obnovení (`accessToken`).
   - Má krátkou životnost (např. 15–60 minut).
   - Ukládá se **výhradně do paměti klienta** (React state, Pinia store, memory cache).
   - **Nikdy neukládejte access token do `localStorage` ani `sessionStorage`!** (Ochrana proti XSS krádeži).
   - Předává se v HTTP hlavičce: `Authorization: Bearer <accessToken>`.
2. **Refresh Token (HttpOnly Cookie):**
   - V souladu s bezpečnostním standardem SEC-08 se refresh token **nikdy nevrací v JSON těle** odpovědi.
   - Server jej nastavuje v bezpečné HTTP hlavičce:
     ```http
     Set-Cookie: refreshToken=<token>; Path=/; HttpOnly; Secure; SameSite=Strict; Max-Age=604800
     ```
   - Má platnost 7 dní a automaticky se obnovuje při každém zavolání `/api/v1/auth/refresh`.

```
[ Frontend Klient ]                                     [ Backend API ]
        |                                                     |
        |--- 1. POST /api/v1/auth/login --------------------->|
        |<-- 2. 200 OK { accessToken, email, user... } -------|
        |       + Set-Cookie: refreshToken (HttpOnly, Strict) |
        |                                                     |
        |--- 3. GET /api/v1/characters (Bearer token) ------->|
        |<-- 4. 200 OK [ data postav ] -----------------------|
        |                                                     |
        |    ... po vypršení platnosti Access Tokenu ...      |
        |                                                     |
        |--- 5. GET /api/v1/characters ---------------------->|
        |<-- 6. 401 Unauthorized -----------------------------|
        |                                                     |
        |--- 7. POST /api/v1/auth/refresh (auto cookie) ----->|
        |<-- 8. 200 OK { new accessToken } + nová cookie -----|
        |                                                     |
        |--- 9. Opakování původního požadavku --------------->|
        |<-- 10. 200 OK [ data ] -----------------------------|
```

### 2.2 Proč `credentials: 'include'` je povinné
Jelikož je refresh token v `HttpOnly` cookie, frontend jej nemůže číst v JavaScriptu. Aby prohlížeč cookie odeslal na server, musíte u každého volání povolit pověření:
- **Fetch API:** `fetch(url, { credentials: 'include', ... })`
- **Axios:** `axios.defaults.withCredentials = true` nebo `apiClient.create({ withCredentials: true })`

### 2.3 Uživatelské role (RBAC)
Při přihlášení obdržíte v poli `roles` výčet rolí uživatele:
- `Player`: Běžný hráč. Může vytvářet a upravovat své postavy, připojovat se do kampaní.
- `DungeonMaster`: Pán jeskyně. Může vytvářet kampaně, spravovat úkoly, zápisy a soukromé DM poznámky. *(Pozor: název role je `DungeonMaster`, nikoli zkratka `DM`!)*
- `Homebrewer`: Uživatel oprávněný vytvářet a publikovat vlastní komunitní obsah.
- `Moderator`: Moderátor schvalující komunitní obsah do veřejného kompendia.
- `Admin`: Správce platformy s plným přístupem ke všem datům a systémovým nastavením.

---

## 3. Globální chování API, chyby a stránkování

### 3.1 Formát chyb (RFC 7807 / RFC 9457 ProblemDetails)
Všechny chybové odpovědi API používají formát `application/problem+json`:

**Obecná doménová chyba (400 / 404 / 403):**
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Porušení doménových pravidel",
  "status": 400,
  "detail": "Postavu nelze přiřadit ke kampani, jejímž nejste aktivním členem.",
  "instance": "/api/v1/campaigns/join"
}
```

**Validační chyba formuláře (400 Bad Request):**
Pokud požadavek neprojde validací vstupních polí, pole `errors` obsahuje slovník názvů polí a pole chybových hlášek pro snadné zobrazení u formulářových prvků:
```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": ["Pole Email musí obsahovat platnou e-mailovou adresu."],
    "Password": ["Nové heslo musí mít alespoň 8 znaků."]
  }
}
```

### 3.2 Omezení četnosti požadavků (Rate Limiting)
Pro ochranu před brute-force a DoS útoky jsou aplikovány rate limity (při překročení vrátí HTTP `429 Too Many Requests`):
- **Autentizace (`AuthPolicy`):** `/api/v1/auth/login`, `/register`, `/refresh` — **max 10 požadavků za minutu** na IP adresu.
- **Připojení ke kampani (`JoinCampaignPolicy`):** `/api/v1/campaigns/join` — **max 5 požadavků za minutu** na IP adresu (ochrana zvacích kódů).

### 3.3 Standardní stránkování (PagedResult)
Všechny seznamové endpointy používají parametry:
- `page`: číslo stránky (číslováno od 1, výchozí hodnota `1`)
- `pageSize`: počet prvků na stránku (výchozí hodnota `20`, maximální hodnota `50` nebo `100` u kompendia)

**Struktura obálky `PagedResult<T>`:**
```json
{
  "items": [ /* pole prvků typu T */ ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 85,
  "totalPages": 5
}
```

---

## 4. Práce s `dataJson` & 5e.tools kompatibilita

### 4.1 Co je `dataJson` a jak s ním pracovat
Všechny entity kompendia (kouzla, nestvůry, předměty, rasy, třídy atd.) obsahují pole `dataJson`. Toto pole je **serializovaný JSON řetězec** obsahující kompletní, bohatě strukturovaná data z originálního 5e.tools schématu.

**Proč je to řetězec a ne rozbalený objekt?**
1. Databáze PostgreSQL ukládá kompletní nestrukturované záznamy efektivně jako JSON text.
2. Klient si z databáze vytáhne jen ta pole, která potřebuje, a `JSON.parse(entity.dataJson)` zavolá pouze na detailu položky nebo při vykreslování tooltipu, což šetří výpočetní výkon i paměť mobilních zařízení.

### 4.2 Značkovací tagy 5e.tools a jejich parsování
Texty v polích `entries`, akcích potvor a popisech předmětů obsahují formátovací značky ve složených závorkách:
- `{@damage 8d6}` → Hod na zranění 8k6 (lze kliknout a hodit virtuální kostkou).
- `{@dice 1d20+5}` → Obecný hod kostkou.
- `{@spell fireball}` → Odkaz na jiné kouzlo v kompendiu.
- `{@creature goblin}` → Odkaz na záznam v bestiáři.
- `{@item dagger}` → Odkaz na předmět v kompendiu.
- `{@condition poisoned}` → Herní stav (otrávený).
- `{@dc 15}` → Obtížnost záchranného hodu DC 15.
- `{@b text}` → Tučný text (**text**).
- `{@i text}` → Kurzíva (*text*).

### 4.3 Ukázková funkce parseru pro frontend
Následující pomocná funkce převede 5e.tools tagy na čistý text nebo HTML/React elementy:

```typescript
// src/utils/format5eTools.ts
export function render5eText(rawText: string): string {
  if (!rawText) return '';
  return rawText
    // Tagy na zranění a kostky: {@damage 8d6} -> 8d6
    .replace(/\{@(?:damage|dice)\s+([^}]+)\}/gi, '$1')
    // Odkazy na kouzla, potvory, předměty: {@spell fireball} -> Fireball
    .replace(/\{@(?:spell|creature|item|condition|feat|race|class)\s+([^}|]+)(?:\|[^}]+)?\}/gi, '$1')
    // Obtížnost: {@dc 15} -> DC 15
    .replace(/\{@dc\s+([^}]+)\}/gi, 'DC $1')
    // Formátování textu: {@b tučně} -> **tučně**
    .replace(/\{@b\s+([^}]+)\}/gi, '**$1**')
    .replace(/\{@i\s+([^}]+)\}/gi, '*$1*');
}
```

---

## 5. Přehled endpointů a přesné datové modely

### 5.1 Autentizace (`/api/v1/auth`)

#### POST `/api/v1/auth/register`
Registruje nového uživatele, vytvoří profil, vydá JWT token a nastaví `refreshToken` cookie.
- **Oprávnění:** Anonymní (Rate limit: 10/min)
- **Request Body (`RegisterRequest`):**
```json
{
  "email": "hrac@caroteka.cz",
  "userName": "GeraltZRivie",
  "password": "MojeSuperHeslo123!",
  "passwordConfirmation": "MojeSuperHeslo123!",
  "avatarUrl": "/static/avatars/classes/fighter.webp"
}
```
- **Response (201 Created — `AuthResponse`):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIzZmE4NWY2NC01NzE3LTQ1NjItYjNmYy0yYzk2M2Y2NmFmYTYiLCJlbWFpbCI6ImhyYWNAY2Fyb3Rla2EuY3oiLCJuYW1lIjoiR2VyYWx0WlJpdmllIiwicm9sZSI6WyJQbGF5ZXIiXX0.abc...",
  "accessTokenExpiresAt": "2026-11-10T18:30:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "avatarUrl": "/static/avatars/classes/fighter.webp",
  "roles": [
    "Player"
  ]
}
```
*(Hlavička Set-Cookie nastaví zabezpečenou cookie `refreshToken`).*

---

#### POST `/api/v1/auth/login`
Přihlášení uživatele e-mailem a heslem.
- **Oprávnění:** Anonymní (Rate limit: 10/min)
- **Request Body (`LoginRequest`):**
```json
{
  "email": "hrac@caroteka.cz",
  "password": "MojeSuperHeslo123!"
}
```
- **Response (200 OK — `AuthResponse`):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "accessTokenExpiresAt": "2026-11-10T18:30:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "avatarUrl": "/static/avatars/classes/fighter.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

#### POST `/api/v1/auth/refresh`
Vydá nový JWT access token a provede rotaci refresh tokenu v cookie.
- **Oprávnění:** Anonymní (Rate limit: 10/min)
- **Request Body:** `{}` (prohlížeč cookie přiloží automaticky)
- **Response (200 OK — `AuthResponse`):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.novyToken...",
  "accessTokenExpiresAt": "2026-11-10T19:30:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "avatarUrl": "/static/avatars/classes/fighter.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

#### POST `/api/v1/auth/logout`
Zneplatní aktivní refresh token a vymaže cookie v prohlížeči.
- **Oprávnění:** `[Authorize]`
- **Hlavička:** `Authorization: Bearer <accessToken>`
- **Request Body:** `{}`
- **Response (204 No Content)**

---

#### GET `/api/v1/auth/me`
Rychlé ověření platnosti přihlášení a načtení profilu aktuálního uživatele.
- **Oprávnění:** `[Authorize]`
- **Response (200 OK — `UserProfileDto`):**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "displayName": "Geralt z Rivie",
  "avatarUrl": "/static/avatars/classes/fighter.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

### 5.2 Uživatelé a profily (`/api/v1/users`)

#### GET `/api/v1/users/avatars`
Vrátí spravovaný katalog předpřipravených profilových obrázků. Vlastní nahrávání libovolných souborů není povoleno z bezpečnostních a licenčních důvodů.
- **Oprávnění:** Veřejné (anonymní)
- **Query parametry:**
  - `category` (volitelné): např. `"Povolání"`, `"Rasy"`, `"Monstra"`, `"Ikony"`.
  - `search` (volitelné): textové hledání v názvu a tazích (např. `"cleric"` nebo `"drak"`).
- **Response (200 OK — `List<ProfileAvatarDto>`):**
```json
[
  {
    "id": "cleric",
    "name": "Kněz",
    "category": "Povolání",
    "imageUrl": "/static/avatars/classes/cleric.webp",
    "thumbnailUrl": "/static/avatars/classes/cleric-thumb.webp",
    "tags": ["cleric", "knez", "svaty", "lecitel", "vira"]
  },
  {
    "id": "wizard",
    "name": "Kouzelník",
    "category": "Povolání",
    "imageUrl": "/static/avatars/classes/wizard.webp",
    "thumbnailUrl": "/static/avatars/classes/wizard-thumb.webp",
    "tags": ["wizard", "kouzelnik", "magie", "svitky"]
  },
  {
    "id": "red-dragon",
    "name": "Rudý drak",
    "category": "Monstra",
    "imageUrl": "/static/avatars/monsters/red-dragon.webp",
    "thumbnailUrl": "/static/avatars/monsters/red-dragon-thumb.webp",
    "tags": ["dragon", "drak", "ohen", "boss"]
  }
]
```

---

#### GET `/api/v1/users/me`
Získá kompletní profil aktuálně přihlášeného uživatele.
- **Oprávnění:** `[Authorize]`
- **Response (200 OK — `UserProfileDto`):**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "displayName": "Geralt z Rivie",
  "avatarUrl": "/static/avatars/classes/fighter.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

#### PUT `/api/v1/users/me/avatar`
Nastaví nebo změní profilový obrázek uživatele výběrem ze spravovaného fondu.
- **Oprávnění:** `[Authorize]`
- **Request Body (`UpdateAvatarRequest`):**
```json
{
  "avatarIdOrUrl": "cleric"
}
```
*(Lze předat ID avatara jako `"cleric"` nebo celou relativní cestu `"/static/avatars/classes/cleric.webp"`).*
- **Response (200 OK — `UserProfileDto`):**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "hrac@caroteka.cz",
  "displayName": "Geralt z Rivie",
  "avatarUrl": "/static/avatars/classes/cleric.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

#### PUT `/api/v1/users/me/password`
Změní heslo přihlášeného uživatele. Vyžaduje potvrzení současného hesla a automaticky zneplatní předchozí aktivní refresh tokeny.
- **Oprávnění:** `[Authorize]`
- **Request Body (`ChangePasswordRequest`):**
```json
{
  "currentPassword": "MojeSuperHeslo123!",
  "newPassword": "NoveBezpecneHeslo2026*"
}
```
- **Response (204 No Content)**

---

#### PUT `/api/v1/users/me/email`
Změní e-mailovou adresu přihlášeného uživatele po ověření současným heslem.
- **Oprávnění:** `[Authorize]`
- **Request Body (`ChangeEmailRequest`):**
```json
{
  "newEmail": "novy.email@caroteka.cz",
  "currentPassword": "NoveBezpecneHeslo2026*"
}
```
- **Response (200 OK — `UserProfileDto`):**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "email": "novy.email@caroteka.cz",
  "displayName": "Geralt z Rivie",
  "avatarUrl": "/static/avatars/classes/cleric.webp",
  "roles": [
    "Player",
    "DungeonMaster"
  ]
}
```

---

### 5.3 Postavy (`/api/v1/characters`)

> [!IMPORTANT]
> **Architektura proti neoprávněné manipulaci (Mass Assignment — SEC-04):**  
> Herní statistiky (Level, Životy, AC, XP) nelze libovolně měnit přes profilový update `PUT /{id}`.  
> - Úroveň se zvyšuje přes `POST /{id}/level-up`.
> - Životy v boji se mění přes `PATCH /{id}/hit-points`.
> - Inventář a měna se upravují přes `PATCH /{id}/inventory`.
> - Kouzelnická kniha se spravuje přes `PATCH /{id}/spells`.
> - Mechaniky (AC, XP, rychlost) se mění přes `PUT /{id}/mechanics` nebo `PATCH /{id}/mechanics`.

#### GET `/api/v1/characters`
Vrátí stránkovaný přehled všech postav přihlášeného uživatele.
- **Oprávnění:** `[Authorize]`
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK — `PagedResult<CharacterListItemDto>`):**
```json
{
  "items": [
    {
      "id": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
      "name": "Thorin Pavéza",
      "className": "Fighter",
      "raceName": "Dwarf",
      "level": 3,
      "currentHitPoints": 28,
      "maxHitPoints": 28,
      "campaignId": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
      "isPublic": true,
      "updatedAt": "2026-11-10T12:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

---

#### GET `/api/v1/characters/{id}`
Vrátí kompletní detailní list postavy včetně JSONB struktur `sheetData`, `inventory`, `spellbook` a kompatibilních aliasů pro frontend.
- **Oprávnění:** `[Authorize]` (Vlastník nebo DM v dané kampani)
- **Response (200 OK — `CharacterDetailDto`):**
```json
{
  "id": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
  "ownerUserId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "campaignId": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
  "name": "Thorin Pavéza",
  "className": "Fighter",
  "subclass": "Champion",
  "subclassName": "Champion",
  "raceName": "Mountain Dwarf",
  "background": "Soldier",
  "alignment": "Lawful Good",
  "level": 3,
  "experiencePoints": 900,
  "currentHitPoints": 28,
  "maxHitPoints": 28,
  "temporaryHitPoints": 0,
  "armorClass": 18,
  "initiative": 1,
  "speed": 25,
  "hitDice": "3d10",
  "proficiencyBonus": 2,
  "abilityScores": {
    "strength": 16,
    "dexterity": 12,
    "constitution": 16,
    "intelligence": 10,
    "wisdom": 13,
    "charisma": 8
  },
  "stats": {
    "strength": 16,
    "dexterity": 12,
    "constitution": 16,
    "intelligence": 10,
    "wisdom": 13,
    "charisma": 8
  },
  "deathSaves": {
    "successes": 0,
    "failures": 0
  },
  "isPublic": true,
  "sheetData": {
    "skills": [
      { "name": "Athletics", "isProficient": true, "hasExpertise": false, "miscModifier": 0 },
      { "name": "Intimidation", "isProficient": true, "hasExpertise": false, "miscModifier": 0 }
    ],
    "savingThrows": [
      { "ability": "Strength", "isProficient": true },
      { "ability": "Constitution", "isProficient": true }
    ],
    "languages": ["Common", "Dwarvish"],
    "features": [
      { "name": "Second Wind", "description": "Obnoví 1d10 + úroveň HP jako bonusovou akci.", "source": "Fighter 1" },
      { "name": "Action Surge", "description": "Jedna dodatečná akce za krátký odpočinek.", "source": "Fighter 2" }
    ],
    "activeConditions": [],
    "resources": { "SecondWind": 1, "ActionSurge": 1 },
    "personalityTraits": "Nikdy neustupuji před nebezpečím.",
    "ideals": "Čest klanu nade vše.",
    "bonds": "Můj rodný štít je památkou na padlé předky.",
    "flaws": "Těžko důvěřuji elfům."
  },
  "inventory": {
    "items": [
      {
        "id": "a1b2c3d4-0000-0000-0000-000000000001",
        "name": "Bitevní sekera",
        "itemName": "Bitevní sekera",
        "quantity": 1,
        "weightPerUnit": 4.0,
        "weight": 4.0,
        "isEquipped": true,
        "isAttuned": false,
        "description": "1d8 (obouručně 1d10) sečné zranění",
        "notes": "Vynikající trpasličí kovářská práce"
      },
      {
        "id": "a1b2c3d4-0000-0000-0000-000000000002",
        "name": "Plátová zbroj",
        "itemName": "Plátová zbroj",
        "quantity": 1,
        "weightPerUnit": 55.0,
        "weight": 55.0,
        "isEquipped": true,
        "isAttuned": false,
        "description": "AC 18, nevýhoda na Nenápadnost",
        "notes": null
      }
    ],
    "platinumPieces": 0,
    "goldPieces": 35,
    "electrumPieces": 0,
    "silverPieces": 15,
    "copperPieces": 40
  },
  "spellbook": {
    "spells": [],
    "spellSlots": [],
    "spellcastingAbility": "None",
    "isConcentrating": false,
    "concentratingOnSpell": null
  },
  "spells": [],
  "createdAt": "2026-11-01T10:00:00Z",
  "updatedAt": "2026-11-10T12:00:00Z"
}
```

---

#### POST `/api/v1/characters`
Vytvoří novou postavu na 1. úrovni.
- **Oprávnění:** `[Authorize]`
- **Request Body (`CreateCharacterRequest`):**
```json
{
  "name": "Legolas",
  "className": "Ranger",
  "subclass": "Hunter",
  "raceName": "Elf",
  "background": "Outlander",
  "alignment": "Chaotic Good",
  "abilityScores": {
    "strength": 10,
    "dexterity": 17,
    "constitution": 14,
    "intelligence": 12,
    "wisdom": 15,
    "charisma": 8
  },
  "maxHitPoints": 12
}
```
- **Response (201 Created — `CharacterDetailDto`)**

---

#### PUT `/api/v1/characters/{id}`
Aktualizuje profil postavy, osobnost a JSONB struktury (deník, výbavu, rysy).
- **Oprávnění:** `[Authorize]` (Pouze vlastník)
- **Limit těla:** Striktní limit **512 KB** (`RequestSizeLimit`).
- **Request Body (`UpdateCharacterProfileRequest`):**
```json
{
  "name": "Legolas Zelenolesý",
  "background": "Outlander",
  "alignment": "Chaotic Good",
  "isPublic": true,
  "sheetData": {
    "skills": [
      { "name": "Stealth", "isProficient": true, "hasExpertise": false, "miscModifier": 0 },
      { "name": "Perception", "isProficient": true, "hasExpertise": true, "miscModifier": 0 }
    ],
    "savingThrows": [
      { "ability": "Dexterity", "isProficient": true },
      { "ability": "Strength", "isProficient": true }
    ],
    "languages": ["Common", "Elvish"],
    "features": [
      { "name": "Darkvision", "description": "Vidění ve tmě na 60 stop.", "source": "Elf" }
    ],
    "activeConditions": [],
    "resources": { "FavoredEnemy": 2 },
    "personalityTraits": "Klidný a bystrý pozorovatel.",
    "ideals": "Přírodní rovnováha.",
    "bonds": "Můj rodný hvozd.",
    "flaws": "Těžko odpouštím křivdy."
  },
  "inventory": {
    "items": [
      {
        "name": "Dlouhý luk",
        "quantity": 1,
        "weightPerUnit": 2.0,
        "isEquipped": true,
        "description": "1d8 bodné zranění, dostřel 150/600"
      }
    ],
    "platinumPieces": 0,
    "goldPieces": 50,
    "electrumPieces": 0,
    "silverPieces": 10,
    "copperPieces": 0
  },
  "spellbook": {
    "spells": [
      { "name": "Hunter's Mark", "level": 1, "isPrepared": true, "school": "Divination" }
    ],
    "spellSlots": [
      { "level": 1, "total": 2, "used": 0 }
    ],
    "spellcastingAbility": "Wisdom",
    "isConcentrating": false,
    "concentratingOnSpell": null
  }
}
```
- **Response (200 OK — `CharacterDetailDto`)**

---

#### POST `/api/v1/characters/{id}/level-up`
Zvýší úroveň postavy o 1 a navýší maximální životy.
- **Oprávnění:** `[Authorize]` (Vlastník)
- **Request Body (`LevelUpCharacterRequest`):**
```json
{
  "additionalHitPoints": 8
}
```
- **Response (200 OK — `CharacterDetailDto` s navýšeným levelem)**

---

#### PUT `/api/v1/characters/{id}/mechanics` nebo PATCH `/api/v1/characters/{id}/mechanics`
Upraví mechanické vlastnosti postavy (XP, AC, iniciativa, rychlost, hit dice).
- **Oprávnění:** `[Authorize]` (Vlastník)
- **Request Body (`UpdateCharacterMechanicsRequest`):**
```json
{
  "experiencePoints": 1500,
  "armorClass": 16,
  "initiative": 3,
  "speed": 35,
  "hitDice": "3d10"
}
```
- **Response (200 OK — `CharacterDetailDto`)**

---

#### PATCH `/api/v1/characters/{id}/hit-points`
Rychlá úprava životů během souboje.
- **Oprávnění:** Vlastník postavy NEBO Pán jeskyně kampaně (pokud hráč povolil `allowsDmStatEdits`).
- **Request Body (`UpdateHitPointsRequest`):**
```json
{
  "currentHitPoints": 14,
  "temporaryHitPoints": 5,
  "deathSaves": {
    "successes": 0,
    "failures": 0
  }
}
```
- **Response (200 OK — `CharacterDetailDto`)**

---

#### PATCH `/api/v1/characters/{id}/inventory`
Atomická aktualizace inventáře a mincí postavy (přidání lupu, utracení zlaťáků, změna vybavení).
- **Oprávnění:** `[Authorize]` (Vlastník)
- **Request Body (`InventoryDataDto`):**
```json
{
  "items": [
    {
      "name": "Léčivý lektvar",
      "quantity": 2,
      "weightPerUnit": 0.5,
      "isEquipped": false,
      "description": "Obnoví 2d4 + 2 životů."
    }
  ],
  "platinumPieces": 1,
  "goldPieces": 45,
  "electrumPieces": 0,
  "silverPieces": 12,
  "copperPieces": 80
}
```
- **Response (200 OK — `CharacterDetailDto`)**

---

#### PATCH `/api/v1/characters/{id}/spells`
Atomická aktualizace knihy kouzel a slotů postavy (příprava kouzel, vyčerpání slotů, soustředění).
- **Oprávnění:** `[Authorize]` (Vlastník)
- **Request Body (`SpellbookDataDto`):**
```json
{
  "spells": [
    { "name": "Fireball", "level": 3, "isPrepared": true, "school": "Evocation" },
    { "name": "Misty Step", "level": 2, "isPrepared": true, "school": "Conjuration" }
  ],
  "spellSlots": [
    { "level": 1, "total": 4, "used": 2 },
    { "level": 2, "total": 3, "used": 1 },
    { "level": 3, "total": 2, "used": 0 }
  ],
  "spellcastingAbility": "Intelligence",
  "isConcentrating": false,
  "concentratingOnSpell": null
}
```
- **Response (200 OK — `CharacterDetailDto`)**

---

#### DELETE `/api/v1/characters/{id}`
Trvale smaže postavu přihlášeného uživatele.
- **Oprávnění:** `[Authorize]` (Vlastník)
- **Response (204 No Content)**

---

### 5.4 Kampaně (`/api/v1/campaigns`)

#### GET `/api/v1/campaigns`
Vrátí seznam kampaní volajícího uživatele (kde je buď Pánem jeskyně, nebo aktivním hráčem).
- **Oprávnění:** `[Authorize]`
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK — `PagedResult<CampaignListItemDto>`):**
```json
{
  "items": [
    {
      "id": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
      "name": "Ztracený důl Fandelveru",
      "dungeonMasterUserId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "memberCount": 4,
      "callerRole": "DungeonMaster",
      "isActive": true,
      "updatedAt": "2026-11-10T15:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

---

#### GET `/api/v1/campaigns/{id}`
Vrátí detail kampaně včetně členů, úkolů a zápisů ze sezení.
- **Oprávnění:** `[Authorize]` (Aktivní člen kampaně nebo DM)
- **Důležité pravidlo ochrany:** Pole `inviteCode` se vrací **pouze Pánu jeskyně** (`DungeonMaster`). Běžným hráčům se vrací `null`. Úkoly v poli `quests` jsou pro hráče automaticky filtrovány na ty s `isVisibleToPlayers: true`.
- **Response (200 OK — `CampaignDetailDto`):**
```json
{
  "id": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
  "name": "Ztracený důl Fandelveru",
  "description": "Klasická kampaň na Mečovém pobřeží.",
  "setting": "Forgotten Realms",
  "dungeonMasterUserId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "dungeonMasterName": "GeraltZRivie",
  "inviteCode": "PHANDALIN-2026",
  "isActive": true,
  "members": [
    {
      "userId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "userName": "Strider",
      "memberRole": "Player",
      "characterId": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
      "characterName": "Thorin Pavéza",
      "allowsDmStatEdits": true,
      "joinedAt": "2026-11-02T18:00:00Z"
    }
  ],
  "quests": [
    {
      "id": "c1d2e3f4-a5b6-7890-1234-56789abcdef0",
      "title": "Záchrana Gundrena Rockseekera",
      "description": "Gundren byl zajat gobliny z kmene Cragmaw.",
      "status": "Active",
      "isVisibleToPlayers": true,
      "rewardDescription": "50 gp a podíl v dole",
      "rewardGoldPieces": 50,
      "createdAt": "2026-11-03T19:00:00Z",
      "updatedAt": "2026-11-03T19:00:00Z"
    }
  ],
  "sessionLogs": [
    {
      "id": "d1e2f3a4-b5c6-7890-1234-56789abcdef1",
      "sequenceNumber": 1,
      "recap": "Družina dorazila k úkrytu Cragmaw a osvobodila Sildara Hallwintera.",
      "sessionDate": "2026-11-05",
      "participantCharacterNames": ["Thorin Pavéza", "Legolas"],
      "createdAt": "2026-11-05T22:30:00Z"
    }
  ],
  "createdAt": "2026-11-01T12:00:00Z",
  "updatedAt": "2026-11-10T15:00:00Z"
}
```

---

#### POST `/api/v1/campaigns`
Vytvoří novou kampaň a vygeneruje unikátní zvací kód.
- **Oprávnění:** Pouze role `DungeonMaster` nebo `Admin`
- **Request Body (`CreateCampaignRequest`):**
```json
{
  "name": "Prokletí Strahda",
  "description": "Gotický horor v mlhách Barovie."
}
```
- **Response (201 Created — `CampaignDetailDto`)**

---

#### PUT `/api/v1/campaigns/{id}`
Upraví název, popis nebo stav aktivity kampaně.
- **Oprávnění:** Pouze Pán jeskyně dané kampaně nebo Admin
- **Request Body (`UpdateCampaignRequest`):**
```json
{
  "name": "Prokletí Strahda — Dokončeno",
  "description": "Hrdinové porazili Strahda a zachránili Barovii.",
  "isActive": false
}
```
- **Response (200 OK — `CampaignDetailDto`)**

---

#### DELETE `/api/v1/campaigns/{id}`
Smaže celou kampaň. Postavy hráčů zůstanou zachovány, pouze se jim vymaže vazba na kampaň.
- **Oprávnění:** Pouze Pán jeskyně dané kampaně nebo Admin
- **Response (204 No Content)**

---

#### POST `/api/v1/campaigns/join`
Připojí hráče ke kampani pomocí zvacího kódu. Volitelně lze rovnou přiřadit existující postavu.
- **Oprávnění:** `[Authorize]` (Rate limit: 5/min)
- **Request Body (`JoinCampaignRequest`):**
```json
{
  "inviteCode": "BAROVIA-2026",
  "characterId": "e4a2c74d-b6a3-4876-b633-1ecdb8088190"
}
```
- **Response (200 OK — `CampaignDetailDto`)**
*(Při neplatném kódu vrací 404 Not Found z důvodu ochrany proti brute-force vyhledávání kódů).*

---

#### DELETE `/api/v1/campaigns/{id}/membership`
Opuštění kampaně přihlášeným hráčem.
- **Oprávnění:** `[Authorize]`
- **Response (204 No Content)**

---

#### GET `/api/v1/campaigns/{id}/members`
Vrátí seznam všech členů dané kampaně.
- **Oprávnění:** `[Authorize]` (Člen kampaně)
- **Response (200 OK — `List<CampaignMemberDto>`):**
```json
[
  {
    "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "userName": "GeraltZRivie",
    "memberRole": "DungeonMaster",
    "characterId": null,
    "characterName": null,
    "allowsDmStatEdits": false,
    "joinedAt": "2026-11-01T12:00:00Z"
  },
  {
    "userId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "userName": "Strider",
    "memberRole": "Player",
    "characterId": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
    "characterName": "Thorin Pavéza",
    "allowsDmStatEdits": true,
    "joinedAt": "2026-11-02T18:00:00Z"
  }
]
```

---

#### GET `/api/v1/campaigns/{id}/player-characters`
Vrátí seznam všech postav hráčů v kampani pro přehledový dashboard Pána jeskyně.
- **Oprávnění:** Pouze role `DungeonMaster` nebo `Admin`
- **Response (200 OK — `List<CharacterListItemDto>`):**
```json
[
  {
    "id": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
    "name": "Thorin Pavéza",
    "className": "Fighter",
    "raceName": "Mountain Dwarf",
    "level": 3,
    "currentHitPoints": 28,
    "maxHitPoints": 28,
    "campaignId": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
    "isPublic": true,
    "updatedAt": "2026-11-10T12:00:00Z"
  }
]
```

---

### 5.5 DM Poznámky ke kampani (`/api/v1/campaigns/{campaignId}/dm-notes`)

Správa soukromých a sdílených poznámek Pána jeskyně v rámci dané kampaně.

#### Viditelnost poznámek (`visibility`):
- `"DmOnly"`: Vidí pouze Pán jeskyně (výchozí stav).
- `"SharedWithPlayer"`: Vidí Pán jeskyně a konkrétní hráč uvedený v `sharedWithUserId`.
- `"Public"`: Vidí všichni členové dané kampaně.

#### GET `/api/v1/campaigns/{campaignId}/dm-notes`
- **Oprávnění:** `[Authorize]` (DM vidí vše, hráči pouze poznámky se sdílenou viditelností)
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK — `PagedResult<DmNoteDto>`):**
```json
{
  "items": [
    {
      "id": "8a7b6c5d-4e3f-2a1b-0c9d-8e7f6a5b4c3d",
      "campaignId": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
      "title": "Tajemství starosty Vallaki",
      "content": "Starosta skrývá v podkroví kultisty a připravuje slavnost.",
      "visibility": "DmOnly",
      "sharedWithUserId": null,
      "createdAt": "2026-11-04T10:00:00Z",
      "updatedAt": "2026-11-04T10:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

---

#### POST `/api/v1/campaigns/{campaignId}/dm-notes`
- **Oprávnění:** Pouze Pán jeskyně kampaně nebo Admin
- **Request Body (`CreateDmNoteRequest`):**
```json
{
  "title": "Dopis pro Thorina",
  "content": "Tvůj klan byl zrazen někým z vnitřního kruhu.",
  "visibility": "SharedWithPlayer",
  "sharedWithUserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```
- **Response (201 Created — `DmNoteDto`):**
```json
{
  "id": "9b8a7c6d-5e4f-3a2b-1c0d-9e8f7a6b5c4d",
  "campaignId": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
  "title": "Dopis pro Thorina",
  "content": "Tvůj klan byl zrazen někým z vnitřního kruhu.",
  "visibility": "SharedWithPlayer",
  "sharedWithUserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "createdAt": "2026-11-10T16:00:00Z",
  "updatedAt": "2026-11-10T16:00:00Z"
}
```

---

#### PUT `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}`
- **Oprávnění:** Pouze Pán jeskyně kampaně nebo Admin
- **Request Body (`UpdateDmNoteRequest`):**
```json
{
  "title": "Dopis pro Thorina — Aktualizováno",
  "content": "Tvůj klan byl zrazen rádcem jménem Duergar.",
  "visibility": "SharedWithPlayer",
  "sharedWithUserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```
- **Response (200 OK — `DmNoteDto`)**

---

#### DELETE `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}`
- **Oprávnění:** Pouze Pán jeskyně kampaně nebo Admin
- **Response (204 No Content)**

---

### 5.6 Databáze pravidel / Kompendium (10 kolekcí)

Databáze pravidel je rozdělena do **10 dedikovaných, silně typovaných modulů** přesně podle kategorií 5e.tools.
Všechny čtecí endpointy kompendia jsou **veřejně přístupné (anonymní)** bez nutnosti předávat JWT token.

Společné query parametry pro všechny seznamové endpointy:
- `search` (string): Fulltextové vyhledávání v názvu a popisu.
- `source` (string): Kód knihy (např. `PHB`, `MM`, `DMG`, `XGE`, `TCE`).
- `page` (int, default `1`): Číslo stránky.
- `pageSize` (int, default `20`, max `100`): Počet položek na stránku.
- `sortBy` (string): Název sloupce pro řazení (např. `name`, `level`, `cr`).
- `sortDirection` (string): `"asc"` nebo `"desc"`.

---

#### 5.6.1 Kouzla (`/api/v1/spells`)
- **Specifické filtry:** `level` (0-9), `school` (Evocation, Abjuration...), `className` (Wizard, Cleric...), `isRitual` (bool), `requiresConcentration` (bool), `damageType` (fire, cold...).
- **Seznam:** `GET /api/v1/spells?school=Evocation&level=3&page=1&pageSize=20`
- **Detail:** `GET /api/v1/spells/{id}`
- **Response (200 OK — `SpellDto`):**
```json
{
  "id": "10000000-0000-0000-0000-000000000001",
  "name": "Fireball",
  "source": "PHB",
  "page": 241,
  "level": 3,
  "school": "Evocation",
  "castingTime": "1 action",
  "range": "150 feet",
  "components": "V, S, M (a tiny ball of bat guano and sulfur)",
  "duration": "Instantaneous",
  "classes": "Sorcerer, Wizard",
  "isRitual": false,
  "requiresConcentration": false,
  "damageTypes": "fire",
  "savingThrow": "dexterity",
  "dataJson": "{\"entries\":[\"A bright streak flashes from your pointing finger to a point you choose within range and then blossoms with a low roar into an explosion of flame.\",\"Each creature in a 20-foot-radius sphere centered on that point must make a Dexterity saving throw. A target takes {@damage 8d6} fire damage on a failed save, or half as much damage on a successful one.\"],\"entriesHigherLevel\":[{\"type\":\"entries\",\"name\":\"At Higher Levels\",\"entries\":[\"When you cast this spell using a spell slot of 4th level or higher, the damage increases by {@scaledamage 8d6|3-9|1d6} for each slot level above 3rd.\"]}]}"
}
```

---

#### 5.6.2 Bestiář nestvůr (`/api/v1/monsters`)
- **Specifické filtry:** `minCr`, `maxCr`, `type` (humanoid, dragon, undead...), `size` (Small, Medium, Large...), `alignment`.
- **Seznam:** `GET /api/v1/monsters?minCr=0.25&maxCr=1&type=humanoid`
- **Detail:** `GET /api/v1/monsters/{id}`
- **Response (200 OK — `MonsterDto`):**
```json
{
  "id": "20000000-0000-0000-0000-000000000001",
  "name": "Goblin",
  "source": "MM",
  "page": 166,
  "size": "Small",
  "type": "humanoid (goblinoid)",
  "alignment": "Neutral Evil",
  "armorClass": 15,
  "hitPoints": 7,
  "hitPointsFormula": "2d6",
  "speed": "30 ft.",
  "challengeRating": 0.25,
  "challengeRatingString": "1/4",
  "strength": 8,
  "dexterity": 14,
  "constitution": 10,
  "intelligence": 10,
  "wisdom": 8,
  "charisma": 8,
  "senses": "Darkvision 60 ft., passive Perception 9",
  "languages": "Common, Goblin",
  "dataJson": "{\"trait\":[{\"name\":\"Nimble Escape\",\"entries\":[\"The goblin can take the Disengage or Hide action as a bonus action on each of its turns.\"]}],\"action\":[{\"name\":\"Scimitar\",\"entries\":[\"Melee Weapon Attack: {@hit 4} to hit, reach 5 ft., one target. Hit: 5 ({@damage 1d6 + 2}) slashing damage.\"]},{\"name\":\"Shortbow\",\"entries\":[\"Ranged Weapon Attack: {@hit 4} to hit, range 80/320 ft., one target. Hit: 5 ({@damage 1d6 + 2}) piercing damage.\"]}]}"
}
```

---

#### 5.6.3 Předměty a vybavení (`/api/v1/items`)
- **Specifické filtry:** `type` (Weapon, Armor, Wondrous Item, Potion...), `rarity` (Common, Rare, Legendary...), `requiresAttunement` (bool).
- **Seznam:** `GET /api/v1/items?rarity=Rare&requiresAttunement=true`
- **Detail:** `GET /api/v1/items/{id}`
- **Response (200 OK — `ItemDto`):**
```json
{
  "id": "30000000-0000-0000-0000-000000000001",
  "name": "Flame Tongue",
  "source": "DMG",
  "page": 170,
  "itemType": "Weapon",
  "rarity": "Rare",
  "requiresAttunement": true,
  "attunementRequirement": "requires attunement",
  "weight": 3.0,
  "valueInCopper": 500000,
  "damage": "1d8",
  "properties": "V, M",
  "dataJson": "{\"entries\":[\"You can use a bonus action to speak this magic sword's command word, causing flames to erupt from the blade. These flames shed bright light in a 40-foot radius and dim light for an additional 40 feet. While the sword is ablaze, it deals an extra {@damage 2d6} fire damage to any target it hits.\"]}"
}
```

---

#### 5.6.4 Rasy (`/api/v1/races`)
- **Seznam:** `GET /api/v1/races`
- **Detail:** `GET /api/v1/races/{id}`
- **Response (200 OK — `RaceDto`):**
```json
{
  "id": "40000000-0000-0000-0000-000000000001",
  "name": "Dwarf",
  "source": "PHB",
  "page": 18,
  "size": "Medium",
  "speed": "25 ft.",
  "abilityBonuses": "Con +2",
  "dataJson": "{\"subraces\":[{\"name\":\"Hill Dwarf\",\"ability\":[{\"wis\":1}],\"entries\":[\"Dwarven Toughness: Your hit point maximum increases by 1, and it increases by 1 every time you gain a level.\"]},{\"name\":\"Mountain Dwarf\",\"ability\":[{\"str\":2}],\"entries\":[\"Dwarven Armor Training: You have proficiency with light and medium armor.\"]}],\"entries\":[\"Darkvision: Accustomed to life underground, you have superior vision in dark and dim conditions.\",\"Dwarven Resilience: You have advantage on saving throws against poison, and you have resistance against poison damage.\"]}"
}
```

---

#### 5.6.5 Povolání (`/api/v1/classes`)
- **Seznam:** `GET /api/v1/classes`
- **Detail:** `GET /api/v1/classes/{id}`
- **Response (200 OK — `ClassDto`):**
```json
{
  "id": "50000000-0000-0000-0000-000000000001",
  "name": "Wizard",
  "source": "PHB",
  "page": 112,
  "hitDie": "d6",
  "primaryAbility": "Intelligence",
  "savingThrows": "Intelligence, Wisdom",
  "spellcastingAbility": "Intelligence",
  "dataJson": "{\"subclasses\":[{\"name\":\"School of Evocation\",\"source\":\"PHB\"},{\"name\":\"School of Abjuration\",\"source\":\"PHB\"}],\"classFeatures\":[\"Spellcasting\",\"Arcane Recovery\",\"Arcane Tradition\"],\"classTableGroups\":[{\"title\":\"Spell Slots per Spell Level\",\"colLabels\":[\"Cantrips Known\",\"1st\",\"2nd\",\"3rd\",\"4th\",\"5th\",\"6th\",\"7th\",\"8th\",\"9th\"]}]}"
}
```

---

#### 5.6.6 Odbornosti (`/api/v1/feats`)
- **Seznam:** `GET /api/v1/feats`
- **Detail:** `GET /api/v1/feats/{id}`
- **Response (200 OK — `FeatDto`):**
```json
{
  "id": "60000000-0000-0000-0000-000000000001",
  "name": "War Caster",
  "source": "PHB",
  "page": 170,
  "category": "General",
  "prerequisite": "The ability to cast at least one spell",
  "dataJson": "{\"entries\":[\"You have advantage on Constitution saving throws that you make to maintain your concentration on a spell when you take damage.\",\"You can perform the somatic components of spells even when you have weapons or a shield in one or both hands.\",\"When a hostile creature's movement provokes an opportunity attack from you, you can use your reaction to cast a spell at the creature, rather than making an opportunity attack.\"]}"
}
```

---

#### 5.6.7 Zázemí (`/api/v1/backgrounds`)
- **Seznam:** `GET /api/v1/backgrounds`
- **Detail:** `GET /api/v1/backgrounds/{id}`
- **Response (200 OK — `BackgroundDto`):**
```json
{
  "id": "70000000-0000-0000-0000-000000000001",
  "name": "Acolyte",
  "source": "PHB",
  "page": 127,
  "skillProficiencies": "Insight, Religion",
  "toolProficiencies": null,
  "languages": "Two of your choice",
  "dataJson": "{\"feature\":{\"name\":\"Shelter of the Faithful\",\"entries\":[\"As an acolyte, you command the respect of those who share your faith, and you can perform the religious ceremonies of your deity.\"]},\"startingEquipment\":[\"A holy symbol\",\"A prayer book\",\"5 sticks of incense\",\"Vestments\",\"15 gp\"]}"
}
```

---

#### 5.6.8 Stavy (`/api/v1/conditions`)
- **Seznam:** `GET /api/v1/conditions`
- **Detail:** `GET /api/v1/conditions/{id}`
- **Response (200 OK — `ConditionDto`):**
```json
{
  "id": "80000000-0000-0000-0000-000000000001",
  "name": "Poisoned",
  "source": "PHB",
  "page": 292,
  "conditionType": "Condition",
  "dataJson": "{\"entries\":[\"A poisoned creature has disadvantage on attack rolls and ability checks.\"]}"
}
```

---

#### 5.6.9 Dobrodružství (`/api/v1/adventures`)
- **Seznam:** `GET /api/v1/adventures`
- **Detail:** `GET /api/v1/adventures/{id}`
- **Response (200 OK — `AdventureDto`):**
```json
{
  "id": "90000000-0000-0000-0000-000000000001",
  "name": "Lost Mine of Phandelver",
  "code": "LMoP",
  "source": "LMoP",
  "storyline": "Starter Set",
  "minLevel": 1,
  "maxLevel": 5,
  "published": "2014-07-15",
  "dataJson": "{\"contents\":[\"Part 1: Goblin Arrows\",\"Part 2: Phandalin\",\"Part 3: The Spider's Web\",\"Part 4: Wave Echo Cave\"]}"
}
```

---

#### 5.6.10 Knihy pravidel (`/api/v1/books`)
- **Seznam:** `GET /api/v1/books`
- **Detail:** `GET /api/v1/books/{id}`
- **Response (200 OK — `BookDto`):**
```json
{
  "id": "a0000000-0000-0000-0000-000000000001",
  "name": "Player's Handbook",
  "code": "PHB",
  "source": "PHB",
  "bookGroup": "core",
  "published": "2014-08-19",
  "author": "Wizards of the Coast",
  "dataJson": "{\"contents\":[{\"name\":\"Ch. 1: Step-by-Step Characters\"},{\"name\":\"Ch. 2: Races\"},{\"name\":\"Ch. 3: Classes\"},{\"name\":\"Ch. 11: Spells\"}]}"
}
```

---

### 5.7 Globální vyhledávač & Filtry (`/api/v1/search`)

#### GET `/api/v1/search?q={text}&limit=10`
Bleskový fulltext napříč všemi 9 000+ položkami pro klávesovou zkratku `Ctrl+K` / modální okno.
- **Oprávnění:** Veřejné (anonymní)
- **Query parametry:**
  - `q` (string, povinné): Hledaný řetězec (např. `"fire"`).
  - `limit` (int, default 10, max 50): Maximální počet výsledků.
- **Response (200 OK — `List<QuickSearchItemDto>`):**
```json
[
  {
    "id": "10000000-0000-0000-0000-000000000001",
    "name": "Fireball",
    "category": "Spell",
    "source": "PHB",
    "badge": "3. úroveň",
    "snippet": "A bright streak flashes from your pointing finger to a point you choose..."
  },
  {
    "id": "30000000-0000-0000-0000-000000000001",
    "name": "Flame Tongue",
    "category": "Item",
    "source": "DMG",
    "badge": "Vzácný",
    "snippet": "You can use a bonus action to speak this magic sword's command word..."
  },
  {
    "id": "20000000-0000-0000-0000-000000000002",
    "name": "Fire Elemental",
    "category": "Monster",
    "source": "MM",
    "badge": "CR 5",
    "snippet": "Large elemental, neutral"
  }
]
```

---

#### GET `/api/v1/search/filters`
Vrátí dynamická metadata pro frontendové filtry a dropdowny (výběr kategorií, škol magie, úrovní, typů potvor, vzácností).
- **Oprávnění:** Veřejné (anonymní)
- **Response (200 OK — `EncyclopediaFilterMetadataDto`):**
```json
{
  "categories": [
    "Kouzla",
    "Nestvůry",
    "Předměty",
    "Rasy",
    "Povolání",
    "Odbornosti",
    "Zázemí",
    "Stavy",
    "Dobrodružství",
    "Knihy"
  ],
  "sources": ["PHB", "MM", "DMG", "XGE", "TCE", "VGM"],
  "spellSchools": [
    "Abjuration",
    "Conjuration",
    "Divination",
    "Enchantment",
    "Evocation",
    "Illusion",
    "Necromancy",
    "Transmutation"
  ],
  "spellLevels": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9],
  "monsterTypes": [
    "aberration",
    "beast",
    "celestial",
    "construct",
    "dragon",
    "elemental",
    "fey",
    "fiend",
    "giant",
    "humanoid",
    "monstrosity",
    "ooze",
    "plant",
    "undead"
  ],
  "challengeRatings": ["0", "1/8", "1/4", "1/2", "1", "2", "3", "4", "5", "10", "15", "20", "30"],
  "itemRarities": ["Běžný", "Neobvyklý", "Vzácný", "Velmi vzácný", "Legendární", "Artefakt"]
}
```

---

### 5.8 Generátory obsahu (`/api/v1/generators`)

Simulace ekonomie a generátory obsahu pro Pána jeskyně.

#### GET `/api/v1/generators/loot`
Vygeneruje poklad (mince, drahokamy, umělecké předměty a magické předměty) dle CR z tabulek v Dungeon Master's Guide.
- **Oprávnění:** Veřejné / [Authorize]
- **Query parametry:**
  - `cr` (double, default `1`): Nebezpečnost nestvůry (např. `0.25`, `1`, `5`, `12`).
  - `isHoard` (bool, default `false`): `false` pro poklad jednotlivce (Individual Treasure), `true` pro poklad v doupěti (Treasure Hoard).
- **Response (200 OK — `LootResultDto`):**
```json
{
  "copper": 0,
  "silver": 45,
  "electrum": 0,
  "gold": 120,
  "platinum": 2,
  "totalGoldValue": 144.5,
  "gemsAndArtObjects": [
    "2x Karneol ( 50 gp)",
    "Stříbrný kalich s rytinou (25 gp)"
  ],
  "magicItems": [
    "Léčivý lektvar (Potion of Healing)",
    "Svitek kouzla: Magická střela (Magic Missile)"
  ]
}
```

---

#### GET `/api/v1/generators/shop`
Vygeneruje nabídku obchodu s položkami, počty a cenami dle typu osady a specializace obchodu.
- **Oprávnění:** Veřejné / [Authorize]
- **Query parametry:**
  - `settlement` (int nebo enum, default `1` / `Town`):
    - `0` / `Village` (Víska, max do 100 gp)
    - `1` / `Town` (Městečko, max do 500 gp)
    - `2` / `City` (Velké město, max do 5 000 gp)
    - `3` / `Metropolis` (Metropole, bez cenového omezení)
  - `shopType` (int nebo enum, default `0` / `GeneralStore`):
    - `0` / `GeneralStore` (Všeobecné zboží a vybavení)
    - `1` / `Blacksmith` (Kovář a zbrojíř)
    - `2` / `Alchemist` (Alchymista a lektvary)
    - `3` / `Temple` (Chrám a svěcená voda)
    - `4` / `MagicShop` (Magický krám)
- **Response (200 OK — `ShopInventoryDto`):**
```json
{
  "settlement": 1,
  "shopType": 1,
  "settlementName": "Městečko (Town)",
  "shopName": "Kovář a zbrojíř",
  "items": [
    {
      "itemId": "30000000-0000-0000-0000-000000000010",
      "name": "Dlouhý meč (Longsword)",
      "category": "Zbraně",
      "priceInGold": 15.0,
      "priceText": "15 gp",
      "rarity": "Běžný",
      "quantity": 3
    },
    {
      "itemId": "30000000-0000-0000-0000-000000000011",
      "name": "Kroužková zbroj (Chain Mail)",
      "category": "Zbroje",
      "priceInGold": 75.0,
      "priceText": "75 gp",
      "rarity": "Běžný",
      "quantity": 1
    },
    {
      "itemId": "30000000-0000-0000-0000-000000000012",
      "name": "Štít (Shield)",
      "category": "Zbroje",
      "priceInGold": 10.0,
      "priceText": "10 gp",
      "rarity": "Běžný",
      "quantity": 4
    }
  ]
}
```

---

### 5.9 Systémové informace & Stav (`/api/v1/changelog`, `/api/v1/status`)

#### GET `/api/v1/changelog`
Vrátí chronologický seznam publikovaných verzí a aktualizací aplikace formátovaných v Markdownu.
- **Oprávnění:** Veřejné (anonymní)
- **Response (200 OK — `List<ChangelogEntryDto>`):**
```json
[
  {
    "id": "f1e2d3c4-b5a6-9788-1234-56789abcdef0",
    "version": "v1.1.0",
    "title": "Nové kompendium a fond profilových obrázků",
    "releaseDate": "2026-11-10T12:00:00Z",
    "category": "Compendium",
    "contentMarkdown": "### Novinky ve verzi 1.1.0\n- Přidán výběr z **40 tematických avatarů**.\n- Doplněno 10 kolekcí kompendia 5e.tools.\n- Optimalizace rychlého vyhledávání pro `Ctrl+K`.",
    "isPublished": true,
    "createdAt": "2026-11-10T11:00:00Z",
    "updatedAt": null
  }
]
```

---

#### GET `/api/v1/status`
Strojově čitelná systémová telemetrie o stavu služeb, dostupnosti databáze a době běhu pro stavový dashboard (např. na `status.caroteka.cz`).
- **Oprávnění:** Veřejné (anonymní)
- **Response (200 OK):**
```json
{
  "status": "operational",
  "version": "1.0.0",
  "serverTime": "2026-11-10T17:30:00.1234567Z",
  "uptimeSeconds": 86400,
  "uptimeFormatted": "1d 0h 0m 0s",
  "services": [
    {
      "name": "Čarotéka Core API",
      "status": "operational",
      "description": ".NET 10 Web API"
    },
    {
      "name": "PostgreSQL Databáze",
      "status": "operational",
      "latencyMs": 2
    },
    {
      "name": "D&D 5e Kompendium",
      "status": "operational",
      "spells": 486,
      "monsters": 1120,
      "books": 24
    }
  ]
}
```

---

## 6. Kompletní TypeScript definice typů

Následující typy uložte do souboru `src/types/api.ts` ve vašem frontend projektu:

```typescript
// src/types/api.ts

// ==========================================
// 1. Společné typy a obálky
// ==========================================

export interface PagedResult<T> {
  items: T[];
  page: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
}

export interface ProblemDetails {
  type?: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  errors?: Record<string, string[]>;
}

export type AppRole = 'Admin' | 'DungeonMaster' | 'Player' | 'Homebrewer' | 'Moderator';
export type CampaignMemberRole = 'DungeonMaster' | 'Player' | 'Spectator';
export type NoteVisibility = 'DmOnly' | 'SharedWithPlayer' | 'Public';

// ==========================================
// 2. Autentizace & Uživatelé
// ==========================================

export interface RegisterRequest {
  email: string;
  userName: string;
  password: string;
  passwordConfirmation: string;
  avatarUrl?: string | null;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RefreshRequest {
  refreshToken?: string;
}

export interface LogoutRequest {
  refreshToken?: string;
}

export interface AuthResponse {
  accessToken: string;
  accessTokenExpiresAt: string;
  userId: string;
  userName: string;
  email: string;
  avatarUrl?: string | null;
  roles: AppRole[];
}

export interface UserProfileDto {
  id: string;
  userName: string;
  email: string;
  displayName: string;
  avatarUrl?: string | null;
  roles: AppRole[];
}

export interface ProfileAvatarDto {
  id: string;
  name: string;
  category: string;
  imageUrl: string;
  thumbnailUrl: string;
  tags: string[];
}

export interface UpdateAvatarRequest {
  avatarIdOrUrl: string;
}

export interface ChangePasswordRequest {
  currentPassword: string;
  newPassword: string;
}

export interface ChangeEmailRequest {
  newEmail: string;
  currentPassword: string;
}

// ==========================================
// 3. Postavy (Characters)
// ==========================================

export interface AbilityScoresDto {
  strength: number;
  dexterity: number;
  constitution: number;
  intelligence: number;
  wisdom: number;
  charisma: number;
}

export interface DeathSavesDto {
  successes: number;
  failures: number;
}

export interface SkillEntryDto {
  name: string;
  isProficient: boolean;
  hasExpertise: boolean;
  miscModifier: number;
}

export interface SavingThrowEntryDto {
  ability: string;
  isProficient: boolean;
}

export interface CharacterFeatureDto {
  name: string;
  description: string;
  source: string;
}

export interface ActiveConditionDto {
  name: string;
  note?: string | null;
}

export interface CharacterSheetDataDto {
  skills: SkillEntryDto[];
  savingThrows: SavingThrowEntryDto[];
  languages: string[];
  features: CharacterFeatureDto[];
  activeConditions: ActiveConditionDto[];
  resources: Record<string, number>;
  personalityTraits: string;
  ideals: string;
  bonds: string;
  flaws: string;
}

export interface InventoryItemDto {
  id?: string;
  name: string;
  itemName?: string;
  quantity: number;
  weightPerUnit: number;
  weight?: number;
  isEquipped: boolean;
  isAttuned?: boolean;
  description?: string | null;
  compendiumItemId?: string | null;
  notes?: string | null;
}

export interface InventoryDataDto {
  items: InventoryItemDto[];
  platinumPieces: number;
  goldPieces: number;
  electrumPieces: number;
  silverPieces: number;
  copperPieces: number;
}

export interface SpellEntryDto {
  name: string;
  level: number;
  isPrepared: boolean;
  school: string;
}

export interface SpellSlotEntryDto {
  level: number;
  total: number;
  used: number;
}

export interface SpellbookDataDto {
  spells: SpellEntryDto[];
  spellSlots: SpellSlotEntryDto[];
  spellcastingAbility: string;
  isConcentrating: boolean;
  concentratingOnSpell?: string | null;
}

export interface CharacterListItemDto {
  id: string;
  name: string;
  className: string;
  raceName: string;
  level: number;
  currentHitPoints: number;
  maxHitPoints: number;
  campaignId?: string | null;
  isPublic: boolean;
  updatedAt: string;
}

export interface CharacterDetailDto {
  id: string;
  ownerUserId: string;
  campaignId?: string | null;
  name: string;
  className: string;
  subclass?: string | null;
  subclassName?: string | null;
  raceName: string;
  background: string;
  alignment: string;
  level: number;
  experiencePoints: number;
  currentHitPoints: number;
  maxHitPoints: number;
  temporaryHitPoints: number;
  armorClass: number;
  initiative: number;
  speed: number;
  hitDice: string;
  proficiencyBonus?: number;
  abilityScores: AbilityScoresDto;
  stats?: AbilityScoresDto;
  deathSaves: DeathSavesDto;
  isPublic: boolean;
  sheetData: CharacterSheetDataDto;
  inventory: InventoryDataDto;
  spellbook: SpellbookDataDto;
  spells?: SpellEntryDto[];
  createdAt: string;
  updatedAt: string;
}

export interface CreateCharacterRequest {
  name: string;
  className: string;
  subclass?: string | null;
  raceName: string;
  background: string;
  alignment: string;
  abilityScores: AbilityScoresDto;
  maxHitPoints: number;
}

export interface UpdateCharacterProfileRequest {
  name: string;
  background: string;
  alignment: string;
  isPublic: boolean;
  sheetData: CharacterSheetDataDto;
  inventory: InventoryDataDto;
  spellbook: SpellbookDataDto;
}

export interface LevelUpCharacterRequest {
  additionalHitPoints: number;
}

export interface UpdateCharacterMechanicsRequest {
  experiencePoints: number;
  armorClass: number;
  initiative: number;
  speed: number;
  hitDice: string;
}

export interface UpdateHitPointsRequest {
  currentHitPoints: number;
  temporaryHitPoints: number;
  deathSaves?: DeathSavesDto | null;
}

// ==========================================
// 4. Kampaně & DM Poznámky
// ==========================================

export interface CampaignListItemDto {
  id: string;
  name: string;
  dungeonMasterUserId: string;
  memberCount: number;
  callerRole: string;
  isActive: boolean;
  updatedAt: string;
}

export interface CampaignMemberDto {
  userId: string;
  userName: string;
  memberRole: string;
  characterId?: string | null;
  characterName?: string | null;
  allowsDmStatEdits: boolean;
  joinedAt: string;
}

export interface QuestDto {
  id: string;
  title: string;
  description: string;
  status: string;
  isVisibleToPlayers: boolean;
  rewardDescription?: string | null;
  rewardGoldPieces?: number | null;
  createdAt: string;
  updatedAt: string;
}

export interface SessionLogDto {
  id: string;
  sequenceNumber: number;
  recap: string;
  sessionDate: string; // YYYY-MM-DD
  participantCharacterNames: string[];
  createdAt: string;
}

export interface CampaignDetailDto {
  id: string;
  name: string;
  description: string;
  setting?: string | null;
  dungeonMasterUserId: string;
  dungeonMasterName?: string | null;
  inviteCode?: string | null; // Přítomno POUZE pro DM!
  isActive: boolean;
  members: CampaignMemberDto[];
  quests: QuestDto[];
  sessionLogs: SessionLogDto[];
  createdAt: string;
  updatedAt: string;
}

export interface CreateCampaignRequest {
  name: string;
  description: string;
}

export interface UpdateCampaignRequest {
  name: string;
  description: string;
  isActive: boolean;
}

export interface JoinCampaignRequest {
  inviteCode: string;
  characterId?: string | null;
}

export interface DmNoteDto {
  id: string;
  campaignId: string;
  title: string;
  content: string;
  visibility: NoteVisibility;
  sharedWithUserId?: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface CreateDmNoteRequest {
  title: string;
  content: string;
  visibility: NoteVisibility;
  sharedWithUserId?: string | null;
}

export interface UpdateDmNoteRequest {
  title: string;
  content: string;
  visibility: NoteVisibility;
  sharedWithUserId?: string | null;
}

// ==========================================
// 5. Kompendium & Pravidla 5e.tools
// ==========================================

export interface SpellDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  level: number;
  school: string;
  castingTime: string;
  range: string;
  components: string;
  duration: string;
  classes: string;
  isRitual: boolean;
  requiresConcentration: boolean;
  damageTypes?: string | null;
  savingThrow?: string | null;
  dataJson: string; // Parsujte pomocí JSON.parse(spell.dataJson)
}

export interface MonsterDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  size: string;
  type: string;
  alignment: string;
  armorClass: number;
  hitPoints: number;
  hitPointsFormula: string;
  speed: string;
  challengeRating: number;
  challengeRatingString: string;
  strength: number;
  dexterity: number;
  constitution: number;
  intelligence: number;
  wisdom: number;
  charisma: number;
  senses?: string | null;
  languages?: string | null;
  dataJson: string;
}

export interface ItemDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  itemType: string;
  rarity: string;
  requiresAttunement: boolean;
  attunementRequirement?: string | null;
  weight?: number | null;
  valueInCopper?: number | null;
  damage?: string | null;
  properties?: string | null;
  dataJson: string;
}

export interface RaceDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  size: string;
  speed: string;
  abilityBonuses?: string | null;
  dataJson: string;
}

export interface ClassDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  hitDie: string;
  primaryAbility: string;
  savingThrows: string;
  spellcastingAbility?: string | null;
  dataJson: string;
}

export interface FeatDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  category?: string | null;
  prerequisite?: string | null;
  dataJson: string;
}

export interface BackgroundDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  skillProficiencies?: string | null;
  toolProficiencies?: string | null;
  languages?: string | null;
  dataJson: string;
}

export interface ConditionDto {
  id: string;
  name: string;
  source: string;
  page?: number | null;
  conditionType: string;
  dataJson: string;
}

export interface AdventureDto {
  id: string;
  name: string;
  code: string;
  source: string;
  storyline?: string | null;
  minLevel?: number | null;
  maxLevel?: number | null;
  published?: string | null;
  dataJson: string;
}

export interface BookDto {
  id: string;
  name: string;
  code: string;
  source: string;
  bookGroup?: string | null;
  published?: string | null;
  author?: string | null;
  dataJson: string;
}

export interface QuickSearchItemDto {
  id: string;
  name: string;
  category: 'Spell' | 'Monster' | 'Item' | 'Race' | 'Class' | 'Feat' | 'Background' | 'Condition' | 'Adventure' | 'Book';
  source: string;
  badge?: string | null;
  snippet?: string | null;
}

export interface EncyclopediaFilterMetadataDto {
  categories: string[];
  sources: string[];
  spellSchools: string[];
  spellLevels: number[];
  monsterTypes: string[];
  challengeRatings: string[];
  itemRarities: string[];
}

// ==========================================
// 6. Generátory & Systém
// ==========================================

export interface LootResultDto {
  copper: number;
  silver: number;
  electrum: number;
  gold: number;
  platinum: number;
  totalGoldValue: number;
  gemsAndArtObjects: string[];
  magicItems: string[];
}

export interface ShopInventoryItemDto {
  itemId?: string | null;
  name: string;
  category: string;
  priceInGold: number;
  priceText: string;
  rarity: string;
  quantity: number;
}

export interface ShopInventoryDto {
  settlement: number;
  shopType: number;
  settlementName: string;
  shopName: string;
  items: ShopInventoryItemDto[];
}

export interface ChangelogEntryDto {
  id: string;
  version: string;
  title: string;
  releaseDate: string;
  category: string;
  contentMarkdown: string;
  isPublished: boolean;
  createdAt: string;
  updatedAt?: string | null;
}
```
