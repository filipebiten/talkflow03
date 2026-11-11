# Gincana "A Revanche" Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a live, autonomous, team-phone-driven gincana ("A Revanche") to `talkflow03`, chained directly after the existing Premiação section, with betting, 4 content blocks + a final round, and a telão that runs itself except for an operator PAUSE/override panel.

**Architecture:** Single-file Preact+htm app (unchanged convention — no build, no new files besides this doc pair). One shared Firebase RTDB document at `talkflow/talkflow03` (unchanged path) gains a new `revanche` shape that replaces the old placeholder "provas" system entirely. Three roles write disjoint slices of that document so there is never a two-writer race on the same field: **telão** owns phase transitions, grading and the autonomous clock (it is the one tab guaranteed open all night); **painel** owns manual overrides, point adjustments and the pause flag; **cada celular de equipe** owns only its own `apostas[idx]` / `respostas[idx]` entries.

**Tech Stack:** Preact 10.19.6 + htm 3.1.1 via esm.sh (unchanged), Firebase 10.13.2 Realtime Database (unchanged project `talkflow-dba50`), Web Audio API for sound (reuse existing `beep()`), a single extra CDN script for QR rendering: `https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.js` (UMD global `window.qrcode`, no bundler needed, consistent with the project's CDN-only policy). No test framework exists anywhere in the TalkFlow family (`talkflow`, `velhosfins`, `overflow-awards`, `talkflow03` all ship with zero automated tests) — this plan follows that established convention. Every task's verification step is a concrete manual scenario run with the `claude-in-chrome` browser tools (as already used to verify `overflow-awards`), not a unit test file.

**Spec:** `docs/superpowers/specs/2026-11-10-gincana-revanche-design.md`

## Global Constraints

- No build step, no npm, no bundler — everything lives in `talkflow03/index.html`, exactly like every sibling app.
- Firebase path stays `talkflow/talkflow03` (unchanged); the simulation mode (Task 13) is the only thing that gets its own path, `talkflow/talkflow03-sim`, mirroring `overflow-awards`' `?teste=1` pattern.
- Design tokens are the ones already defined in `index.html`'s `:root` — do not introduce new colors beyond `COR_HEX` (already matches the spec's 10 team colors exactly) and the existing teal/off-white/preto system.
- `EQUIPES_INICIAIS` and `COR_HEX` already match the spec's team list and starting points exactly (lines 203–219 of the current `index.html`) — reuse as-is, do not re-type them.
- Bet cap: 60 per regular block, 120 on the final. Bet window: 45s. Round timers: 45s (Escolha), 90s (Cronos), 120s (Quatro por Quatro), final 180s. Speed bonus: flat +5, awarded once per round to the fastest correct answer. Regular-block scoring: ≥2 of 3 correct → bet doubles; 0 or 1 correct → lose half the bet (floor). Final scoring: all correct → bet doubles; any wrong → lose half the bet (floor).
- Content (questions, orderings, word groups, clues, the 12 Ezequiel phrases) is copied verbatim from the spec's "O CONTEÚDO — as 13 rodadas" section into the `BLOCOS_GINCANA` / `FINAL_GINCANA` constants — it is exact data the spec fixes, not something an implementer invents.
- "Telão só lê" from the existing README is **no longer true for the Revanche portion**: the telão client now also writes phase transitions, grading results and the autonomous clock. Update the README to say so (Task 14).

## Review Focus

- **Team phone loses connection mid-round and reconnects**: it must resume showing the current round/phase exactly where the sync left off, never re-show the bet screen or a stale round, and its locally-buffered answer (if any) must still get written once connectivity returns.
- **Timeout fires at the same instant the last team submits**: resolution must be idempotent (a `resolvidoEm`/already-resolved guard), so the round or bet window is never graded twice with conflicting results.
- **A team never bets before the window closes**: it must default to a bet of 0 at close time, not `undefined` — anything that later divides or doubles an undefined bet breaks the scoreboard.
- **Painel corrects a response after the block was already scored**: the correction must undo the old point delta and re-apply the new one, not just flip a `correta` flag while leaving stale points on the board.
- **PAUSAR fires in the same tick an autonomous transition was about to fire**: pause must win — no phase change, sound, or point write may land after `paused` becomes true until RESUME is pressed.

---

## File Structure

Everything below modifies `talkflow03/index.html` in place (971 lines today). No new runtime files. Two docs already created: this plan and its spec.

- `index.html` `<style>` block: a handful of new rules for the team-phone view, the telão's 10-block status grid, the bet slider, and the drag-reorder lists (Cronos/Final) and the Quatro-por-Quatro grid. Follow the existing naming style (`.time-*`, `.bloco-grid`, `.aposta-*`, `.cronos-*`, `.quatro-*`, `.pistas-*`).
- `index.html` `<script>` module: new constants, new `revanche` state shape, new pure grading functions, new components for `StageView` (telão), `ControlView` (painel) and a brand-new `TimeView` (celular) and `SimulacaoView`, wired into the existing `App()` router by `view` query param (`time`, `simulacao` join the existing `telao`/`painel`).
- `README.md`: new section documenting the three roles, how teams join, how to run Simulação, and that the telão now writes state for the Revanche portion.

