# Auditoria de Tratamento de Erros — Sunny Sistema

## Resumo executivo

Mapeei ~135 operações async/risco no `index.html`. **O problema dominante NÃO é falta de `try/catch`** — é o oposto: `sv()` e `FB.get/set/del` têm `try/catch` *internos* que **engolem o erro** (só `console.error`), e os handlers de UI mostram `fl("✅ Salvo!")` **incondicionalmente** depois do `await sv(...)`. Resultado: quando o save falha, o usuário vê confirmação de sucesso e o dado **não foi gravado**. Esse é o cenário que a manager nova vai bater contra.

Secundariamente: ~22 PDFs abrem via `window.open` sem fallback se popup for bloqueado (usuário clica, nada acontece).

## Estatísticas

| Métrica | Quantidade |
|---|---|
| `await FB.get/set/del` | 63 |
| `await sv(...)` (linhas distintas) | 47 (≈60 chamadas individuais) |
| `await fetch(...)` | 6 |
| `FB.set(...)` direto (não via sv) | 14 |
| `FB.get(...)` direto (boot + outros) | 55 |
| `window.open("","_blank")` para PDF | 22 |
| `}catch{}` vazio | 41 |
| `catch(e){console.x(...)}` (só log) | 3 |
| `.catch(...)` promise chain | 1 (no SW register) |
| Toasts de sucesso (`fl("✅ ...")`) | 200 |

**Com feedback de erro real ao usuário:** apenas os 4 fetches de AI (chat, fbAi, SOP gen, report gen) e o LoginScreen. **Estimo <5% das operações de save expõem erro ao usuário.** O resto cai silencioso (success toast otimista ou nada).

## Como `sv()` e `FB.*` falham hoje

```js
// linha 769 — sv() captura e SÓ loga, nunca lança
const sv=async(k,d)=>{try{...await FB.set(k,JSON.stringify(d));...}catch(e){console.error("Erro ao salvar "+k+":",e);}};

// linha 28 — FB.get/set retornam null silencioso
FB={get:async(k)=>{try{...}catch(e){return null;}}, set:async(k,v)=>{try{...}catch(e){return null;}}, ...}
```

Consequência: `await sv("fn3", u);fl("✅ Lançamento salvo!");` **sempre** mostra "✅ salvo" — mesmo quando não salvou.

## Inventário por categoria

### 1. Firestore reads no boot (linhas ~698-757) — 63 ocorrências

| Padrão | Quantidade | try/catch? | Feedback? | Criticidade |
|---|---|---|---|---|
| `try{const r=await FB.get("xxx");if(r&&r.value)setX(JSON.parse(r.value));}catch{}` | ~30 | Sim, vazio | Nenhum | Média |
| Boot com migrations + auto-fix (linhas 700, 703, 720, 738) | 4 | Sim, vazio | Nenhum | Média |
| Recovery loop de `_pend_*` (linha 755) | 1 | Sim, `console.log` | Nenhum | Alta |
| Auto-backup semanal (linha 757) | 1 | Sim, vazio | Nenhum | Baixa |
| Auto-faturamento triggers (linhas 769) | 2 | Sim | Nenhum | Alta |

**Diagnóstico:** maioria pode ficar silenciosa (key vazia = default OK), mas o **recovery loop** e os triggers de auto-faturamento são críticos — se falharem o usuário não sabe.

### 2. Saves no boot — 11 ocorrências (linhas 700-755, 770)

Todas são **auto-migrations de dados antigos**. Se uma falhar, o usuário não pode fazer nada — não é ação dele. Console.error suficiente.

### 3. Save handlers de UI — ~36 ocorrências (linhas 795-905, 1084-1325, 1618, 3520)

**Aqui mora o problema.** Padrão dominante:

```js
const savP = async(status) => {
  ...
  setSP(u); await sv("sp3", u);
  setFin(uf); await sv("fn3", uf);
  fl("✅ Folha emitida!");  // SEMPRE roda, mesmo se sv() falhou silenciosamente
};
```

