# Simplified Mode - Dokumentacja Implementacji

## Opis funkcjonalności

Feature flag `simplifiedMode` pozwala ukryć wybrane elementy UI Control Dashboard:
- Ogranicza widoczne zakładki do: **Chat, Overview, Channels, Instances, Sessions, Cron Jobs**
- Ukrywa zakładki: Skills, Nodes, Config, Debug, Logs
- Ukrywa sekcję "Resources > Docs"
- Ukrywa logo
- Zmienia tytuł z "OPENCLAW" na "DASHBOARD"
- Ukrywa podtytuł "Gateway Dashboard"

## Konfiguracja

Plik: `~/.openclaw/openclaw.json`

```json
{
  "gateway": {
    "controlUi": {
      "tabs": {
        "simplifiedMode": true
      }
    }
  }
}
```

---

## Zmiany w plikach (szczegóły implementacji)

### 1. Backend: Typy konfiguracji

**Plik:** `src/config/types.gateway.ts`

Dodać typ `GatewayControlUiTabsConfig` i rozszerzyć `GatewayControlUiConfig`:

```typescript
export type GatewayControlUiTabsConfig = {
  /** Enable simplified mode showing only Chat and Cron tabs. */
  simplifiedMode?: boolean;
};

export type GatewayControlUiConfig = {
  enabled?: boolean;
  basePath?: string;
  allowInsecureAuth?: boolean;
  dangerouslyDisableDeviceAuth?: boolean;
  /** Tab visibility configuration for the Control UI dashboard. */
  tabs?: GatewayControlUiTabsConfig;  // <-- DODAĆ
};
```

---

### 2. Backend: Walidacja schematu

**Plik:** `src/config/zod-schema.ts`

W sekcji `gateway.controlUi` dodać pole `tabs`:

```typescript
controlUi: z
  .object({
    enabled: z.boolean().optional(),
    basePath: z.string().optional(),
    allowInsecureAuth: z.boolean().optional(),
    dangerouslyDisableDeviceAuth: z.boolean().optional(),
    tabs: z                                    // <-- DODAĆ
      .object({
        simplifiedMode: z.boolean().optional(),
      })
      .strict()
      .optional(),
  })
  .strict()
  .optional(),
```

---

### 3. Backend: Schemat protokołu snapshot

**Plik:** `src/gateway/protocol/schema/snapshot.ts`

Dodać schematy dla ControlUi i rozszerzyć SnapshotSchema:

```typescript
// DODAĆ po SessionDefaultsSchema:
export const ControlUiTabsSchema = Type.Object(
  {
    simplifiedMode: Type.Optional(Type.Boolean()),
  },
  { additionalProperties: false },
);

export const ControlUiSchema = Type.Object(
  {
    tabs: Type.Optional(ControlUiTabsSchema),
  },
  { additionalProperties: false },
);

// W SnapshotSchema DODAĆ pole:
export const SnapshotSchema = Type.Object(
  {
    presence: Type.Array(PresenceEntrySchema),
    health: HealthSnapshotSchema,
    stateVersion: StateVersionSchema,
    uptimeMs: Type.Integer({ minimum: 0 }),
    configPath: Type.Optional(NonEmptyString),
    stateDir: Type.Optional(NonEmptyString),
    sessionDefaults: Type.Optional(SessionDefaultsSchema),
    controlUi: Type.Optional(ControlUiSchema),  // <-- DODAĆ
  },
  { additionalProperties: false },
);
```

---

### 4. Backend: Budowanie snapshot

**Plik:** `src/gateway/server/health-state.ts`

W funkcji `buildGatewaySnapshot()` dodać `controlUi` do zwracanego obiektu:

```typescript
export function buildGatewaySnapshot(): Snapshot {
  const cfg = loadConfig();
  // ... existing code ...
  return {
    presence,
    health: emptyHealth,
    stateVersion: { presence: presenceVersion, health: healthVersion },
    uptimeMs,
    configPath: CONFIG_PATH,
    stateDir: STATE_DIR,
    sessionDefaults: {
      defaultAgentId,
      mainKey,
      mainSessionKey,
      scope,
    },
    controlUi: {                              // <-- DODAĆ
      tabs: cfg.gateway?.controlUi?.tabs,
    },
  };
}
```

