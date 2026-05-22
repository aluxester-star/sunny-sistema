# REDESIGN-LOG.md — Sidebar + Header

**Branch:** `feature/redesign-sidebar-header` (a partir de `81ab70c`)
**Início:** 2026-05-20 — sessão presencial com Eduardo
**Escopo:** sidebar fixa esquerda + header fixo topo. NÃO toca em conteúdo das telas.

---

## 1. Estado real do `index.html`

Total: **6.838 linhas**. Branch ainda em `81ab70c` (não tem os 2 commits novos de `main` — só CLAUDE.md/docs diferem; o `index.html` é idêntico).

### Pontos críticos confirmados

| Item | Linha real | Notas |
|---|---|---|
| `allTabs` (18 abas) | **1352** | confirma spec |
| `tabs` filter por role/customTabs | **1353** | reuso direto na sidebar |
| `getInvoicesDueToday()` | **1136** | reuso no sininho |
| `getPayrollDueToday()` | **1201** | reuso no sininho |
| Header sticky atual (a substituir) | **1614** | gradient + sticky + zIndex 100 |
| Tab nav scroll horizontal (a remover) | **1693–1704** | inclui badges por aba |
| Content wrapper principal | **1705** | `maxWidth:1020,margin:"0 auto",padding:"12px 8px",paddingBottom:60` |
| **Bottom nav real** (`className="np"`) | **6825** | spec dizia ~6142, errado |
| Spacer abaixo da bottom nav | **6828** | `<div style={{height:52}}/>` remover junto |
| Props `onLogout`, `userEmail`, `userRole`, `customTabs` | App em **646** | tudo já disponível |
| Dark mode state/toggle | `darkMode`/`setDarkMode` em **647** + toggle inline no Dashboard em **1709** | migra pro header novo |

### Três `<div className="np">` no arquivo — só uma vai sair

- **1360** e **1377** → faixas sticky-top com botões `← Back / 📥 PDF` dos modos de impressão `payslip` e `preview`. **NÃO MEXER.**
- **6825** → bottom nav real. **REMOVER no Commit 5.**

O CSS `@media print{.np{display:none!important}}` (linhas 1372 e 1390) continua válido pros stickies dos PDFs — sem colisão.

### Returns antecipados (sidebar/header NÃO devem aparecer aqui)

- `if(vw==="payslip")` em **1358** → return próprio
- `if(vw==="preview")` em **1375** → return próprio

Ambos saem do componente antes do `// === MAIN ===` (1392). Sidebar/header serão inseridos **dentro** do return principal a partir de 1394 — não afetam payslip/preview.

---

## 2. Plano de inserção — anatomia do return principal

Estrutura atual (1394–6830):

```
<div minHeight bg font color>           ← root (1394)
  <link DM Sans/>                       ← (1395)
  <style global/>                       ← (1396) transitions
  <ModalC/>                             ← (1397)
  {scanModal && ...}                    ← (1399–1432)
  <FAB scan> <FAB search>               ← (1435, 1439)
  {showSearch && ...} {clientReport && ...} ...   ← modais
  {toast} {saveFlash}                   ← (1612, 1613)
  ┌─ HEADER STICKY ATUAL (1614)        ← VAI SUMIR no C3
  ├─ TAB NAV SCROLL (1693–1704)        ← VAI SUMIR no C5 (proposto, ver §6)
  └─ <div maxWidth 1020 ...>            ← content wrapper (1705) — ajusta padding-left
       ... todas as views (vw==="..." && ...) ...
     </div>                             ← (6823)
  ┌─ BOTTOM NAV `np` (6825)             ← VAI SUMIR no C5
  └─ <div h:52/> spacer (6828)          ← VAI SUMIR no C5
</div>                                  ← root close (6830)
```

Estrutura alvo:

```
<div root>
  <link/> <style/> <ModalC/> modais ... toasts
  ┌─ <Sidebar/>                         ← INSERE em C1, estiliza em C2
  ├─ <Header/>                          ← INSERE em C3
  └─ <div content wrapper>              ← AJUSTA padding-left:220px, padding-top:60px (C1)
       views ...
     </div>
  {/* bottom nav e tab nav removidos em C5 */}
</div>
```

---

## 3. Snippets ANTES/DEPOIS dos pontos críticos

### 3a. Content wrapper (linha 1705) — Commit 1