| Handler | Linha | Risco se falhar |
|---|---|---|
| `savP` (folha pagamento) | 795 | Folha "salva" some no próximo refresh — perda de horas de trabalho |
| `savI` (invoice) | 869 | Invoice perdida — cobrança não acontece |
| `emitirInv` / `pagarInv` | 871, 872 | Status de pagamento perdido — confusão financeira |
| `delP` / `delI` | 879, 880 | Deleção parcial — dado fantasma |
| `addFin` / `delFin` / `updateFinEntry` / `dupFin` | 885-889 | Lançamento financeiro perdido |
| `syncFin` (rebuild financeiro) | 891-893 | Rebuild parcial — financeiro inconsistente |
| `uPr` / `uLn` / `uLB` / `uLoss` / `uCC` / `uCU` (updates inventário) | 900 | Mudança no estoque perdida |
| `uInsp` / `uAg` / `uTk` | 903-905 | Inspeção/agenda/task perdida |
| iCal sync (várias linhas) | 1084-1128 | Sync iCal acha que sincronizou, mas turnover não persiste |
| Auto-faturamento triggers | 1281-1328 | Trigger automático falha silente |

### 4. `sv()` fire-and-forget (sem await) — ~5+ ocorrências críticas

Linhas 877 e 878 (`autoGenPayrolls`, `autoGenInvoices`):
```js
if(created>0){setSP([...sP]);sv("sp3",sP);  // SEM await
  fl("✅ "+created+" folha(s) gerada(s)!");
```
Linha 901 (`toTrash`): `try{sv("_trash",t);}catch{}` — pegou erro síncrono, mas a Promise rejection escapa.

### 5. HTTP fetch — 6 await + 2 dentro de helpers

| Linha | Onde | try/catch? | Feedback? | Criticidade |
|---|---|---|---|---|
| 957, 963 | `fetchICalURL` (iCal proxy) | Sim, **lança erro descritivo** | Depende do caller | Média |
| 4948 | AI chat (`askQuestion`) | Sim, mostra erro no próprio chat | **Sim, no chat** | OK |
| 4969 | SOP AI gen (dentro do modal) | Sim, salva em `sopEdit.lastError` | **Sim, banner vermelho** | OK |
| 5483 | Feedback AI insight | Sim, mostra "❌ Erro" no card | **Sim** | OK |
| 6007 | Report AI gen | Sim com `AbortController` 55s | Provável (revisar) | OK |

**iCal sync** é o caso a verificar: o caller precisa renderizar o erro. (Vi na auditoria anterior que existe um log de sync visível — provavelmente OK.)

### 6. PDF / Canvas generation — ~22 `window.open` + `gImg` (canvas)

Padrão típico (linha 815, `genEstPDF`):
```js
var w=window.open("","_blank");
if(w){w.document.write(h.join(""));w.document.close();setTimeout(()=>w.print(),600);}
```

Quando `w` é `null` (popup blocker), **nada acontece, sem aviso**. Em 22 botões.

`gImg` (canvas → PNG download, linha 437) não tem fallback se `cv.toDataURL` lançar (CORS no `<img>` do logo, p.ex.) — embora hoje use logo same-origin.

### 7. Auth — 3 chamadas

- `auth.signInWithEmailAndPassword` (linha 399) — `try/catch` com 4 mensagens específicas por código. **OK.**
- `auth.signOut()` (linha 406) — sem error handling, mas signOut raramente falha.
- `auth.onAuthStateChanged` (linha 403, dentro de `try/catch` para carregar role).

## Top 10 críticas (priorizar nesta ordem)

1. **`savP` (linha 795)** — folha de pagamento. Perder uma folha é horas de trabalho do usuário descartadas. **Toast otimista incondicional.**
2. **`savI` / `emitirInv` / `pagarInv` (869-872)** — invoices/cobranças. Perder = problema financeiro com cliente.
3. **`addFin` / `updateFinEntry` / `delFin` (885-888)** — lançamentos financeiros. Auditoria contábil quebra se silenciosamente perde entradas.
4. **iCal sync (1084-1128)** — turnovers do Airbnb fantasmas. Verificar se já há feedback no log de sync.
5. **Auto-faturamento triggers (1281-1328)** — automação invisível ao usuário. Precisa pelo menos notificação no boot se falhar.
6. **PDF generators (22 lugares)** — popup blocker é silencioso. Pelo menos 5 PDFs principais (`genEstPDF`, payslip via `gImg`, invoice, tax, inspection report) precisam fallback.
7. **`uPr` / `uLn` / `uLB` / `uLoss` (linha 900)** — updates de inventário em massa. Erro silencioso → estoque incorreto.
8. **`syncFin` (891-893)** — rebuild de financeiro. Se falhar no meio, dados ficam inconsistentes sem aviso.
9. **`uInsp` / `uAg` / `uTk` (903-905)** — saves de inspeção/agenda/tasks.
10. **`autoGenPayrolls` / `autoGenInvoices` (877-878)** — fire-and-forget `sv()`. Toast diz "✅ geradas!" e nem sequer espera o save.

