## 1. Globální chování API, chyby a limity

### 1.1 Formát chyb
Veškeré chybové odpovědi API mají Content-Type `application/problem+json` a jednotnou strukturu:
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Porušení doménových pravidel",
  "status": 400,
  "detail": "Postavu nelze přiřadit ke kampani, jejímž nejste aktivním členem.",
  "instance": "/api/v1/campaigns/join"
}
```

Při validačních chybách (např. chybějící povinné pole nebo heslo kratší než 8 znaků) vrací status `400 Bad Request` s detailem polí:
```json
{
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Email": ["Pole Email musí obsahovat platnou e-mailovou adresu."],
    "Password": ["The field Password must be a string or array type with a minimum length of '8'."]
  }
}
```

### 1.2 Rate Limiting
Pro ochranu proti brute-force útokům jsou zavedeny limity (při překročení vrací HTTP `429 Too Many Requests`):
- **Autentizace (`AuthPolicy`):** `POST /api/v1/auth/login`, `/register`, `/refresh` – **max 10 požadavků za minutu** na IP adresu.
- **Připojení ke kampani (`JoinCampaignPolicy`):** `POST /api/v1/campaigns/join` – **max 5 požadavků za minutu** na IP adresu (ochrana zvacích kódů).

### 1.3 PagedResult
Seznamové endpointy používají query parametry:
- `page`: číslo stránky (číslováno od 1, výchozí hodnota `1`)
- `pageSize`: počet prvků na stránku (výchozí hodnota `20`, maximální hodnota `50`)

Struktura odpovědi `PagedResult<T>`:
```json
{
  "items": [ ... ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 42,
  "totalPages": 3
}
```

### 1.4 Velikostní limity a integrita dat
- **Limit velikosti requestu:** `POST /api/v1/characters` a `PUT /api/v1/characters/{id}` mají nastavený striktní limit **512 KB** (`RequestSizeLimit`).
- **Limity polí v listu postavy:**
  - Dovednosti (`skills`): max 50 položek
  - Záchranné hody (`savingThrows`): max 10 položek
  - Jazyky (`languages`): max 50 položek
  - Rysy a vlastnosti (`features`): max 200 položek
  - Aktivní stavy (`activeConditions`): max 50 položek
  - Inventář (`items`): max 200 položek
  - Kniha kouzel (`spells`): max 200 položek
  - Kouzelnické sloty (`spellSlots`): max 20 úrovní

---

## 2. API Reference

### 2.1 Autentizace (`/api/v1/auth`)

| Metoda | Endpoint | Oprávnění | Popis |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Anonymní (Rate limit: 10/min) | Registrace nového uživatele |
| `POST` | `/api/v1/auth/login` | Anonymní (Rate limit: 10/min) | Přihlášení existujícího uživatele |
| `POST` | `/api/v1/auth/refresh` | Anonymní (Rate limit: 10/min) | Obnovení access tokenu pomocí cookie |
| `POST` | `/api/v1/auth/logout` | `[Authorize]` | Odhlášení a zneplatnění refresh tokenu |

#### POST `/api/v1/auth/register`
**Request Body (`RegisterRequest`):**
```json
{
  "email": "hrac@caroteka.cz",
  "userName": "GeraltZRivie",
  "password": "MojeSuperHeslo123!",
  "passwordConfirmation": "MojeSuperHeslo123!"
}
```
**Response (201 Created):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "accessTokenExpiresAt": "2026-10-25T13:51:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "roles": ["Player"]
}
```
*(Současně nastaví HttpOnly cookie `refreshToken`).*

#### POST `/api/v1/auth/login`
**Request Body (`LoginRequest`):**
```json
{
  "email": "hrac@caroteka.cz",
  "password": "MojeSuperHeslo123!"
}
```
**Response (200 OK):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "accessTokenExpiresAt": "2026-10-25T13:51:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "roles": ["Player", "DungeonMaster"]
}
```
*(Současně nastaví HttpOnly cookie `refreshToken`).*

#### POST `/api/v1/auth/refresh`
**Request Body:** Volitelné prázdné tělo `{}` (cookie se odešle automaticky).
```json
{}
```
**Response (200 OK):**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "accessTokenExpiresAt": "2026-10-25T14:51:00Z",
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "userName": "GeraltZRivie",
  "roles": ["Player"]
}
```

#### POST `/api/v1/auth/logout`
**Hlavička:** `Authorization: Bearer <accessToken>`  
**Request Body:** Volitelné prázdné tělo `{}`  
**Response (204 No Content):** Cookie `refreshToken` je smazána.

---

### 2.2 Postavy (`/api/v1/characters`)

