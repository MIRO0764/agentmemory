# mem.md — pracovné poznámky k tomuto forku

Interná poznámka pre prácu na forku `MIRO0764/agentmemory` (vetva `main`).
Nie je súčasť projektovej dokumentácie pre upstream — len záznam rozhodnutí
a stavu pre ďalšiu prácu na tomto repozitári.

## Účel forku

Vlastná, bezpečnostne udržiavaná kópia `rohitg00/agentmemory` s cieľom
odstrániť/zdokumentovať zraniteľnosti a zastarané balíky bez závislosti na
tom, kedy (a či) ich rieši upstream.

## Git remotes

```
origin   -> https://github.com/MIRO0764/agentmemory.git   (fetch + push)
upstream -> https://github.com/rohitg00/agentmemory.git   (fetch only, push URL zámerne "DISABLED")
```

**Pravidlo: nikdy nepushovať do `upstream`.** Push URL je zámerne nastavená
na neplatnú hodnotu (`DISABLED`), aby náhodný `git push upstream ...` zlyhal
skôr, než by čokoľvek odišlo do `rohitg00/agentmemory`.

## Branch workflow

- `main` = stabilná, otestovaná vetva. Zodpovedá `origin/main`.
- Experimentálne/rizikové zmeny (dependency bumpy, refaktory) idú do
  vetiev `security/<meno>` alebo `chore/<meno>`, nikdy priamo do `main`.
- Pred mergom do `main`: `npm run build` + `npm test` a porovnanie výsledku
  s baseline na `main` (rovnaký počet/zoznam zlyhaní = žiadna regresia).
- Merge do `main` až po overení, potom `git push origin main` +
  `git push origin <branch>` (vetvu možno nechať na forku ako históriu).

## Stav security auditu (2026-09-12)

Pred zásahom: **14 zraniteľností** (8 moderate, 6 high) cez `npm audit`.
Po zásahu (aktuálny `main`): **10 zraniteľností** (8 moderate, 2 high —
oba z rovnakého reťazca, pozri nižšie).

### Opravené

| Balík | Zmena | Vetva |
|---|---|---|
| `adm-zip`, `sharp` | `overrides` na `^0.6.1` / `^0.35.4` (caret rozsah `@huggingface/transformers`/`onnxruntime-node` bránil npm-u vyriešiť patchnuté verzie automaticky) | `security/embedding-deps-review` |
| `@anthropic-ai/sdk` | `^0.100.1` → `^0.125.0`, žiadne breaking API zmeny | `security/anthropic-sdk-0.125` |
| `tsdown` | `^0.21.10` → `^0.23.0` (predpoklad pre budúci TS7 pokus) | `security/tsdown-0.23` |
| `vitest` | `^4.1.6` → `^5.0.0`, + `clearMocks/mockReset/restoreMocks: false` v `vitest.config.ts` (vitest 5 zmenil defaulty, rozbilo to `test/mcp-standalone.test.ts`) | `security/vitest-5` |

### Vedome odložené (zdokumentované v `SECURITY.md`, vetva `security/iii-sdk-0.23`, zatiaľ nezlúčená do main)

- **`iii-sdk@0.11.2` → 8 moderate + 1 z 2 high CVE cez OpenTelemetry 1.x reťazec.**
  Fix existuje len v `iii-sdk@0.23.0`, ale `iii-sdk` je verzovaná v lockstepe
  s pinnutým `iii-engine` binary (`IIPINNED_VERSION = "0.11.2"` v `src/cli.ts`).
  Engine `0.11.6+` má iný worker model, na ktorý agentmemory ešte nie je
  zrefaktorovaná (spôsobuje EPIPE loops, prázdne search výsledky). Neupgradovať
  `iii-sdk` samostatne bez koordinovaného refaktoru + bumpu enginu.
  - Reálna dosiahnuteľnosť: `W3CBaggagePropagator` sa skutočne používa v
    `iii-sdk` pre `onInvokeFunction` RPC medzi engine a workerom, ale ide po
    internom WebSockete (port `49134`), ktorý defaultne bindí na `localhost`.
    Pre bežné lokálne spustenie nízke riziko.
  - Druhý high nález (`@opentelemetry/propagator-jaeger`) je **mŕtvy kód** —
    `JaegerPropagator` sa v `iii-sdk` nikde neinštancuje, len leží v
    `node_modules` ako tranzitívna závislosť.
- **TypeScript 7.0.2** — oficiálne experimentálne, `tsgo` (natívny kompilátor)
  zlyháva pri generovaní `.d.ts` pre `src/hooks/task-completed.ts` aj s
  najnovším `tsdown`/`rolldown-plugin-dts`. Nie je to security issue, len
  nezrelý tooling — ostáva na `typescript@^6.0.3`, kým TS7 API nestabilizuje.

## Ako spustiť lokálne (namiesto `npx @agentmemory/agentmemory`)

`npx` sťahuje neopravenú verziu z verejného npm registra. Aby si bežal na
opravenom kóde z tohto forku:

```bash
npm run build
node dist/cli.mjs          # == npm start; ekvivalent npx @agentmemory/agentmemory
node dist/cli.mjs demo     # demo dáta + recall test
```

Pre globálny príkaz `agentmemory` ukazujúci na tento fork: `npm link`
(vytvorí globálny symlink; treba `npm unlink -g @agentmemory/agentmemory`
pre návrat k npm verzii).

## Poznámky k prostrediu

- Lockfile (`package-lock.json`) sa zámerne necommituje (viď `SECURITY.md`
  v origináli) — generuje sa lokálne pri `npm install`, je v `.gitignore`.
- Na Windows `npm run build` hlási na konci neškodné
  `The system cannot find the path specified.` — build skript používa
  POSIX `cp`/`mkdir -p`, ktoré `cmd.exe` (cez ktorý npm scripty spúšťa)
  nepozná. TS/JS build aj tak prebehne správne, chýba len kopírovanie
  statických súborov (`iii-config*.yaml`, viewer assets) do `dist/`.
- Testy majú ~29 predexistujúcich zlyhaní aj na čistom upstream `main` —
  sú to Windows-špecifické problémy (symlinky, drive-letter cesty, Docker),
  CI beží len na ubuntu-latest + macos-latest. Použi tento počet ako
  baseline pri overovaní, že nová zmena nepridala regresiu.