**ANTES:**
```jsx
<div style={{maxWidth:1020,margin:"0 auto",padding:"12px 8px",paddingBottom:60}}>
```

**DEPOIS:**
```jsx
<div style={{maxWidth:1020,margin:"0 auto",padding:"12px 8px",paddingBottom:60,paddingLeft:sbMobile?8:228,paddingTop:72}}>
```

(`sbMobile` = `window.innerWidth < 768`, calculado em useState+resize listener no C4. Em C1 deixo `paddingLeft:228,paddingTop:72` fixo e ajusto em C4.)

### 3b. Bottom nav (linhas 6825–6828) — Commit 5

**ANTES:**
```jsx
<div className="np" style={{position:"fixed",bottom:0,...}}>
  {[{k:"N",l:"Folha",i:"💰",g:"payroll"},...].map(a=><button .../>)}
</div>
<div style={{height:52}}/>
```

**DEPOIS:** (remover ambos os blocos — 4 linhas a menos)

### 3c. Header sticky atual (linha 1614) — Commit 3

**ANTES:** bloco de ~80 linhas (1614–1692) com nome empresa, hamburger antigo, etc.

**DEPOIS:** substituído por `<Header/>` com título dinâmico via `vw`, seletor PT/EN/ES visual, sininho, toggle dark, avatar dropdown.

### 3d. Toggle dark dentro do Dashboard (linha 1709) — Commit 3

**ANTES:**
```jsx
<button onClick={()=>{const nv=!darkMode;setDarkMode(nv);...}} style={{...}}>{darkMode?"☀️":"🌙"}</button>
```

**DEPOIS:** removido daqui (vai pro Header). Lógica `setDarkMode + localStorage.sunny_theme` segue idêntica.

---

## 4. Estrutura de dados — 5 grupos da sidebar

```js
const SB_GROUPS=[
  {id:"principal",  icon:"🏠", label:"PRINCIPAL",      items:["dashboard","agenda","inspect"]},
  {id:"financeiro", icon:"💰", label:"FINANCEIRO",     items:["invoice","payroll","finance","tax"]},
  {id:"estrategia", icon:"📈", label:"ESTRATÉGIA",     items:["strategy","report","feedback"]},
  {id:"operacao",   icon:"⚙️", label:"OPERAÇÃO",       items:["team","escala","laundry","deepclean","manual"]},
  {id:"config",     icon:"🗄️", label:"CONFIGURAÇÕES",  items:["database","icalsync","history"]}
];
```

Total 18 items = `allTabs.length`. Cada item se cruza com `tabs` (já filtrado por role/customTabs em 1353) para respeitar permissões.

---

## 5. Novos states a adicionar (perto de `darkMode`, ~647)

```js
const[sbGroups,setSbGroups]=useState(()=>{
  try{return JSON.parse(localStorage.getItem("sidebar_collapsed_groups")||'{}');}
  catch{return{};}
});
// chave = id do grupo, valor = true se COLAPSADO. Default ausente = aberto para "principal", fechado p/ resto.
const[sbOpen,setSbOpen]=useState(false);      // C4 — drawer mobile aberto?
const[lang,setLang]=useState(()=>{try{return localStorage.getItem("lang")||"pt";}catch{return"pt";}});
const[notifOpen,setNotifOpen]=useState(false);
const[profOpen,setProfOpen]=useState(false);
```

Helper auxiliar:
```js
const sbIsOpen=(gid)=>gid==="principal" ? sbGroups[gid]!==true : sbGroups[gid]===false;
const toggleSbGroup=(gid)=>{
  const nv={...sbGroups,[gid]:sbIsOpen(gid)};
  setSbGroups(nv);
  try{localStorage.setItem("sidebar_collapsed_groups",JSON.stringify(nv));}catch{}
};
```

Grep prévio: `sbGroups|sbOpen|notifOpen|profOpen` → **zero ocorrências** (sem colisão).

---

## 6. Decisões aprovadas pelo Eduardo (2026-05-20)