---

### Task 1: Premiação lê o Overflow Awards ao vivo

**Files:**
- Modify: `index.html:162-268` (sync helpers), `index.html:628-764` (`PremiacaoControl`)

**Interfaces:**
- Consumes: `overflow-awards`' Firebase shape at `talkflow/overflowawards/real` — `nomes/{catId}/{key}`, `votos/{catId}/{key}/{voterId}`, `respostas/{voterId}/{catId}` (see `overflow-awards/index.html:174-178` for the exact paths and `rankingFromNomesVotos(nomes, votos)` for the ranking shape it already computes client-side).
- Produces: `premiacao.categorias[i].vencedor/votos/ranking` populated live instead of from an uploaded JSON; `premiacao.totalVotantes` live.

- [ ] **Step 1: Add a live subscription to the Overflow Awards data**

In `index.html`, add `const OVERFLOW_BASE = 'talkflow/overflowawards/real';` near `stateRef`, and a `subscribeOverflow(handler)` helper mirroring the existing `subscribe(path, handler)` pattern in `overflow-awards/index.html:180-186` (copy `rankingFromNomesVotos` and `normalizeName`-free ranking logic verbatim — it only needs `nomes`/`votos`, no normalization at this end since that already happened at vote time).

- [ ] **Step 2: Compute winners in `PremiacaoControl` instead of reading uploaded data**

Replace the `carregarJson` flow: `PremiacaoControl` subscribes to Overflow's `nomes`/`votos`/`respostas` on mount, derives `categorias[i].vencedor/votos/ranking` via `rankingFromNomesVotos` for each of the 15 `CATEGORIAS_PREMIACAO` ids, and derives `totalVotantes` from `Object.keys(respostas).length`. Keep the manual `vencedor`/`votos` text/number inputs working as an override the operator can still type into — when the operator edits them, stop recomputing that one category from live data until "recalcular ao vivo" is pressed again (a small per-category `travado` boolean is enough).
- Remove the "Carregar JSON da votação" file button and `carregarJson` function; keep the rest of `PremiacaoControl` (fase machine, revelar/próxima/pular/voltar, ranking toggle) unchanged.

- [ ] **Step 3: Manual verification**