## Padrões problemáticos identificados

- **A. Toast otimista após `await sv()` silencioso** — ~30+ handlers. O usuário sempre vê "✅ salvo".
- **B. `}catch{}` vazio em boot** — 41 ocorrências. Aceitável para fallback de defaults, problemático se mascarar JSON corrompido.
- **C. `window.open` sem fallback de popup blocker** — 22 lugares.
- **D. `sv()` fire-and-forget em handler** — ≥5 lugares. Promise rejection escapa silenciosa.
- **E. `console.error/log` sem toast** — 3 catches diretos. (Inúmeros também dentro de `sv` e `FB`.)

## Recomendação técnica

A causa raiz é arquitetural: `sv()` engole o erro. Não dá pra resolver só envolvendo os callers em try/catch — eles **não recebem** o erro. Precisa intervir na origem.

Proposta em **duas peças** que cabem em poucas linhas (estilo do arquivo):

### Peça 1 — `svT(k, d)` que LANÇA (e mantém `sv` antigo intacto)

```js
const svT=async(k,d)=>{if(CRITICAL_KEYS.includes(k)&&Array.isArray(d)&&d.length===0){const x=await FB.get(k);if(x?.value){const old=JSON.parse(x.value);if(Array.isArray(old)&&old.length>0)throw new Error("BLOQUEADO: array vazio em "+k);}}const r=await FB.set(k,JSON.stringify(d));if(r===null)throw new Error("Falha ao salvar "+k);setSaveFlash(true);setTimeout(()=>setSaveFlash(false),800);};
```

Mas `FB.set` hoje retorna `null` silencioso — precisa um mini-refactor para FB também lançar, ou `svT` precisa verificar com `FB.get` (caro). Mais simples: **mudar `FB.set` para lançar em vez de retornar null** (efeito colateral: chamadas existentes de `FB.set` direto também passariam a lançar — checagem extra necessária).

### Peça 2 — wrapper `safe(fn, okMsg, errMsg)` para handlers

```js
const safe=async(fn,okMsg,errMsg)=>{try{await fn();if(okMsg)fl(okMsg);}catch(e){console.error(e);fl("❌ "+(errMsg||"Erro")+": "+(e?.message||"falha"));}};
```

Uso (compara antes/depois):

```js
// Antes (linha 869)
const savI=async(status)=>{...setSI(u);await sv("si3",u);...fl("📨 Invoice emitida!");};

// Depois
const savI=async(status)=>{await safe(async()=>{...setSI(u);await svT("si3",u);...},"📨 Invoice emitida!","Falha ao salvar invoice");};
```

### Peça 3 — helper `popup(title)` para PDFs

```js
const popup=(title)=>{const w=window.open("","_blank");if(!w){fl("⚠️ Permita popups deste site para gerar "+(title||"PDF"));return null;}return w;};
```

Trocar 22 ocorrências de `var w=window.open("","_blank");if(!w)return;` por `const w=popup("Invoice");if(!w)return;` (linha extra: shows toast).

### Estimativa de código novo

- `svT` + `safe` + `popup` + ajuste em `FB.set`: ~12 linhas adicionadas.
- Migração dos 10 handlers críticos: ~30 linhas tocadas.
- Migração dos 22 PDFs: trocar string em cada lugar (replace_all candidato).
- Total esperado: ~50-70 linhas de mudança líquida, em 3-5 commits.

## Áreas que NÃO precisam tratamento mais agressivo

- **Boot useEffect com `catch{}`** — fallback para defaults é o comportamento certo. Único acréscimo razoável: agregar erros e mostrar uma única toast "⚠️ Alguns dados não carregaram (X falhas)" se contagem > 0.
- **Auto-backup semanal (linha 779)** — falha tolerável, console suficiente.
- **`localStorage.setItem` em `dsv`** — fallback para Firestore depois cobre. Console suficiente.
- **`auth.signOut()`** — operação cliente-side, falha rara.

## Bugs colaterais detectados (informativo, não corrigir agora)

1. **Linha 901**: `toTrash` faz `try{sv("_trash",t);}catch{}` — pega erro síncrono mas a Promise rejection passa. Hoje funciona porque `sv` engole; após mudança em `sv`, vai escapar como unhandled rejection.
2. **Linhas 877-878** (`autoGenPayrolls`/`autoGenInvoices`): chamam `sv("sp3", sP)` SEM `await` e em seguida mostram toast "✅ geradas!". Toast antes do save terminar.