1. **Tab nav scroll horizontal global (1693–1704)** → **REMOVER no C5** (vira redundante com sidebar). Eduardo confirmou: "Remove só tab nav do TOPO/MENU PRINCIPAL, mantém sub-tabs dentro das telas" — ou seja, NÃO mexer em abas internas de cada view.
2. **FABs scan/search (linhas 1435, 1439)** → **MOVER `bottom:80` → `bottom:16`** dentro do C5 (quando a bottom nav sair).
3. **`sidebar_collapsed_groups` default** → **IMPLÍCITO** (chave nasce vazia no localStorage; helper `sbIsOpen` infere "principal aberta, resto fechado" pelo nome).

---

## 7. Riscos e mitigações

| # | Risco | Mitigação |
|---|---|---|
| R1 | Babel parser falhar (JSX mal-formado) | Validação por commit: extrai `<script id="app-jsx">`, escreve em `.tmp-jsx`, roda `npx --yes @babel/parser` via node one-liner. Máx 3 tentativas, reverte buffer entre cada. |
| R2 | Z-index conflitos | Sidebar zIndex 101, Header zIndex 102, drawer overlay 103. Modais (9996–9999) ficam por cima. |
| R3 | Returns antecipados (payslip/preview) | Sidebar/header só dentro do return principal pós-1394. Returns em 1358/1375 ficam intocados. |
| R4 | Customer login (`cliente:Xxx`) ou customTabs restritos | Filtro `SB_GROUPS[*].items.filter(id => tabs.find(t=>t.id===id))` antes de renderizar — grupos vazios viram invisíveis. |
| R5 | Dashboard quebrado por remoção do toggle dark inline | Garantir em C3 que `setDarkMode` ainda existe no escopo (não move state, só move botão). |
| R6 | Service worker servir HTML cacheado pós-mudança | **Spec proíbe bumpar cache** (`sunny-v4`). Eduardo aceita refresh manual em produção. Confirmado restrição #5. |
| R7 | Sininho lento se `agEvents` enorme | `getInvoicesDueToday()` e `getPayrollDueToday()` já são memoizados via `useMemo` no boot; turnover sem inspeção filtra com `Array.find` simples. Tudo derivado em render — sem useState/useEffect novo. |
| R8 | Mobile drawer fica preso aberto se overlay click não pega | Overlay com `onClick={()=>setSbOpen(false)}` e `e.stopPropagation()` no conteúdo da sidebar (pattern já usado nos modais). |

---

## 8. Estimativas e validação por commit

| Commit | Escopo | Linhas afetadas (aprox.) | Tempo estimado | Validação |
|---|---|---|---|---|
| **C1** Estrutura JSX sidebar | inserir bloco sidebar (sem estilos finais), `SB_GROUPS`, ajustar padding do content wrapper | +60 / -1 | 15 min | babel parser ok + abrir `index.html` mentalmente |
| **C2** Estilos finais + state + animação + localStorage | states `sbGroups`, helpers, classes inline, transição `max-height` | +25 / ~20 | 20 min | babel parser ok |
| **C3** Header completo | substitui header 1614, remove toggle dark inline 1709, state `lang`/`notifOpen`/`profOpen`, sininho com getInvoicesDueToday + getPayrollDueToday + turnovers | +130 / -80 | 30 min | babel parser ok |
| **C4** Mobile responsivo | state `sbMobile`, drawer offcanvas, overlay, hamburger handler, padding ajustado | +35 / -3 | 20 min | babel parser ok |
| **C5** Remover bottom nav + tab nav global + FABs `bottom:16` | deleta 6825–6828 e 1693–1704; ajusta 1435/1439 | +0 / -15 + 2 edits | 10 min | babel parser ok + grep `className="np"` confirma só 1360/1377 |

**Total: ~95 min efetivo + aprovação Eduardo entre commits.**

---

## 9. Restrições absolutas (lembrete)

1. ❌ Nunca push pra origin (commits ficam locais)
2. ❌ Não toca em conteúdo das telas (Dashboard, Folha, Invoice, etc.)
3. ❌ Não refatora variáveis (`vw`, `co`, `db`, `em`)
4. ❌ Não mexe em Firebase/save/load
5. ❌ Não bumpa service worker (`sunny-v4` permanece)
6. ❌ Não adiciona dependências (`npm install` proibido)
7. ⚠️ Babel parser falhar → reverte buffer, máx 3 tentativas
8. ✅ Eduardo aprova cada commit antes do próximo

---

**Status: aguardando OK do Eduardo nas 3 decisões pendentes (§6) antes de iniciar Commit 1.**