---

### 5. UI: Helpers nawigacji

**Plik:** `ui/src/ui/navigation.ts`

Dodać na końcu pliku:

```typescript
/** Tabs visible in simplified mode */
export const SIMPLIFIED_MODE_TABS: readonly Tab[] = [
  "chat",
  "overview",
  "channels",
  "instances",
  "sessions",
  "cron",
] as const;

/** Filter tab groups based on simplified mode */
export function filterTabGroups(
  groups: typeof TAB_GROUPS,
  simplifiedMode: boolean
): Array<{ label: string; tabs: Tab[] }> {
  if (!simplifiedMode) {
    return groups.map((g) => ({ label: g.label, tabs: [...g.tabs] }));
  }

  return groups
    .map((group) => ({
      label: group.label,
      tabs: group.tabs.filter((tab) =>
        (SIMPLIFIED_MODE_TABS as readonly string[]).includes(tab)
      ) as Tab[],
    }))
    .filter((group) => group.tabs.length > 0);
}
```

---

### 6. UI: Typ stanu

**Plik:** `ui/src/ui/app-view-state.ts`

Dodać typ `TabsConfig` i pole w `AppViewState`:

```typescript
// Na początku pliku DODAĆ:
export type TabsConfig = {
  simplifiedMode: boolean;
};

// W typie AppViewState DODAĆ pole (po "tab: Tab;"):
export type AppViewState = {
  settings: UiSettings;
  password: string;
  tab: Tab;
  tabsConfig: TabsConfig;  // <-- DODAĆ
  // ... rest of fields
};
```

---

### 7. UI: Stan komponentu

**Plik:** `ui/src/ui/app.ts`

Dodać stan `tabsConfig`:

```typescript
@customElement("openclaw-app")
export class OpenClawApp extends LitElement {
  @state() settings: UiSettings = loadSettings();
  @state() password = "";
  @state() tab: Tab = "chat";
  @state() tabsConfig = { simplifiedMode: false };  // <-- DODAĆ
  // ... rest of state
}
```

---

### 8. UI: Obsługa snapshot z gateway

**Plik:** `ui/src/ui/app-gateway.ts`

**A) Dodać importy:**

```typescript
import { SIMPLIFIED_MODE_TABS, type Tab } from "./navigation";
import { setTab } from "./app-settings";
```

**B) Dodać typ TabsConfig do GatewayHost:**

```typescript
type TabsConfig = {
  simplifiedMode: boolean;
};

type GatewayHost = {
  // ... existing fields ...
  tabsConfig: TabsConfig;  // <-- DODAĆ
};
```

**C) Dodać typ ControlUiSnapshot:**

```typescript
type ControlUiSnapshot = {
  tabs?: {
    simplifiedMode?: boolean;
  };
};
```

**D) W funkcji `applySnapshot()` dodać obsługę controlUi:**

```typescript
export function applySnapshot(host: GatewayHost, hello: GatewayHelloOk) {
  const snapshot = hello.snapshot as
    | {
        presence?: PresenceEntry[];
        health?: HealthSnapshot;
        sessionDefaults?: SessionDefaultsSnapshot;
        controlUi?: ControlUiSnapshot;  // <-- DODAĆ
      }
    | undefined;

  // ... existing code for presence, health, sessionDefaults ...

  // DODAĆ na końcu funkcji:
  // Always set tabsConfig from snapshot (default to simplifiedMode: false if not present)
  const simplifiedMode = snapshot?.controlUi?.tabs?.simplifiedMode ?? false;
  host.tabsConfig = { simplifiedMode };

  // Redirect to chat if current tab is hidden in simplified mode
  if (simplifiedMode && !(SIMPLIFIED_MODE_TABS as readonly string[]).includes(host.tab)) {
    setTab(host as unknown as Parameters<typeof setTab>[0], "chat");
  }
}
```

---

### 9. UI: Renderowanie

**Plik:** `ui/src/ui/app-render.ts`

**A) Dodać import:**