> [!IMPORTANT]
> **Ochrana proti Mass Assignment (SEC-04):**  
> Vytváření a úprava postavy jsou striktně rozděleny. Herní statistiky (Level, XP, Max HP, AC, Atributy) **nelze** libovolně přepsat v rámci editace profilu `PUT /{id}`!  
> Zvýšení úrovně se provádí výhradně přes `/level-up`, úprava HP přes `/hit-points` a herní mechaniky přes `/mechanics`.

| Metoda | Endpoint | Oprávnění | Popis |
|---|---|---|---|
| `GET` | `/api/v1/characters` | `[Authorize]` | Seznam postav přihlášeného uživatele |
| `GET` | `/api/v1/characters/{id}` | `[Authorize]` | Detail konkrétní postavy |
| `POST` | `/api/v1/characters` | `[Authorize]` | Vytvoření nové postavy (1. úroveň) |
| `PUT` | `/api/v1/characters/{id}` | `[Authorize]` (Vlastník) | Úprava profilu, příběhu, deníku a JSONB dat |
| `POST` | `/api/v1/characters/{id}/level-up` | `[Authorize]` (Vlastník) | Zvýšení úrovně postavy o +1 a navýšení HP |
| `PUT` | `/api/v1/characters/{id}/mechanics` | `[Authorize]` (Vlastník) | Úprava mechanik (XP, AC, iniciativa, rychlost) |
| `PATCH` | `/api/v1/characters/{id}/hit-points` | `[Authorize]` (Vlastník nebo DM) | Rychlá změna HP a death saves během sezení |
| `DELETE` | `/api/v1/characters/{id}` | `[Authorize]` (Vlastník) | Smazání postavy |

#### GET `/api/v1/characters`
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK):** `PagedResult<CharacterListItemDto>`
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
      "updatedAt": "2026-10-25T11:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

#### GET `/api/v1/characters/{id}`
- **Response (200 OK):** `CharacterDetailDto` obsahující kompletní data včetně `abilityScores`, `sheetData`, `inventory` a `spellbook`. Pokud postava neexistuje nebo k ní volající nemá práva, vrací **404 Not Found**.

#### POST `/api/v1/characters`
Vytvoření postavy na 1. úrovni.  
> **Pozor (SEC-01):** Postavu **nelze** při vytvoření rovnou přiřadit do kampaně. Do kampaně se postava přiřazuje až přes zvací kód v `POST /api/v1/campaigns/join`.

**Request Body (`CreateCharacterRequest`):**
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
**Response (201 Created):** `CharacterDetailDto`