Open `?view=painel` in one tab against the real Overflow Awards Firebase data (or seed a couple of test votes in `overflow-awards`'s `?teste=1`, pointed at the same project, if you don't want to touch real votes). Confirm: category winners and vote counts in the Premiação panel match what `overflow-awards`'s own `?view=resultado` shows, and update live when a new vote is cast in another tab.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: premiação lê resultado da votação ao vivo do Firebase"
```

---

### Task 2: Modelo de dados da Revanche

**Files:**
- Modify: `index.html:139-339` (constants + `initialState`/`sanitizeState`)

**Interfaces:**
- Produces:
  - `BLOCOS_GINCANA: Array<{ tipo: 'escolha'|'cronos'|'quatro'|'pistas', nome: string, tetoAposta: number, tempoApostaSeg: number, tempoRodadaSeg: number, rodadas: Array<RoundSpec> }>` (4 entries)
  - `FINAL_GINCANA: { tipo: 'cronos', nome: 'EZEQUIEL EMBARALHADO', tetoAposta: 120, tempoRodadaSeg: 180, itens: string[12], referencia: string }`
  - `RoundSpec` per tipo: `escolha` → `{ pergunta, opcoes: string[4], certaIdx }`; `cronos` → `{ titulo, itens: string[5] }` (items already in correct order — shuffle for display happens client-side in Task 7); `quatro` → `{ grupos: Array<{ tema, palavras: string[4] }> }` (4 entries); `pistas` → `{ resposta, pistas: string[5] }`.
  - `avaliarEscolha(opcaoIdx, round) -> boolean`, `avaliarCronos(ordemEnviada: number[], round) -> boolean` (compares to `[0,1,2,3,4]` since the spec order *is* the correct order and submissions are indices into the shuffled display array — see Task 7 for how the shuffle+index mapping works), `avaliarQuatro(gruposEnviados: number[][], round) -> boolean` (true only if every submitted group of 4 word-indices maps to the same `tema`), `avaliarPistas(texto: string, resposta: string) -> boolean` (case/accent-insensitive compare — reuse `normalizeName`-style normalization, inline a small `normalizeTexto` if `overflow-awards`' version isn't already present in this file).
  - `resolverAposta(aposta: number, acertos: number, totalRodadas: number) -> number` (delta to add to pontos: `acertos*2 >= totalRodadas ? aposta : -Math.floor(aposta/2)` — this one formula covers both "2 of 3" and the final's "all of 1", since `totalRodadas=1` makes `acertos*2>=1` mean `acertos===1`).

- [ ] **Step 1: Write `BLOCOS_GINCANA` and `FINAL_GINCANA`**

Copy every question, ordering, word group and clue set verbatim from `docs/superpowers/specs/2026-11-10-gincana-revanche-design.md`'s "O CONTEÚDO — as 13 rodadas" section, in the exact order given (Bloco 1 → 4, then a separate `FINAL_GINCANA`). Keep them at the top of the `<script>`, next to `CATEGORIAS_PREMIACAO`.

- [ ] **Step 2: Replace the `revanche` shape in `initialState()`**

Remove `provas`/`provaIndex`/`subfase`/`colocacao` entirely (the old placeholder system). New shape:

```js
revanche: {
  equipes: EQUIPES_INICIAIS.map((e) => ({ ...e, pontosIniciais: e.pontos })), // unchanged
  timeConectado: Array(10).fill(null), // [idx] = phone's localStorage id, or null
  fase: 'entrada', // entrada | aposta | revelacaoAposta | rodada | resultadoRodada | resultadoBloco | final | podio
  blocoIndex: 0, // 0-3 = BLOCOS_GINCANA, 4 = FINAL_GINCANA
  rodadaIndex: 0, // 0-2 within a block; always 0 on the final
  apostas: {}, // { [equipeIdx]: number }
  respostas: {}, // { [equipeIdx]: { valor, tempoMs, correta } }
  acertosBloco: {}, // { [equipeIdx]: number }, reset every block
  primeiroAcerto: null, // equipeIdx or null, per round
  prevRanking: null, // reuse existing arrow-animation field
  deadlineAt: null, // epoch ms, or null when nothing is counting down
  paused: false,
  pausedRemainingMs: null,
  resolvido: null, // guard key (e.g. 'aposta-0' or 'rodada-0-1') already graded, to make resolution idempotent
}
```

- [ ] **Step 3: Update `sanitizeState`**

Extend the `revanche` merge to default every new field (`timeConectado`, `apostas`, `respostas`, `acertosBloco`, etc.) the same way `colocacao` was defaulted before, so Firebase's array/empty-object quirks (noted in the existing comment at `index.html:245`) don't resurrect `undefined`.

- [ ] **Step 4: Manual verification**

In the browser console on `?view=painel` (or any view), `import`-free check isn't possible since it's a module — instead temporarily log `JSON.stringify(initialState().revanche)` via a throwaway `console.log` and confirm the shape matches the above, then remove the log. Call `avaliarEscolha`, `avaliarCronos`, `avaliarQuatro`, `avaliarPistas`, `resolverAposta` directly with a few hand-picked inputs from the spec's own rounds (e.g. `avaliarCronos([0,1,2,3,4], BLOCOS_GINCANA[1].rodadas[0])` → `true`; `avaliarQuatro(wrongGrouping, ...)` → `false`; `resolverAposta(60, 2, 3)` → `60`; `resolverAposta(60, 1, 3)` → `-30`) and confirm each by eye before moving on — this is the closest thing to a unit test this codebase has, so do it carefully.

- [ ] **Step 5: Commit**

```bash
git add index.html docs/superpowers/specs docs/superpowers/plans
git commit -m "feat: data model for gincana blocks, rounds, and grading"
```

---

### Task 3: Relógio autônomo + abertura (QR e entrada dos times)

**Files:**
- Modify: `index.html` (new helpers near the sound helpers; new telão component; new painel control)

**Interfaces:**
- Consumes: `revanche.deadlineAt`, `revanche.paused`, `revanche.pausedRemainingMs` (Task 2).
- Produces: `useContagem(deadlineAt, paused, pausedRemainingMs) -> msRestantes:number` (Preact hook, ticks every 250ms via `setInterval`+`useState`, returns 0 when `deadlineAt` is null or past); `abrirContagem(segundos) -> { deadlineAt }` helper the telão calls to start a window; a `<ContagemBadge ms=${msRestantes} />` component rendering "próximo em Ns…" discreetly, reused by every later autonomous screen.
- Produces: `StageEntrada` (telão) renders the QR (via `window.qrcode`, pointed at `location.origin + location.pathname + '?view=time'`) plus "X de 10 times" and auto-flips `fase` to `'aposta'` with `blocoIndex:0` 5s after the 10th `timeConectado` slot fills — **this write happens in the telão component's effect**, establishing the "telão owns transitions" pattern every later task follows.
- Produces: painel gets a "Liberar início" button (writes the same `fase:'aposta'` transition manually, for when a team is missing) and the single **PAUSAR/RETOMAR** button (writes `paused`/`pausedRemainingMs`), both living in a new always-visible `RevancheControl` header area (replacing the old `overflowPlacar`/`resultadoFinal` buttons tasks 11-12 will finish replacing).

- [ ] **Step 1: Add `qrcode-generator` script tag with Subresource Integrity**

In `index.html`'s `<head>`, add a classic `<script src="https://cdn.jsdelivr.net/npm/qrcode-generator@1.4.4/qrcode.js" integrity="sha384-..." crossorigin="anonymous"></script>` before the module `<script>` (defines `window.qrcode`, no import needed). Compute the real `integrity` hash at implementation time — fetch the exact pinned file and hash it (`curl -s <url> | openssl dgst -sha384 -binary | openssl base64 -A`) rather than guessing one; jsdelivr also publishes the hash directly if you append `?sha384` conventions aren't reliable, so compute it yourself. This is the one new external script this plan adds — the project's existing CDN scripts (esm.sh, Google Fonts) predate this convention and are out of scope here.

- [ ] **Step 2: Implement `useContagem` and `abrirContagem`**

`useContagem` is a plain hook using `useState`+`useEffect`+`setInterval`; when `paused` is true, return `pausedRemainingMs` directly without ticking. `abrirContagem(segundos)` just returns `{ deadlineAt: Date.now() + segundos * 1000 }` for the caller to merge into an `updateState`/telão-side `update()` call.

- [ ] **Step 3: Implement `StageEntrada` and wire telão's autonomous-advance effect**

Telão's `StageRevanche` (or a new root for the gincana, replacing the old `StageRevanchePlacarOverflow`) renders `StageEntrada` when `fase === 'entrada'`. Its effect watches `timeConectado.filter(Boolean).length`; on reaching 10, if not already counting down, writes `deadlineAt: Date.now()+5000`; a second effect (or the same one) watches `useContagem`'s returned ms and, on hitting 0 while `fase === 'entrada'`, writes `fase:'aposta', blocoIndex:0, deadlineAt: Date.now()+BLOCOS_GINCANA[0].tempoApostaSeg*1000`.

- [ ] **Step 4: Manual verification**

Open `?view=telao` — confirm a scannable QR renders and "0 de 10 times" shows. From a second tab's console (standing in for Task 4's not-yet-built phone UI), manually `update(dbRef(db, stateRef_path + '/revanche/timeConectado/0'), 'fake-id')` ten times (indices 0-9) and confirm: the counter climbs, and 5s after the 10th write the telão flips to `fase:'aposta'` on its own. Then test PAUSAR mid-countdown from the painel (once Step 3's control exists) and confirm the countdown visibly freezes and RETOMAR resumes it from where it froze, not from the full 5s.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: autonomous countdown clock, QR entry, and pause control"
```