```typescript
import {
  TAB_GROUPS,
  filterTabGroups,  // <-- DODAĆ
  iconForTab,
  pathForTab,
  subtitleForTab,
  titleForTab,
  type Tab,
} from "./navigation";
```

**B) Na początku funkcji `renderApp()` dodać filtrowanie:**

```typescript
export function renderApp(state: AppViewState) {
  // Filter tabs based on simplified mode
  const filteredGroups = filterTabGroups(TAB_GROUPS, state.tabsConfig.simplifiedMode);

  // ... rest of function
}
```

**C) W sekcji nawigacji zamienić `TAB_GROUPS.map` na `filteredGroups.map`:**

```typescript
<aside class="nav ${state.settings.navCollapsed ? "nav--collapsed" : ""}">
  ${filteredGroups.map((group) => {  // <-- ZMIENIĆ z TAB_GROUPS
    // ... existing group rendering code
  })}
```

**D) Warunkowe renderowanie sekcji Resources:**

```typescript
${state.tabsConfig.simplifiedMode ? nothing : html`
<div class="nav-group nav-group--links">
  <div class="nav-label nav-label--static">
    <span class="nav-label__text">Resources</span>
  </div>
  <div class="nav-group__items">
    <a class="nav-item nav-item--external" href="https://docs.openclaw.ai" ...>
      ...Docs...
    </a>
  </div>
</div>
`}
```

**E) Warunkowe renderowanie branding:**

```typescript
<div class="brand">
  ${state.tabsConfig.simplifiedMode ? nothing : html`
  <div class="brand-logo">
    <img src="..." alt="OpenClaw" />
  </div>
  `}
  <div class="brand-text">
    <div class="brand-title">${state.tabsConfig.simplifiedMode ? "DASHBOARD" : "OPENCLAW"}</div>
    ${state.tabsConfig.simplifiedMode ? nothing : html`<div class="brand-sub">Gateway Dashboard</div>`}
  </div>
</div>
```

---

## Deployment na VPS

Po każdej aktualizacji kodu:

```bash
# 1. Ściągnij kod
git pull

# 2. Przebuduj UI
cd ui
npm install
npm run build
cd ..

# 3. Zrestartuj gateway
```

**Ważne:** Gateway serwuje statyczne pliki z `dist/control-ui/`. Bez przebudowania UI zmiany nie będą widoczne.

---

## Testowanie lokalne

```bash
# 1. Ustaw config
cat > ~/.openclaw/openclaw.json << 'EOF'
{
  "gateway": {
    "mode": "local",
    "auth": {
      "token": "dev-token-123"
    },
    "controlUi": {
      "tabs": {
        "simplifiedMode": true
      }
    }
  }
}
EOF

# 2. Uruchom gateway
node openclaw.mjs gateway run &

# 3. Uruchom UI dev server
cd ui && npm run dev &

# 4. Otwórz http://localhost:5173/
```

---

## Commity (historia zmian)

1. `feat(ui): add simplified mode feature flag for Control UI tabs`
2. `feat(ui): expand simplified mode tabs and hide Resources section`
3. `feat(ui): customize branding in simplified mode`
4. `fix(ui): always set tabsConfig from snapshot`

---

## Pliki do modyfikacji (podsumowanie)

| Plik | Opis zmiany |
|------|-------------|
| `src/config/types.gateway.ts` | Typ `GatewayControlUiTabsConfig` |
| `src/config/zod-schema.ts` | Walidacja `tabs.simplifiedMode` |
| `src/gateway/protocol/schema/snapshot.ts` | Schemat `ControlUiSchema` |
| `src/gateway/server/health-state.ts` | `controlUi` w snapshot |
| `ui/src/ui/navigation.ts` | `SIMPLIFIED_MODE_TABS`, `filterTabGroups()` |
| `ui/src/ui/app-view-state.ts` | Typ `TabsConfig` |
| `ui/src/ui/app.ts` | Stan `tabsConfig` |
| `ui/src/ui/app-gateway.ts` | Obsługa snapshot + redirect |
| `ui/src/ui/app-render.ts` | Filtrowanie nawigacji, branding |