#### PUT `/api/v1/characters/{id}`
Úprava profilu, popisu, deníku a JSONB struktur.  
**Request Body (`UpdateCharacterProfileRequest`):**
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
    "resources": { "FavoredEnemyCharges": 3 },
    "personalityTraits": "Klidný a bystrý pozorovatel.",
    "ideals": "Příroda a rovnováha.",
    "bonds": "Můj rodný hvozd.",
    "flaws": "Těžko odpouštím křivdy."
  },
  "inventory": {
    "items": [
      { "name": "Dlouhý luk", "quantity": 1, "weightPerUnit": 2.0, "isEquipped": true, "description": "1d8 piercing" },
      { "name": "Šípy", "quantity": 20, "weightPerUnit": 0.05, "isEquipped": false, "description": null }
    ],
    "platinumPieces": 0,
    "goldPieces": 45,
    "electrumPieces": 0,
    "silverPieces": 12,
    "copperPieces": 30
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

#### POST `/api/v1/characters/{id}/level-up`
Zvýší úroveň postavy o 1 a přičte zvolené HP.  
**Request Body (`LevelUpCharacterRequest`):**
```json
{
  "additionalHitPoints": 7
}
```
**Response (200 OK):** `CharacterDetailDto` s navýšeným levelem a novým `maxHitPoints`.

#### PUT `/api/v1/characters/{id}/mechanics`
Ruční aktualizace mechanických údajů (např. po získání magického předmětu nebo rozdělení XP).  
**Request Body (`UpdateCharacterMechanicsRequest`):**
```json
{
  "experiencePoints": 1500,
  "armorClass": 16,
  "initiative": 3,
  "speed": 30,
  "hitDice": "3d10"
}
```

#### PATCH `/api/v1/characters/{id}/hit-points`
Rychlá úprava životů během souboje.  
> **Oprávnění:** Může volat vlastník postavy NEBO Pán jeskyně kampaně (pokud hráč v kampani povolil `AllowsDmStatEdits`).

**Request Body (`UpdateHitPointsRequest`):**
```json
{
  "currentHitPoints": 8,
  "temporaryHitPoints": 5,
  "deathSaves": {
    "successes": 0,
    "failures": 0
  }
}
```

#### DELETE `/api/v1/characters/{id}`
Smaže postavu. Vrací **204 No Content**.

---

### 2.3 Kampaně (`/api/v1/campaigns`)

| Metoda | Endpoint | Oprávnění | Popis |
|---|---|---|---|
| `GET` | `/api/v1/campaigns` | `[Authorize]` | Seznam kampaní volajícího (kde je DM nebo hráč) |
| `GET` | `/api/v1/campaigns/{id}` | `[Authorize]` (Člen) | Detail kampaně (včetně úkolů a zápisů) |
| `GET` | `/api/v1/campaigns/{id}/player-characters` | `DungeonMaster`, `Admin` | Seznam postav hráčů v kampani |
| `POST` | `/api/v1/campaigns` | `DungeonMaster`, `Admin` | Vytvoření nové kampaně |
| `PUT` | `/api/v1/campaigns/{id}` | `DungeonMaster`, `Admin` (Vlastník) | Úprava názvu, popisu a aktivity |
| `DELETE` | `/api/v1/campaigns/{id}` | `DungeonMaster`, `Admin` (Vlastník) | Smazání kampaně |
| `POST` | `/api/v1/campaigns/join` | `[Authorize]` (Rate limit: 5/min) | Připojení hráče ke kampani pomocí kódu |
| `DELETE` | `/api/v1/campaigns/{id}/membership` | `[Authorize]` | Opuštění kampaně hráčem |
| `GET` | `/api/v1/campaigns/{id}/members` | `[Authorize]` (Člen) | Seznam členů kampaně |

#### GET `/api/v1/campaigns`
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK):** `PagedResult<CampaignListItemDto>`
```json
{
  "items": [
    {
      "id": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
      "name": "Ztracený důl Fandelveru",
      "dungeonMasterUserId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "memberCount": 5,
      "callerRole": "DungeonMaster",
      "isActive": true,
      "updatedAt": "2026-10-25T10:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

#### GET `/api/v1/campaigns/{id}`
- **Response (200 OK):** `CampaignDetailDto`
> **Důležité pravidlo:** Pole `inviteCode` je v odpovědi vyplněno **pouze pro Pána jeskyně**. Hráčům se vrací `null`. Úkoly v poli `quests` jsou pro běžné hráče automaticky filtrovány na ty, které mají `isVisibleToPlayers: true`.

```json
{
  "id": "9b12a813-f93d-4ec5-a0cf-82d8c3686d0b",
  "name": "Ztracený důl Fandelveru",
  "description": "Klasické dobrodružství na Mečovém pobřeží.",
  "dungeonMasterUserId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "inviteCode": "PHANDALIN-2026",
  "isActive": true,
  "members": [
    {
      "userId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "userName": "Strider",
      "memberRole": "Player",
      "characterId": "e4a2c74d-b6a3-4876-b633-1ecdb8088190",
      "characterName": "Aragorn",
      "allowsDmStatEdits": true,
      "joinedAt": "2026-10-10T18:00:00Z"
    }
  ],
  "quests": [
    {
      "id": "c1d2e3f4-a5b6-7890-1234-56789abcdef0",
      "title": "Najít Gundrena Rockseekera",
      "description": "Gundren byl unesen gobliny z kmene Cragmaw.",
      "status": "Active",
      "isVisibleToPlayers": true,
      "rewardDescription": "50 gp a podíl v dole",
      "rewardGoldPieces": 50,
      "createdAt": "2026-10-10T19:00:00Z",
      "updatedAt": "2026-10-10T19:00:00Z"
    }
  ],
  "sessionLogs": [
    {
      "id": "d1e2f3a4-b5c6-7890-1234-56789abcdef1",
      "sequenceNumber": 1,
      "recap": "Družina byla přepadena gobliny na stezce k Phandalinu.",
      "sessionDate": "2026-10-12",
      "participantCharacterNames": ["Aragorn", "Legolas"],
      "createdAt": "2026-10-12T22:30:00Z"
    }
  ],
  "createdAt": "2026-10-06T12:00:00Z",
  "updatedAt": "2026-10-25T10:00:00Z"
}
```

#### GET `/api/v1/campaigns/{id}/player-characters`
- **Oprávnění:** Pouze `DungeonMaster` a `Admin`
- **Response (200 OK):** `List<CharacterListItemDto>` (přehled všech postav hráčů v kampani pro DM dashboard).

#### POST `/api/v1/campaigns`
- **Oprávnění:** Pouze uživatel s rolí `DungeonMaster` nebo `Admin`.
**Request Body (`CreateCampaignRequest`):**
```json
{
  "name": "Prokletí Strahda",
  "description": "Gotický horor v mlhách Barovie."
}
```
**Response (201 Created):** `CampaignDetailDto` (s automaticky vygenerovaným `inviteCode`).

#### PUT `/api/v1/campaigns/{id}`
- **Oprávnění:** Pouze Pán jeskyně, který kampaň vlastní.
**Request Body (`UpdateCampaignRequest`):**
```json
{
  "name": "Prokletí Strahda — Dokončeno",
  "description": "Hrdinové porazili upířího lorda.",
  "isActive": false
}
```

#### POST `/api/v1/campaigns/join`
Připojí přihlášeného hráče do kampaně pomocí zvacího kódu. Volitelně lze rovnou přiřadit existující postavu hráče.  
> **Rate Limiting:** Max 5 volání za minutu. Při nesprávném kódu vrací **404 Not Found** (záměrně nevrací 400 kvůli ochraně proti enumeraci kódů).

**Request Body (`JoinCampaignRequest`):**
```json
{
  "inviteCode": "BAROVIA-2026",
  "characterId": "e4a2c74d-b6a3-4876-b633-1ecdb8088190"
}
```
**Response (200 OK):** `CampaignDetailDto`

#### DELETE `/api/v1/campaigns/{id}/membership`
Opuštění kampaně přihlášeným hráčem. Postava hráče v databázi zůstává, pouze se jí vymaže vazba `campaignId = null`. Vrací **204 No Content**.

#### DELETE `/api/v1/campaigns/{id}`
Smazání celé kampaně Pánem jeskyně. Postavy hráčů se nesmažou, dojde pouze k jejich odpojení. Vrací **204 No Content**.

---

### 2.4 DM Poznámky ke kampani (`/api/v1/campaigns/{campaignId}/dm-notes`)

Správa soukromých a sdílených poznámek Pána jeskyně v rámci dané kampaně.

| Metoda | Endpoint | Oprávnění | Popis |
|---|---|---|---|
| `GET` | `/api/v1/campaigns/{campaignId}/dm-notes` | `[Authorize]` | Seznam poznámek (DM vidí vše, hráči jen sdílené/veřejné) |
| `GET` | `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}` | `[Authorize]` | Detail konkrétní poznámky (dle viditelnosti) |
| `POST` | `/api/v1/campaigns/{campaignId}/dm-notes` | `DungeonMaster`, `Admin` | Vytvoření nové poznámky |
| `PUT` | `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}` | `DungeonMaster`, `Admin` | Úprava poznámky |
| `DELETE` | `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}` | `DungeonMaster`, `Admin` | Smazání poznámky |

#### Viditelnost poznámek (`Visibility`):
- `"DmOnly"`: Vidí pouze Pán jeskyně (výchozí).
- `"SharedWithPlayer"`: Vidí DM a konkrétní hráč zadaný v `sharedWithUserId`.
- `"Public"`: Vidí všichni členové dané kampaně.

#### GET `/api/v1/campaigns/{campaignId}/dm-notes`
- **Query parametry:** `page` (default 1), `pageSize` (default 20, max 50)
- **Response (200 OK):** `PagedResult<DmNoteDto>`

#### POST `/api/v1/campaigns/{campaignId}/dm-notes`
- **Oprávnění:** Pouze Pán jeskyně dané kampaně.
**Request Body (`CreateDmNoteRequest`):**
```json
{
  "title": "Tajemství starosty Vallaki",
  "content": "Starosta skrývá v podkroví kultisty a připravuje slavnost.",
  "visibility": "DmOnly",
  "sharedWithUserId": null
}
```
**Response (201 Created):** `DmNoteDto`

#### PUT `/api/v1/campaigns/{campaignId}/dm-notes/{noteId}`
- **Oprávnění:** Pouze Pán jeskyně dané kampaně.
**Request Body (`UpdateDmNoteRequest`):**
```json
{
  "title": "Dopis pro Aragorna",
  "content": "Tvůj rodokmen sahá až k dávným králům Arnoru.",
  "visibility": "SharedWithPlayer",
  "sharedWithUserId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```
**Response (200 OK):** `DmNoteDto`

---

## 3. Kompletní TypeScript Modely

```typescript
// ==========================================
// Společné typy a obálky
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
// Autentizace (Auth)
// ==========================================

export interface RegisterRequest {
  email: string;
  userName: string;
  password: string;
  passwordConfirmation: string;
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
  roles: AppRole[];
}

// ==========================================
// Postavy (Characters)
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
  name: string;
  quantity: number;
  weightPerUnit: number;
  isEquipped: boolean;
  description?: string | null;
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
  abilityScores: AbilityScoresDto;
  deathSaves: DeathSavesDto;
  isPublic: boolean;
  sheetData: CharacterSheetDataDto;
  inventory: InventoryDataDto;
  spellbook: SpellbookDataDto;
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
// Kampaně (Campaigns)
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
  dungeonMasterUserId: string;
  inviteCode?: string | null; // Vyplněno POUZE pro DM!
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

// ==========================================
// DM Poznámky (DmNotes)
// ==========================================

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
```