---

### Task 4: Celular — entrada e escolha de cor

**Files:**
- Modify: `index.html` (new `TimeView` root component)

**Interfaces:**
- Consumes: `revanche.timeConectado` (Task 3), `revanche.equipes` (existing), `revanche.fase` (Task 2/3).
- Produces: `getTimeId()` (localStorage-backed id, same pattern as `overflow-awards`'s `getVoterId()`); `TimeView` renders a color-picker grid of the `equipes` whose `timeConectado[idx]` is still `null`, writes `timeConectado[idx] = myId` on tap, and from then on always renders whatever `fase` the shared state says (starting with a "aguardando o início" screen while `fase === 'entrada'`).

- [ ] **Step 1: Implement `getTimeId` and the color-claim screen**

Mirror `overflow-awards/index.html:210-218`'s `getVoterId` exactly, with its own localStorage key (`revancheTimeId`). On mount, `TimeView` subscribes to the whole `revanche` slice; if `timeConectado.includes(myId)`, it already knows its `equipeIdx` (the matching index) and skips the picker forever (including after a reload — this satisfies "Se o celular recarregar, volta no mesmo estado"). Otherwise render the grid of unclaimed equipes (name + `COR_HEX[cor]` swatch); tapping one calls `update(dbRef(db, .../timeConectado/${idx}), myId)`.

- [ ] **Step 2: Add the "aguardando" screen**

Once an `equipeIdx` is known and `fase === 'entrada'`, show a screen with the team's name/color and "aguardando o início da Revanche…" — this is also the fallback screen for any `fase` value this task doesn't yet handle (Tasks 5-11 fill those in one at a time), so `TimeView`'s phase switch should have a safe default branch from the start.

- [ ] **Step 3: Manual verification**

With Task 3's telão open, open `?view=time` in two more tabs (or two phones). Confirm each shows the unclaimed-colors grid, tapping a color removes it from the grid in the *other* tab live, and reloading a tab that already claimed a color skips straight to "aguardando" with the same color. Claim all 10 (can reuse the console trick from Task 3 for the remaining ones) and confirm the telão auto-advances per Task 3's Step 4.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: team phone entry and color claim flow"
```

---

### Task 5: Aposta

**Files:**
- Modify: `index.html` (`TimeView`'s `'aposta'` branch, telão's `'aposta'`/`'revelacaoAposta'` branches, telão's autonomous-advance effect)

**Interfaces:**
- Consumes: `BLOCOS_GINCANA[blocoIndex].tetoAposta/tempoApostaSeg` (or `FINAL_GINCANA` when `blocoIndex===4`), `revanche.apostas`.
- Produces: phone writes `apostas[idx]` (0..teto, via a stepper/slider, default 0, a "Confirmar aposta" button that's really just "lock in the current value" — the spec allows betting 0). Telão renders the 10-team grid showing which teams have bet (amount hidden or shown? spec says "Grid de 10 blocos mostra a aposta de cada equipe chegando" — show the number once submitted, dash before). Telão's effect: when `Object.keys(apostas).length === 10` OR the countdown hits 0 (guarded by `resolvido !== 'aposta-${blocoIndex}'`), fill any missing team with `apostas[idx]=0`, set `resolvido:'aposta-${blocoIndex}'`, move to `fase:'revelacaoAposta'` with `deadlineAt: Date.now()+4000`; a second transition (same pattern as Task 3) flips `fase:'rodada', rodadaIndex:0, respostas:{}, deadlineAt: now+tempoRodadaSeg*1000` after those 4s.

- [ ] **Step 1: Phone bet UI**

A numeric stepper (±5 or a drag slider, team preference — pick whichever is simplest to make reliably touch-friendly) clamped to `[0, teto]`, a big "Apostar" button, disabled once `apostas[idx]` already has a value for this `blocoIndex` (prevents double-submit; re-enable only if the painel explicitly reopens the bet per Task 12).

- [ ] **Step 2: Telão bet-arrival grid and reveal screen**

10-cell grid, each cell = team name + color + (its bet once present, else "…"). `revelacaoAposta` screen just lists all 10 teams' bets side by side for the 4s window.

- [ ] **Step 3: Telão autonomous resolution effect**

As described in Interfaces above — reuse the `resolvido` guard so the all-submitted path and the timeout path can't both fire.

- [ ] **Step 4: Manual verification**

With all 10 phones (or console-simulated writes) on the bet screen: bet with 9 of them and let the 45s timer expire — confirm the 10th gets `0`, the grid shows all 10, and it auto-advances through the 4s reveal into the first round. Repeat having all 10 bet before the timer — confirm it advances immediately rather than waiting out the clock.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: per-block betting with autonomous close"
```

---

### Task 6: Bloco Escolha (múltipla escolha)

**Files:** Modify: `index.html` (`TimeView`'s `'rodada'` branch for `tipo==='escolha'`, telão's round-status grid and result reveal, grading wire-up)

**Interfaces:**
- Consumes: `avaliarEscolha` (Task 2), the round-timer/`resolvido` pattern (Task 5).
- Produces: phone renders the 4 options as big tap targets, writes `respostas[idx] = { valor: opcaoIdx, tempoMs: Date.now()-rodadaIniciadaEm, correta: null }` once (correctness filled in by telão at resolution, per the plan's "telão grades" decision — the phone does **not** compute `correta` itself, so the answer key never needs to reach a shape the phone could display accidentally). Telão's status grid shows aguardando/jogando/entregue per team (derived from presence in `respostas`, no extra field needed) and, during `'resultadoRodada'`, reveals correct/incorrect per team, highlights `primeiroAcerto` (+5), and bumps `acertosBloco[idx]` and `equipes[idx].pontos` (+5 only) — the block-level bet payout happens in Task 9, not here.

- [ ] **Step 1: Phone UI for `tipo==='escolha'`**

- [ ] **Step 2: Telão status grid for `'rodada'` and reveal for `'resultadoRodada'`**

- [ ] **Step 3: Telão resolution effect** (grading + speed bonus + `acertosBloco` + the 8s auto-advance to the next round or to `'resultadoBloco'` after round 3 — this effect is the template Tasks 7-9 copy for their own `tipo`)

- [ ] **Step 4: Manual verification**

Run round 1 of Bloco 1 end to end with 10 simulated answers (mix of correct/incorrect, staggered timing); confirm: the grid never shows an option/answer before reveal, the fastest correct team gets highlighted +5, a team that never answers is graded wrong at timeout, and it auto-advances to round 2 after the 8s result screen.

- [ ] **Step 5: Commit** — `git commit -m "feat: multiple-choice round (Bloco 1)"`

---

### Task 7: Bloco Cronos (ordenar)

**Files:** Modify: `index.html` (phone drag-reorder list, telão grid/reveal reuse from Task 6, `avaliarCronos` wiring)

**Interfaces:**
- Consumes: `avaliarCronos` (Task 2), the shuffle-for-display concern noted in Task 2 — shuffle `round.itens` client-side on mount with a per-round-stable seed (e.g. `rodadaIndex+blocoIndex` through a tiny deterministic shuffle, or just `Math.random()` once in `useState(() => shuffle(round.itens))`, since each phone only needs *a* shuffled order, not the same one as other phones), submit the array of **original indices** in the order the team arranged them, so `avaliarCronos` can compare directly to `[0,1,2,3,4]`.
- Produces: drag-and-drop reorderable list (native HTML5 drag events or pointer-based reorder — pick whichever survives mobile touch reliably; HTML5 `dragstart`/`dragover`/`drop` is unreliable on touch, so use pointer events: `pointerdown` on a handle, `pointermove` to reorder by swapping with the item under the pointer, `pointerup` to settle), a "Confirmar ordem" button, and a 3-tentativa counter (spec: "3 tentativas" for Cronos) — a wrong confirm shows which positions are right/wrong *to that team only* (not revealing the answer to others) and lets them retry until tentativas run out or they get it right, whichever first; only the *final* submitted order (last attempt or the correct one) is what `respostas[idx].valor` holds for grading.

- [ ] **Step 1: Implement the pointer-based reorder list component**, reusable by Task 11 (12 items instead of 5).

- [ ] **Step 2: Wire the 3-attempt local retry loop** (local-only state on the phone until final submit or attempts exhausted).

- [ ] **Step 3: Reuse Task 6's telão grid/reveal and resolution effect**, swapping in `avaliarCronos`.

- [ ] **Step 4: Manual verification** — drag-reorder on an actual phone (or Chrome device emulation with touch) feels right (no jank, handles sloppy drags), 3-attempt limit enforced, grading matches the spec's culto/fim-de-semana/setlist orderings.

- [ ] **Step 5: Commit** — `git commit -m "feat: drag-to-order round (Bloco 2 Cronos)"`

---

### Task 8: Bloco Quatro por Quatro

**Files:** Modify: `index.html` (phone 16-word tap-to-group grid, telão reuse, `avaliarQuatro` wiring)

**Interfaces:**
- Consumes: `avaliarQuatro`, the "7 chances" cap (spec) — each wrong group-of-4 guess consumes one chance; phone shows chances remaining; once 4 correct groups are found or chances hit 0, submit whatever was found as `respostas[idx].valor` (array of the groups actually solved, graded by "found all 4 groups correctly" for the block's correct/incorrect purposes — partial solves still count as incorrect for the 2-of-3 bet math, matching NYT Connections-style all-or-nothing per round).
- Produces: 16-word grid (flatten+shuffle the 4 groups' 16 words once per round the same way Task 7 shuffles), tap-to-select exactly 4, "Confirmar grupo" checks against `grupos` client-side only to decrement chances and give immediate right/wrong feedback (not final grading — final `correta` is still computed by telão from whether all 4 groups ended up solved), locks solved groups visually.

- [ ] **Step 1: Implement the 16-word select-4 grid with the 7-chances counter and solved-group lock-in.**
- [ ] **Step 2: Wire submit-on-all-solved-or-chances-exhausted into `respostas[idx]`.**
- [ ] **Step 3: Reuse Task 6's telão grid/reveal/resolution, swapping in `avaliarQuatro`.**
- [ ] **Step 4: Manual verification** — solve a round correctly, fail a round (exhaust chances with ≤3 groups found), confirm grading and chances math both match.
- [ ] **Step 5: Commit** — `git commit -m "feat: four-groups-of-four round (Bloco 3)"`

---

### Task 9: Bloco Pistas

**Files:** Modify: `index.html` (telão's own clue-reveal timer separate from the team deadline, phone guess box, `avaliarPistas` wiring, "fora da rodada" exclusion)

**Interfaces:**
- Consumes: `avaliarPistas`, the decreasing point table `[10,8,6,4,2]` (index = clue count already revealed when the correct guess lands, 0-indexed), a per-round `eliminados: equipeIdx[]` (teams that guessed wrong once).
- Produces: telão reveals `round.pistas[i]` one at a time on its own short interval (not the full round deadline — e.g. a new clue every `tempoRodadaSeg/5` seconds, still bounded by the overall round deadline as a safety net); phone shows the *same* clue index (read from `revanche` state, not a local timer, so it never shows the clue ahead of the room) plus a text input + "Responder" button, disabled once that team is in `eliminados`; a correct guess at clue-index `i` scores `[10,8,6,4,2][i]` added directly to `equipes[idx].pontos` (outside the bet — matching "Valor: pista 1 = 10…"), sets `resolvido` for the round immediately (first correct answer ends it for everyone, like the spec says) and also counts as this round's "correct" for `acertosBloco`/speed-bonus purposes (the fastest-and-only correct guesser *is* the speed-bonus winner here, so the existing `primeiroAcerto` field still applies); a wrong guess adds that `equipeIdx` to `eliminados` and disables further guesses for them this round, without ending the round for everyone else.

- [ ] **Step 1: Telão clue-reveal ticker** (independent interval advancing a `pistaIndex` field, capped by the round's overall deadline as the hard stop if nobody ever answers).
- [ ] **Step 2: Phone guess box with eliminados-based disabling.**
- [ ] **Step 3: Resolution**: first correct guess *or* deadline, whichever first; apply points and the `resolvido` guard; move to `'resultadoRodada'`.
- [ ] **Step 4: Manual verification** — have two simulated teams guess wrong (confirm both get excluded and can't retry), then the right answer from a third (confirm points match the clue count used, and the round ends immediately for everyone rather than waiting for the other 7 teams).
- [ ] **Step 5: Commit** — `git commit -m "feat: progressive-clue round (Bloco 4 Pistas)"`

---

### Task 10: Resultado do bloco, placar animado, encadeamento

**Files:** Modify: `index.html` (telão's `'resultadoBloco'` screen, the bet-payout application, chaining into the next block's `'aposta'` or into the final/podium)

**Interfaces:**
- Consumes: `resolverAposta` (Task 2), `PlacarBars` (existing component, already does the arrow/position-change animation via `prevRanking`) — reuse it unchanged.
- Produces: telão resolution effect, firing once the 3rd round of a block resolves: snapshots `prevRanking = sortedEquipesIdx(equipes)` *before* applying payouts, applies `resolverAposta(apostas[idx], acertosBloco[idx], 3)` to every team's `pontos`, shows `'resultadoBloco'` for 12s (spec) rendering `PlacarBars` with the new `prevRanking`, then auto-advances: `blocoIndex+1` with a fresh `apostas:{}, respostas:{}, acertosBloco:{}, resolvido:null` and the new block's bet window — or, if `blocoIndex` was already 3 (last regular block), goes to `blocoIndex:4` (the final) instead; if it was already 4, goes to `fase:'podio'` (Task 11 builds that screen).

- [ ] **Step 1: Implement the payout application + `'resultadoBloco'` screen.**
- [ ] **Step 2: Implement the 12s auto-chain into the next block's bet window (or into the final/podium).**
- [ ] **Step 3: Manual verification** — finish Bloco 1 with a mix of winning/losing bets across the 10 teams; confirm each team's `pontos` moved by exactly the spec's doubling/halving formula, the bar chart's up/down arrows reflect the real position changes, and Bloco 2's bet window opens on its own 12s later.
- [ ] **Step 4: Commit** — `git commit -m "feat: block payout, animated scoreboard, auto-chain to next block"`

---

### Task 11: Final "Ezequiel Embaralhado" + pódio

**Files:** Modify: `index.html` (telão/phone branches for `blocoIndex===4`, podium screen rework)

**Interfaces:**
- Consumes: Task 7's reorder component (12 items instead of 5), `FINAL_GINCANA`, the existing `StageRevancheResultadoFinal`/`posicaoAntes` (lines 547-576 of current `index.html` — reuse almost as-is, it already computes "estava em Nº lugar antes").
- Produces: final round behaves like a single Cronos round with `tetoAposta:120` and all-or-nothing grading (`resolverAposta(aposta, acertos, 1)` from Task 2 already covers this — `acertos` here is `1` or `0`, never a 2-of-3 count); on the first team to get all 12 right, telão shows the fully-assembled text with `FINAL_GINCANA.referencia` (spec: "Quando a primeira equipe acertar, o telão mostra o texto inteiro montado"); the existing podium component runs unchanged once `fase:'podio'`.

- [ ] **Step 1: Wire `blocoIndex===4` into the bet flow (Task 5) and the Cronos-shaped round flow (Task 7) with the final's own timer/teto.**
- [ ] **Step 2: Add the "texto montado" reveal moment on first full-correct submission.**
- [ ] **Step 3: Confirm `StageRevancheResultadoFinal` renders correctly off the new `equipes`/`pontosIniciais` after a full run** (should need no changes, but verify).
- [ ] **Step 4: Manual verification** — run the final end to end, confirm the all-or-nothing payout, the assembled-text reveal, and the podium's "estava em Nº lugar" line against a known pre-final ranking.
- [ ] **Step 5: Commit** — `git commit -m "feat: final round and podium wiring"`

---

### Task 12: Controles de emergência restantes no painel

**Files:** Modify: `index.html` (`RevancheControl`, replacing what's left of the old provas UI)

**Interfaces:** Every control here writes directly to the fields already defined by Tasks 2-11; this task adds the UI and the handful of handlers not yet covered:
- Avançar/voltar bloco e rodada manualmente (force the same transitions the autonomous effects do, for recovery).
- Abrir/fechar aposta manualmente (force `fase` between `'aposta'`/`'revelacaoAposta'` regardless of the clock).
- Forçar fim de rodada (same as a timeout, callable early).
- Ajustar pontos manualmente por equipe (+/-, reuse existing `EquipesAdjust`/`adjustPonto`).
- Corrigir uma resposta marcada errada: flips `respostas[idx].correta`, **and** undoes/redoes that team's share of the already-applied block payout and any speed bonus — do the math as "recompute this team's bloco delta from scratch with the corrected `acertosBloco` and apply the difference", not a blind re-add.
- Mostrar o placar a qualquer momento (existing `modo:'placar'` tab already does this — just confirm it still works against the new `revanche.equipes`).
- Resetar a gincana sem perder o placar inicial: reset every new field to Task 2's initial shape but keep `equipes[i].pontos = equipes[i].pontosIniciais`.
- Atalhos de teclado: extend the existing `onKey` pattern (see `index.html:695-706`'s Premiação example) — Space already means something there, so pick distinct keys for the Revanche panel (document them next to the buttons, like the existing `kbd-hint`).

- [ ] **Step 1: Build the control panel UI for the above, replacing the old `overflowPlacar`/prova-by-prova UI.**
- [ ] **Step 2: Implement the "corrigir resposta" undo/redo-delta logic carefully** (this is the Review Focus item — write out the before/after math in a code comment since it's the one non-obvious algorithm in this task).
- [ ] **Step 3: Manual verification** — force-advance past a stuck team, correct a wrong grading after a block already paid out and confirm the scoreboard lands exactly where it would have if the correction had been right from the start, reset mid-game and confirm `pontos` return to `pontosIniciais` while `equipes` order/colors are untouched.
- [ ] **Step 4: Commit** — `git commit -m "feat: remaining panel overrides for gincana operation"`

---

### Task 13: Modo Simulação

**Files:** Modify: `index.html` (`App()` router gains `view=simulacao`; new `SimulacaoView`)

**Interfaces:**
- Consumes: everything from Tasks 2-12, but pointed at a separate Firebase path.
- Produces: `const basePath = isSim ? 'talkflow/talkflow03-sim' : 'talkflow/talkflow03';` thread through `stateRef` the same way `overflow-awards` threads `isTeste` (see `overflow-awards/index.html:173-174` for the exact pattern); `?simulacao=1` on `?view=telao`/`?view=painel` points those at the sim path too (so you can watch a full simulated run on a second screen), and `?view=simulacao` is a single-page UI listing all 10 teams as tabs — tapping a tab lets the operator act as that team (bet, answer, drag-reorder) without the QR/color-claim flow, writing straight to that team's `apostas[idx]`/`respostas[idx]` in the sim path. A "Resetar simulação" button wipes the sim path back to initial state (`remove(dbRef(db, 'talkflow/talkflow03-sim'))`, same pattern as `overflow-awards`' `limparTudo`).

- [ ] **Step 1: Thread the sim/real path switch through every `dbRef`/`stateRef` use.**
- [ ] **Step 2: Build `SimulacaoView`'s 10-team-tab UI, reusing `TimeView`'s per-`tipo` round components with a team-select prop instead of the localStorage-id lookup.**
- [ ] **Step 3: Add the reset button.**
- [ ] **Step 4: Manual verification** — run the entire Premiação→Entrada→4 blocos→Final→Pódio flow solo from `?view=simulacao` plus `?view=telao&simulacao=1`, confirm nothing touches the real `talkflow/talkflow03` path (check Firebase console or just confirm the real-path telão stays untouched throughout), then reset and confirm the sim path is empty again.
- [ ] **Step 5: Commit** — `git commit -m "feat: solo simulation mode for pre-event testing"`

---

### Task 14: Sons, README, teste de fluxo completo

**Files:** Modify: `index.html` (sound helpers), `README.md`

**Interfaces:** Consumes the existing `beep`/`playSoundByName` pattern (`index.html:273-303`) — add `playAberturaAposta`, `playFimTempo` (already exists, reuse), `playAcerto`, `playErro`, `playRevelacao`, `playCelebracaoFinal` (reuse existing `playCelebracao`) following the same oscillator-beep style as the existing five sounds, wired into the same `sound`/`soundSeq` state field so every client that plays sound already has the plumbing.

- [ ] **Step 1: Add the 3-4 missing sound functions and wire them into the relevant telão transitions from Tasks 5-11** (bet window opens, wrong answer, correct answer, result reveal).
- [ ] **Step 2: Rewrite `README.md`'s relevant sections**: the three roles (telão/painel/time), how a team joins, how each block is operated day-of (should mostly run itself — mention PAUSAR), how to run `?view=simulacao` before the event, and the corrected claim that the telão now writes state for the Revanche portion.
- [ ] **Step 3: Full manual flow test**, exactly as the spec's own "ENTREGA" section asks: Premiação ao vivo → QR → 10 times entram → 4 blocos → final → pódio, in one continuous run, using `?view=simulacao` to stand in for the 10 phones and real `?view=telao`/`?view=painel` tabs alongside it. Fix anything that breaks before calling this done.
- [ ] **Step 4: Commit** — `git commit -m "feat: gincana sounds, README, end-to-end verification"`
