# Auditoria de Higiene de Código — Nível 1

## Resumo executivo

O `index.html` é um SPA single-file de ~6.853 linhas em produção, sem build/lint/test, com estilo deliberadamente comprimido (one-liners, variáveis de 1-2 letras). A auditoria identificou **higiene viável de melhorar sem risco arquitetural**: ~137 cores hex únicas espalhadas (vs. paleta `T`/`TD`/`TL` que tem só 12-13 tokens), 41 blocos `try{const r=await FB.get(...)}` quase-idênticos no boot, ~13 estados/variáveis confirmadamente mortos (declarados e nunca usados), 21 linhas vazias suspeitas (671-691) com um `;` órfão na linha 692 e várias inconsistências de PDF (cores `#0094B0` no canvas/HTML vs `T.ac` no app). **Recomendação de prioridade:** (1) remover dead code, (2) corrigir o bug do address hardcoded no `genEstPDF`, (3) só depois mexer em cores/duplicação — estas últimas têm risco visual maior em produção.

## Estatísticas

- Linhas em `index.html`: **6.853**
- Cores hex únicas encontradas (case-insensitive): **137** (`#000`, `#fff`, `#FFFFFF` contam separadas dos shorthand; veja apêndice abaixo)
- Cores hex que duplicam valores do tema T (`TD`/`TL`): pelo menos **12** (`#00B4D8`, `#0094B0`, `#0A0F1E`, `#111833`, `#1A2545`, `#182040`, `#E8EBF0`, `#707A8A`, `#FF6B6B`, `#4ADE80`, `#FFB547`, `#A78BFA`, `#60A5FA` — todas com **>20 ocorrências cada** fora do bloco `TD`/`TL`)
- Variáveis de estado de 1-2 letras dentro de `App`: **~50** (de ~77 chamadas `useState` totais no arquivo)
- Funções/blocos sem header comment marker (estilo `// =====`): **~10** blocos principais (`vw==="strategy"`, `vw==="database"`, `vw==="icalsync"`, helpers `gImg`, `genQRDataURL`, `genMonthlyReport`, `genTag`, `genEstPDF`, `genSopPDF`, init `useEffect`)
- Itens potencialmente dead code (variáveis + comentários + linhas vazias): **~13 estados mortos confirmados + 21 linhas vazias + 1 setor de `<script>` sobrando**

---

## a) Constantes hardcoded

### Achados

**a.1 Cores hex que duplicam o tema `T`** (mais sério — quebra dark/light mode)

| Cor hex | Equivale a | Ocorrências aprox. | Exemplos `line` |
| --- | --- | --- | --- |
| `#00B4D8` | `TD.ac` / `T.ac` | ~30 | 192, 203, 217, 233, 266, 442, 446, 479, 1388, 6098 |
| `#0094B0` | `TL.ac` / "hover" do ac | ~50 | 462, 1381, 1399, 1407, 1421, 1492, 6188, 6347, etc. |
| `#0A0F1E` | `TD.bg` / `T.bg` | ~25 | 2, 192, 196, 217, 237, 348, 1383, 1388, 6149 |
| `#111833` | `TD.cd` / `T.cd` | ~12 | 193, 210, 251, 259, 350, 1388 |
| `#1A2545` | `TD.bd` / `T.bd` | ~25 | 199, 210, 217, 244, 350, 1710 |
| `#182040` | `TD.c2` / `T.c2` | ~5 | 416, 528, 553, 3845, 4505, 5118 |
| `#E8EBF0` | `TD.tx` / `T.tx` | ~15 | 193, 204, 211, 233, 358, 480 |
| `#707A8A` | `TD.dm` / `T.dm` | ~22 | 194, 199, 214, 223, 230, 508, 530, 545 |
| `#FF6B6B` | `T.rd` | ~30 | 213, 228, 491, 597, 606, 612, 614, 621, 635, 638, 1718, 1812, 1817 |
| `#4ADE80` | `T.gr` | ~25 | 194, 491, 573, 605, 614, 638, 645, 1812, 1817, 3859, 3974 |
| `#FFB547` | `T.or` | ~12 | 491, 554, 578, 597, 606, 1727, 1812, 1817, 3861 |
| `#A78BFA` | `T.pu` | ~15 | 416, 645, 1457, 1774, 1797, 1949, 1957, 4503, 4521, 4733 |
| `#60A5FA` | `T.bl` | ~10 | 416, 1727, 1775, 3606, 3654, 4349 |
| `#fff` / `#FFF` / `#FFFFFF` | branco | ~30+ | 266, 392, 399, 442, 444, 466, 1380, 1383, 1384, 6079 |

**a.2 Cores hex únicas que NÃO estão em `T`** (paleta "off-theme" hardcoded — alguns provavelmente intencionais em PDFs, mas vale catalogar):

- Tons gold: `#FFD700`, `#DAA520` (linhas 1634, 5129, 5153, 5181, 5219, 5220, 5222, 5233, 5256) — bloco de "podium/award" na escala
- Tons medalha: `#C0C0C0`, `#CD7F32`, `#FFD700` (linha 1829) — usado uma única vez (top-3 ranking)
- Verde-bem-escuro: `#15803D`, `#16A34A`, `#22C55E`, `#2E7D32` (várias) — outra "escala de verde" paralela ao `T.gr`
- Roxos extras: `#A855F7`, `#8B5CF6`, `#EA80FC`, `#E879F9` (645, 798, 843, 1457, 4945, 4991) — usados em badges de role/category
- WhatsApp green: `#25D366` (1381, 2002) — específico da marca
- Cores PDF cinzas: `#222`, `#333`, `#444`, `#555`, `#666`, `#777`, `#888`, `#999`, `#aaa`, `#bbb`, `#ccc`, `#ddd`, `#eee` (centenas de ocorrências, **só nos PDFs gerados via `window.open`**) — esses são aceitáveis (PDF não respeita tema dark)
- Tons "ice"/"creme": `#F0FDF9`, `#E0F7FA`, `#F0FDF4`, `#FFF8E1`, `#FFFFFFCC`, `#0A0F1EEE` (várias) — esses são variações com alpha que deveriam virar tokens `T.acBg`, etc.

**a.3 Strings/URLs/configs duplicadas**

| Item | Linha(s) | Status |
| --- | --- | --- |
| Logo URL `https://sunny-sistema.vercel.app/sunny-logo.png` | 427 (`LOGO`), 836 (redeclarado dentro de `genEstPDF`) | duplicado |
| Logo SVG `LOGO_URL` (data:image base64) | 838 | OK |
| Domínio Sunny `https://sunny-sistema.vercel.app` | 427, 836, 869 (`SUNNY_URL`) | duplicado entre `LOGO` e `SUNNY_URL` |
| Address `"3272 N Arlington Ave. - FL 33603"` | 415 (`DC.address`) | OK — usado via `co.address` |
| **Address conflitante** `"2501 Orient Rd, Tampa, FL 33619"` | 836 (hardcoded dentro do `genEstPDF`) | **BUG** — não bate com `DC.address` |
| Phone `813-546-2577` | 415 (`DC.phone`), 836 (hardcoded dentro do `genEstPDF`) | duplicado |
| `"SUNNY CLEANING SERVICES"` (fallback) | 415, 444, 836 | duplicado |
| Font URL DM Sans | 2, 1377, 1394, 1413 (4×) | repetida |
| Font URL Inter | 843 (PDF SOP), 1513 (outro PDF) | repetida |
| Cache name `sunny-v3` | 6852 | OK (CLAUDE.md instrui bump manual) |
| Allorigins proxy URL | 984 | OK (única) |

**a.4 Números mágicos**

| Valor | Significado | Onde |
| --- | --- | --- |
| `3e3` | 3000ms toast | 794 (`fl`) |
| `1500` | 1500ms debounce dsv default | 792 |
| `800` | 800ms saveFlash | 791, 1576 |
| `600` | 600ms PDF print delay | 837, 871 (genEstPDF, genMonthlyReport) |
| `700` | 700ms PDF print delay | 843 (genSopPDF) — inconsistente com 600 |
| `1000` | 1000ms QR/tag print delay | 809 |
| `100` | 100ms QR generation polling | 870 |
| `7*24*60*60*1000` | 1 semana ms | 779, 1181 (uma como literal, uma como string `7*86400000` — inconsistente) |
| `86400000` | 1 dia ms (literal) | 861, 863, 1181, 3946 |
| `0.92` | scale animation | 2 |
| `0.5` | JPEG compression (inspeção) | 619 |
| `400` | maxImage size (compressão) | 619 |
| `100` | autoTriggers slice cap | 1306, 1345 |
| `20` | autoTriggers slice (icalLog) | 1107, 1108 |
| `12` | suggestions slice limit | 863 |
| `40000` / `50000` / `80000` | metas hardcoded | 1815 (META=50000), 5724 (`GOAL_12MO=80000`) |
| `7000` | prolabore default | 5724 |
| `15` (RESERVE_PCT, TAX_PCT) | percentuais default | 5724 |
| `80` (MAX_CAP, GOAL_6MO) | capacidades default | 5724 |
| `2000` | hireCost default | 5724 |
| `1.35` | priceLb default | 747, default `ldCfg` |
| `0.15`, `0.30`, `0.38`, `0.22` | preços sqft hardcoded | 834 (EST_SVCS), 836 (fallback no PDF) |

### Contagem: ~25 casos críticos (cores) + ~15 strings/URLs + ~20 números mágicos = **~60 itens**

### Como resolver
- Criar bloco `MS = {TOAST:3e3, DEBOUNCE:1500, SAVE_FLASH:800, PRINT_DELAY:600, WEEK_MS:7*86400000, DAY_MS:86400000, ...}` perto de `MO`.
- Mover `SUNNY_URL`/`LOGO` para o topo (já existe em 869 mas duplicado em 427 e 836). Garantir que `genEstPDF` use `co.address`/`co.phone` em vez dos hardcoded.
- Para cores, substituir strings literais por interpolação com `T.ac`, `T.bg`, etc. nos blocos que **estão dentro do `App`** (não nos blocos `FeedbackPublicPage` / `EmpRegisterPublicPage` que são públicos e não têm acesso a `T`).

### Risco: **médio** para cores (afeta UX em dark/light), **baixo** para constantes numéricas, **médio** para a fix do endereço (é bug de exibição mas não quebra dados)

### Tempo estimado: 4-6h para cores (precisa testar dark E light), 30min para constantes, 15min para o address fix.

---

## b) Código duplicado

### Achados

**b.1 Padrão `try{const r=await FB.get("X");if(r&&r.value)setY(JSON.parse(r.value));}catch{}`**

41 ocorrências (`grep -c` confirmou). Concentradas no `useEffect` inicial linhas **720-775** (cerca de 30 instâncias contíguas, exatamente o mesmo padrão variando só a chave e o setter). Exemplos:
- 720: `sp3 → setSP`
- 721: `si3 → setSI`
- 724: `fn3 → setFin`
- 726-729, 732-736, 745-763, 774-775

**b.2 Geração de iniciais de cliente (`prefix`/`code`)**

Lógica `(parts[0][0]+parts[1][0]).toUpperCase()` aparece em pelo menos 3 lugares:
- 722 (dentro do migration do `db3`, inline): `const ini=(client||"XX").trim().split(/\s+/).length>=2?(client.trim().split(/\s+/)[0][0]+client.trim().split(/\s+/)[1][0]).toUpperCase():client.slice(0,2).toUpperCase();`
- 797: `getInitials` (helper já existente, mas a inline do 722 **não usa o helper**)
- 801: `genCodes` reutiliza `getInitials` corretamente

A inline da 722 deveria chamar `getInitials(client)` — mas `getInitials` está definida na linha 797, **depois** do useEffect inicial que a usa. Acidente histórico.

**b.3 PDF/canvas drawing**

Funções de geração de PDF separadas com lógica de header/footer/style sobreposta:
- `gImg` (437-474) — canvas-based payslip PNG
- `genMonthlyReport` (871) — HTML report
- `genTag` (804-809) — QR tags
- `genEstPDF` (836-837) — estimate
- `genSopPDF` (843) — SOP
- `pdfHeader` (840), `pdfFooter` (841), `pdfBack` (842) — helpers só usados parcialmente

Cada PDF redeclara seu próprio CSS quase-idêntico (`*{margin:0;padding:0;box-sizing:border-box}body{font-family:Arial,sans-serif;...}`), define o logo inline, define footer manualmente. `pdfHeader`/`pdfFooter` existem mas só são usados em `genMonthlyReport`; `genEstPDF` e `genSopPDF` reimplementam tudo do zero.

**b.4 Modal outer (overlay + close button)**

25 instâncias do padrão `position:"fixed",top:0,left:0,right:0,bottom:0,background:"rgba(0,0,0,...)"` para overlay de modal. Cada uma reimplementa: backdrop, centralização, animation `fadeIn`, click-outside-close, botão `✕` no canto. Há um componente `<ModalC/>` na linha 1415 mas **não substituiu** essas instâncias.

**b.5 `parts.split(...).split(...)` para data**

Pattern `new Date(dateStr+"T12:00")` aparece **dezenas** de vezes — poderia ser helper `toDate(s)`.

**b.6 Recuperação de prop por nome+cliente**

`db.find(p=>norm(p.n)===norm(...)&&norm(p.c)===norm(...))` — pattern que `findProp` (linha 796) já encapsula, mas a maioria do código não usa.

**b.7 Save flush wrappers**

Linhas 822-829 declaram 11 wrappers quase-idênticos: `svLdH`, `svLdCfg`, `svLdCatalog`, `svLdLog2`, `svLdManual`, `svLdLots`, `svLdLead`, `svDc`, `svManual`, `svEsc`, `svEst`. Cada um chama `set<State>(d);await sv(<key>,d)`. Poderia ser fábrica: `makeSv = (key, setter) => (d) => { setter(d); return sv(key, d); }`.

**b.8 Estilos `inp`/`lb`/`sel` duplicados**

`inp`, `lb`, `sel` definidos em 1361-1364 (escopo `App`). Mas `InspForm` redefine `inp2`/`lb2`/`sel2` (508-510) e `EmpRegisterPublicPage` redefine `inpS`/`lbS` (358-359). Os valores são **muito** parecidos mas não idênticos (diferenças minúsculas de padding/border-color).

### Contagem: **~7 famílias de duplicação, totalizando ~120+ instâncias**

### Como resolver
- Criar helper `loadKey(key, setter, transform=JSON.parse)` para reduzir as 30 linhas do useEffect a um loop sobre `[["sp3",setSP],["si3",setSI],...]`.
- Criar `makeSv(key,setter)` para os wrappers.
- Substituir overlay-modal inline pelo `<ModalC>` que já existe (ou estender o `<ModalC>` para receber `children`).
- Extrair `pdfBoilerplate` (CSS + header + footer comuns) para fora.

### Risco: **médio** — o loader inicial é frágil (ordem importa porque alguns setters dependem de leitura encadeada); o resto é baixo.

### Tempo estimado: 3-5h.

---

## c) Variáveis com nomes curtos demais

### Tabela de renomeações sugeridas

**Seguras de renomear** (local a `App`, sem persistência de schema):

| Atual | Sugerido | Onde | Linhas afetadas (declaração) | Risco |
| --- | --- | --- | --- | --- |
| `vw` / `setVw` | `view` / `setView` | navegação principal | 650 | baixo (~80 referências) |
| `T` | `theme` | paleta ativa | 649 | médio (centenas de refs `T.ac`, `T.bg`) |
| `co` / `setCo` | `company` / `setCompany` | dados da empresa | 651 | baixo |
| `db` / `setDb` | `properties` / `setProperties` | array de casas | 651 | médio (~100 refs) |
| `em` / `setEm` | `employees` / `setEmployees` | array de funcionários | 651 | baixo |
| `cls` | `clients` | derivado de `db` | 795 | baixo |
| `sP` / `setSP` | `signedPayrolls` / `setSignedPayrolls` | folhas salvas | 654 | baixo |
| `sI` / `setSI` | `signedInvoices` | invoices salvas | 654 | baixo |
| `vP` / `vI` | `viewingPayroll` / `viewingInvoice` | item em preview | 655 | baixo |
| `pI` / `setPI` | `payrollItems` | itens em edição | 693 | baixo |
| `iI` / `setII` | `invoiceItems` | idem | 694 | baixo |
| `pTot` / `pInv` | `payrollTotal` / `payrollInvoiceTotal` | totais | 814 | baixo |
| `iTot` | `invoiceTotal` | total fatura | (várias) | baixo |
| `fl` | `flashToast` | toast helper | 794 | médio (~150 refs) |
| `bt` | `btnStyle` | helper de botão | 1364 | médio (~200 refs) |
| `fm` | `formatMoney` | formatter | 419 | médio (~50 refs) |
| `fmD` | `formatDate` | date | 420 | médio (~40 refs) |
| `lb` / `inp` / `sel` | `labelStyle`/`inputStyle`/`selectStyle` | estilos | 1361-1363 | médio (~300 refs) |
| `D` / `IE` / `DC` | `DEFAULT_PROPERTIES` / `DEFAULT_EMPLOYEES` / `DEFAULT_COMPANY` | seeds | 407, 414, 415 | baixo |
| `MO` | `MONTHS_PT` | meses | 418 | baixo |
| `EC` / `ESC` / `CCTR` / `DSC` | `FIN_INCOME_CATS` / `FIN_EXPENSE_CATS` / `COST_CENTERS` / `INVOICE_DESCRIPTIONS` | listas | 426, 430, 431, 432 | baixo |
| `LT` / `ILINEN` / `IPROD` | `LINEN_TYPES` / `INITIAL_LINEN` / `INITIAL_PRODUCTS` | inventário | 433, 434, 435 | baixo |
| `fE` / `fM` / `fIC` / `fIM` | `filterEmp` / `filterMonth` / `filterInvClient` / `filterInvMonth` | filtros UI | 656, 692 | baixo |
| `fF` | `financeForm` | form state | 698 | baixo |
| `fV` / `fMo` / `fWk` | `financeView` / `financeMonth` / `financeWeek` | filtros | 697 | baixo |
| `fPS` | `filterPayrollStatus` | filtro de status | 656 | baixo |
| `pM` | `payrollMonth` | mês selecionado | 693 | baixo |
| `sE` | `selectedEmployee` | empregado em edição | 693 | baixo |
| `wkFrom` / `wkTo` / `wk` | `weekStart` / `weekEnd` / `weekLabel` | período | 693, 816 | baixo |
| `iN` / `iD` / `iC` | `invoiceNumber` / `invoiceDate` / `invoiceClient` | invoice em edição | 694 | baixo |
| `ldHouses` etc. | já razoáveis | — | — | — |
| `lF` / `lpP` / `lpF` | `lossForm`/... | inventário | 706 | baixo |
| `pF` | `productForm` | inventário | 703 | baixo |
| `iTab` / `setITab` | `inventoryTab` | sub-tab estoque | 703 | baixo |

**CUIDADO — não renomear (são keys do Firestore ou usadas em strings):**

- Chaves de docs no Firestore (`"db3"`, `"em3"`, `"sp3"`, `"si3"`, `"fn3"`, `"ag3"`, `"ep3"`, `"insp3"`, `"ld_h2"`, `"ld_lots"`, `"fb_campaigns"`, `"users"`, `"co3"`, `"pr3"`, `"ln3"`, `"lb3"`, `"cc3"`, `"cfc3"`, `"cu3"`, `"ls3"`, `"tk3"`, `"man3"`, `"esc3"`, `"dc3"`, `"sops3"`, `"est3"`, `"est_cnt"`, `"ld_cfg2"`, `"ld_log2"`, `"ld_man2"`, `"ld_lead2"`, `"ld_cat"`, `"cdsc"`, `"emp_invites"`, `"emp_roles"`, `"ical_sync_log"`, `"auto_triggers"`, `"fb_learned"`, `"_trash"`, `"_autoBackup"`, `"_lastBackup"`)
- Campos dos objetos persistidos: `p.n` (nome), `p.c` (cliente), `p.a` (endereço), `p.ip` (invoice price), `p.ep` (employee pay), `p.cd` (código), `e.amt`, `e.cat`, `e.desc`, `e.type`, `e.date`, `e.month`, `e.week`, `e.ref`, `e.centro`. **Esses NÃO podem mudar sem migração.**
- `T.ac`, `T.bg`, `T.cd`, `T.c2`, `T.tx`, `T.dm`, `T.bd`, `T.rd`, `T.gr`, `T.or`, `T.pu`, `T.bl`, `T.aD` — se renomear `T → theme` precisa renomear **todas** as ~600 referências.
- `vw` aparece em URL parsing? **Não** — só estado interno, então é seguro, mas é massivo (~80 refs).

### Risco geral: **médio**. Renomeações são puramente locais (sem efeito de schema), mas a quantidade de refs e a falta de IDE/linter significa que um erro de digitação **só aparece em runtime** (página em branco com `<pre>` vermelho).

### Tempo estimado: 6-8h se feito de uma vez. Recomendável fatiar: módulo a módulo (payroll, invoice, finance, etc.), 1-2h cada.

---

## d) Seções sem comentários

| Linha | View / bloco | Header sugerido |
| --- | --- | --- |
| 407 | Seed data (`D`, `IE`, `DC`, etc.) | `// =============== SEEDS & DEFAULT DATA ===============` |
| 416 | Paletas `TD`/`TL` | `// =============== TEMA & PALETAS ===============` |
| 418-435 | Formatters + constantes globais | `// =============== HELPERS GLOBAIS (fm, fmD, norm, etc.) ===============` |
| 437 | `gImg` (canvas payslip PNG) | `// =============== CANVAS RECEIPT (gImg) ===============` |
| 476-481 | `TI`, `NI`, `BarChart`, `DonutChart`, `MiniLine` | `// =============== COMPONENTES BASE (inputs, charts) ===============` |
| 482-491 | `INSP_CHECKLIST` + helpers de score | `// =============== INSPEÇÃO: CHECKLIST & SCORES ===============` |
| 494 | `InspForm` | `// =============== INSPECT FORM (componente) ===============` |
| 645 | `ROLES` | `// =============== ROLES & PERMISSÕES ===============` |
| 646 | `App` body começa | `// =============== APP PRINCIPAL ===============` |
| 719 | `useEffect` inicial gigante (boot) | `// =============== BOOTSTRAP: load all Firestore keys ===============` |
| 782-784 | `React.useEffect` sync codes / scan | `// =============== EFFECTS: code sync + scan param ===============` |
| 790-794 | `CRITICAL_KEYS`, `sv`, `dsv`, `fl` | `// =============== PERSISTÊNCIA: sv / dsv / fl ===============` |
| 796-803 | `findProp`, `getInitials`, `genCodes`, `genTag` | `// =============== CÓDIGOS DE CASA & TAGS ===============` |
| 812-818 | Payroll handlers (`addPi`, `updPi`, `savP`, `updatePStatus`) | `// =============== PAYROLL HANDLERS ===============` |
| 820-832 | Laundry helpers + save wrappers | `// =============== LAUNDRY HELPERS & SV WRAPPERS ===============` |
| 833-869 | Estimates, SOPs, PDF helpers, scan | `// =============== ESTIMATES & PDF GENERATORS ===============` |
| 870-871 | `genQRDataURL`, `genMonthlyReport` | `// =============== QR CODE & MONTHLY REPORT ===============` |
| 872-928 | `keepScroll`, `safeDel`, `uAg`, `addAg`, agenda helpers, `inspectFromAgenda` | `// =============== UI HELPERS & AGENDA HANDLERS ===============` |
| 1129 | já tem (`AUTO-FATURAMENTO & FOLHA`) | — |
| 1370-1373 | `allTabs`, `tabs`, sync effects | `// =============== NAVEGAÇÃO: TABS & BADGES ===============` |
| 1414 | bloco de styles inline | `// =============== STYLE TAG (scrollbar, transitions) ===============` |
| 5043 | `vw==="escala"` | `// =============== ESCALA (calendário 5x1) ===============` |
| 5267 | `vw==="feedback"` | já tem marker `{/* FEEDBACK */}` na 5266, ok |
| 5724 | `vw==="strategy"` | **falta marker** — sugerido: `{/* STRATEGY */}` |
| 6627 | `vw==="database"` | já tem marker, ok |
| 6686 | `vw==="icalsync"` | **falta marker** — sugerido: `{/* ICALSYNC */}` |
| 6840 | bottom nav | `{/* BOTTOM NAV */}` |

### Risco: **baixo** (comentários não mudam comportamento)

### Tempo estimado: 20-30min.

---

## e) Inconsistências de formatação

### Achados

**e.1 Indentação**
- Predominantemente **espaços** (não há tabs). 2.743 linhas começam com 2+ espaços; 2.198 com 4+.
- Maior parte do código `App` é **one-liner sem indentação** (linhas de 500+ chars com tudo numa linha).
- Alguns blocos novos (`FeedbackPublicPage`, `EmpRegisterPublicPage`, `parseICalText`, `getInvoicesDueToday`, `genAutoLot`) **usam indentação multilinha** (2 espaços). Isso cria estilo inconsistente entre o "core App" (denso) e os "blocos novos" (formatados).
- Linha 408-413: `D` array está quebrado em ~6 linhas com elementos agrupados por cliente — **OK** (legível).

**e.2 Espaços ao redor de operadores**
- Convenção: **sem espaços** (estilo `a===b`, `x+1`, `p=>p.n`). Consistente quase 100%.
- Exceções pontuais: linhas 408-413 às vezes têm `,` sem espaço, outras vezes vão com aspas em strings com espaços (esperado).
- **Sem inconsistência grave detectada.**

**e.3 Quebras de linha**
- Linhas excessivamente longas (>500 chars): muito comuns. Linhas que **se beneficiariam de quebra** (passos lógicos discretos):
  - Linha 28 (declaração de `FB` — get/set/del em uma única linha) — poderia ser 3 linhas.
  - Linha 407 — array `D` de 24 properties em uma linha (3000+ chars).
  - Linha 415 — `DC` em uma linha.
  - Linha 791 — `sv` com lógica de bloqueio crítica em uma linha.
  - Linha 836 — `genEstPDF` inteira em UMA linha (~5000 chars).
  - Linha 843 — `genSopPDF` inteira em UMA linha.
  - Linha 871 — `genMonthlyReport` inteira em UMA linha.

**e.4 Estilos inline JSX**
- Três padrões coexistem:
  - **Objeto literal direto:** `style={{padding:"8px",color:"#fff",...}}` — dominante.
  - **Função `bt(bg,cl,extra)`:** linha 1364. Usado para botões.
  - **Variáveis nomeadas:** `inp`, `lb`, `sel` (1361-1363) para inputs/labels/selects.
- **Inconsistência**: alguns inputs/labels usam `style={inp}` / `style={lb}`, outros redeclaram `style={{padding:"...",border:"1px solid "+T.bd,...}}` inline. Não há regra clara.
- `InspForm` (494) redefine **localmente** `inp2`, `lb2`, `sel2`, `bd2`, `cd2` (508-511) em vez de usar `inp`/`lb`/`sel` — porque o `InspForm` é declarado **antes** de `App`, então não tem acesso a `T` nem a `inp` (escopo).
- `EmpRegisterPublicPage` redefine `inpS`/`lbS` (358-359) pelo mesmo motivo.

**e.5 Vírgulas pendentes e ponto-e-vírgulas extras**
- Linha 692: `;const[fIC,setFIC]=useState("");...` — **`;` órfão no início da linha** após 21 linhas vazias. Sinal de que algo foi deletado.
- Linha 836: `const addr=[e.street,e.city,e.state,e.zip].filter(Boolean).join(", ");;` — **duplo `;;`**. Bug menor, não quebra mas é sujeira.

### Contagem: ~5 inconsistências sistemáticas + 2 anomalias pontuais (`;` órfão, `;;`).

### Como resolver
- Decidir convenção: o código existente em produção é one-liner-denso. **Não reformatar one-liners** — quebrá-los gera diff enorme e dificulta diff/blame futuros. Mas o `;` órfão (692) e `;;` (836) podem ser limpos.
- Para `InspForm` e `EmpRegisterPublicPage`: aceitar a duplicação de `inp2`/`inpS` ou extrair as funções para dentro de `App` (mas isso muda escopo).

### Risco: **baixo** (formatação não muda comportamento — mas reformatar one-liners gigantes pode introduzir bug se faltar `;` em alguma quebra).

### Tempo estimado: 30min para limpezas pontuais. NÃO reformatar one-liners.

---

## f) Dead code

| Símbolo | Tipo | Declarado em | Referências (uso real) | Pode remover? |
| --- | --- | --- | --- | --- |
| `nCl`, `setNCl` | useState | 695 | 0 | **Sim** (apenas declaração) |
| `nNm`, `setNNm` | useState | 695 | 0 | **Sim** |
| `nAd`, `setNAd` | useState | 695 | 0 | **Sim** |
| `nIp`, `setNIp` | useState | 695 | 0 | **Sim** |
| `nEp`, `setNEp` | useState | 695 | 0 | **Sim** |
| `eIdx`, `setEIdx` | useState | 696 | 0 | **Sim** |
| `eD`, `setED` | useState | 696 | 0 | **Sim** |
| `hfPay`, `setHfPay` | useState | 717 | 0 | **Sim** |
| `hfInv`, `setHfInv` | useState | 717 | 0 | **Sim** |
| `editCl`, `setEditCl` | useState | 717 | 0 | **Sim** |
| `navGrp`, `setNavGrp` | useState | 650 | 0 | **Sim** |
| `escEdit`, `setEscEdit` | useState | 705 | 0 | **Sim** |
| Linhas 671-691 (21 linhas vazias) | whitespace | — | — | **Sim** (limpar) |
| `;` órfão na linha 692 | sintaxe | 692 | — | **Sim** (mas testar — JS aceita) |
| `;;` na linha 836 | sintaxe | 836 | — | **Sim** |
| `qrcodejs` CDN | script tag | 6 | usado em `genQRDataURL` linha 870, `new QRCode(...)` | **Não** (é usado) |
| Bloco comentado de scratch (linhas 765-773) | whitespace | 765-773 | — | já são linhas em branco, não comentário antigo |

**Verificações negativas (NÃO dead, apesar de aparência):**

- `ss` / `setSs` (linha 652): `setSs` é chamado em 1634. Mantém.
- `dragIdx` / `setDragIdx` (656): usado em 1974, 2031. Mantém.
- `weekStartDate`: usado em 3128-3129. Mantém.
- `agQuick`, `agRows`: usados (14 refs total). Mantém.
- `editFinId` / `editFinData`: usados (2219-2236). Mantém.
- `inclSporadics`, `showSSN`, `quickEmpModal`: todos usados.

### Risco: **baixo**. Remover useState não usadas é seguro. O `;` órfão em 692 é cosmetic — JavaScript trata como statement vazio.

### Tempo estimado: 20min.

---

## Bugs detectados durante a análise (informativo, NÃO corrigir agora)

### Bug 1 — Endereço inconsistente em `genEstPDF`
- **Linha:** 836
- **Sintoma:** O PDF de estimate gerado por `genEstPDF` mostra hardcoded `2501 Orient Rd, Tampa, FL 33619` e phone `813-546-2577`. Mas o resto do sistema usa `co.address` (`DC.address = "3272 N Arlington Ave. - FL 33603"`). Cliente que receber um Estimate vê endereço diferente do Invoice/Payslip.
- **Severidade:** Média (estética/profissionalismo, não afeta dados).

### Bug 2 — `idMap` sem cleanup completo de duplicados em `ld_h2`
- **Linha:** 737-743 (migration de `ld_h2`).
- **Sintoma:** A migration usa `mergeMap` para deduplicar, mas se `idMap[h.id]=null` (linha 739) e o `idMap` é depois usado em 742 para mapear `houseId`, há casos onde `idMap[h.id]` é null mas é resolvido para `undefined`. Não rastreei o efeito final mas a lógica está intricada.
- **Severidade:** Baixa (provavelmente já estabilizou em produção).

### Bug 3 — `;;` em `genEstPDF`
- **Linha:** 836 (`const addr=...filter(Boolean).join(", ");;`).
- **Sintoma:** Statement nulo extra. JS aceita. Cosmético.

### Bug 4 — `getInitials` usado **antes** de ser declarado
- **Linhas:** 722 (linha de migração usa lógica inline equivalente), `getInitials` declarado em 797. Linha 784 (`React.useEffect` para `scanParam`) já chama `getInitials` numa closure que só executa depois do mount, então OK em runtime — mas fica esquisito de ler.
- **Severidade:** Nula em runtime, alta em legibilidade.

### Bug 5 — `beforeunload` faz `XMLHttpRequest` síncrono mas chama `sv` (async) na sequência
- **Linha:** 793.
- **Sintoma:** O código abre um XHR síncrono para `https://firestore.googleapis.com/` mas **não envia nada**, depois chama `sv(k,d)` que é async — `beforeunload` não espera promise. O pending save provavelmente **não chega** ao Firestore. O fallback `localStorage._pend_*` (792) é o que realmente salva, e é recuperado em 777-778 no próximo load. Mas o XHR ali é dead code.
- **Severidade:** Baixa (existe fallback).

### Bug 6 — `monthlyGoal` vs `meta`
- **Linha:** 1815 (`META=co.meta||50000`), 1841 (`const goal=co.monthlyGoal||0`).
- **Sintoma:** Duas chaves diferentes em `co` para a mesma ideia ("meta mensal de receita"). Provavelmente legado de duas iterações. O usuário pode ter configurado `meta` mas o código que usa `monthlyGoal` retorna `0`.
- **Severidade:** Média (dashboard mostra metas conflitantes).

### Bug 7 — `getMonthFromDate` vs `MO.indexOf(...)` 
- **Linhas:** 871 — usa `MO.indexOf(mo)` (recebe nome do mês em string) e compara com `getMonthFromDate(i.date)` que retorna nome do mês. OK, mas em outros lugares (e.g. 1247-1259) o código mistura indexação numérica (`getMonth()`) com string. Risco de bug de fuso horário se alguém adicionar lógica futura.

---

## Ordem de execução recomendada

1. **Dead code** (categoria f) — mais seguro: remove useState não usadas, limpa linhas vazias 671-691, remove `;` órfão 692, remove `;;` 836. Risco baixo, ganho imediato em clareza. Tempo: 20-30min.

2. **Headers de seção** (categoria d) — adiciona comentários. Risco zero, melhora navegação. Tempo: 30min.

3. **Constantes numéricas e URLs duplicadas** (parte a.3 e a.4) — extrair `MS` constants, unificar `LOGO`/`SUNNY_URL`, corrigir o address hardcoded do `genEstPDF` (resolve Bug 1). Risco baixo (são literais). Tempo: 1h.

4. **Save wrappers + loader loop** (parte b.1, b.7) — refatora o `useEffect` inicial para um loop sobre array de pares `[key, setter]`. Reduz ~30 linhas para ~5 linhas. Testar boot rigorosamente. Risco médio. Tempo: 1-2h.

5. **Cores hardcoded → tema `T`** (categoria a.1) — substitui literais por refs `T.ac/T.bg/...` dentro de `App`. **NÃO mexer** em `FeedbackPublicPage` nem `EmpRegisterPublicPage` (não têm `T`). **NÃO mexer** em PDFs (window.open documents — eles intencionalmente usam cores fixas porque não respeitam tema). Testar dark E light. Risco médio. Tempo: 3-4h.

6. **Renomeações de variáveis** (categoria c) — fazer **uma view por vez** (payroll → invoice → finance → ...). Testar cada uma. Risco médio-alto pela quantidade. Tempo: 1-2h por módulo.

7. **Modal/PDF extraction** (b.3, b.4) — risco alto: extrair componentes muda escopo. Adiar até ter testes.

---

## Avisos sobre risco em código de produção

- **Chaves do Firestore (`CRITICAL_KEYS = ["db3","em3","sp3","si3","fn3","ag3","ep3","insp3"]`)**: nunca renomear nem mudar o formato dos dados sem migração. O sistema **bloqueia** save de array vazio nessas chaves (linha 791), mas isso não protege contra schema corrupto.
- **Campos persistidos nos objetos** (`p.n`, `p.c`, `p.a`, `p.ip`, `p.ep`, `p.cd`, `e.amt`, `e.cat`, `e.desc`, `e.type`, `e.date`, `e.month`, `e.week`, `e.ref`, `e.centro`, `h.name`, `h.client`, `h.status`, `h.cd`, `h.weight`, `h.lotId`, etc.) — **NÃO podem mudar**. São o "schema" implícito do sistema.
- **Service Worker cache `sunny-v3`** (linha 6852): se você mudar `index.html` mas não bumpar para `sunny-v4`, usuários que já carregaram a versão antiga continuarão vendo a antiga. **Sempre bumpar quando shippar mudança visual ou de comportamento.**
- **Sem build, sem lint, sem teste**: qualquer erro de sintaxe quebra a **página inteira** com um `<pre>` vermelho. Não há early warning. Recomenda-se rodar `node --check` ou um Babel parse standalone localmente antes de commitar mudanças no `<script type="text/plain">` (mas note que Babel rebanca a sintaxe React via Babel-standalone em runtime — então erros JSX vão **só** no console). Vale tirar o arquivo do `app-jsx` para um JS standalone temporário antes de mudanças grandes.
- **Babel-standalone roda no cliente**: cada page load custa ~150-300ms transpilando os ~800KB de JSX. Isso é tolerado em prod, mas qualquer mudança que aumente o arquivo muito (e.g. expansão de one-liners para multilinha bonita) tem custo de TTI.
- **`FB.set` swallows errors silencioso** (linha 28: `catch(e){return null;}`). Saves falhos só aparecem se você checar `saveFlash`. Refatorações que dependam de `await sv()` devem **conferir o retorno**.
- **`auth.onAuthStateChanged` é a única coisa que gate o `App`**. Se `users` no Firestore for corrompido, role default vira `"admin"` (linha 403). Cuidado ao refatorar role parsing.
- **Mensagem do CLAUDE.md**: "Match the surrounding density when adding code in a section, don't refactor toward a different style." — esta auditoria **propõe quebrar essa regra** intencionalmente, mas só nas categorias onde o ganho de clareza supera o custo de diff. Renomear `vw → view` é o tipo de mudança que viola o estilo de propósito.

---

## Apêndice: lista completa de cores hex únicas (137)

```
#000  #0094B0  #0094B00A  #0094B012  #0094B015  #0094B022  #0094B033  #0094B044
#00B4D8  #00B4D808  #00B4D822  #00B4D844  #00B4D888
#0A0F1E  #0A0F1EEE  #0B0E14  #0D1B30  #0EA5E9
#10B981  #111  #111833  #15803D  #16A34A  #182040  #1A1A2E  #1A1A3E  #1A2545
#222  #22C55E  #25D366  #2E7D32  #333  #3B82F6  #444
#4ADE80  #4ADE800A  #4ADE8018  #4ADE8020  #4ADE8022  #4ADE8033
#555  #60A5FA  #60A5FA22  #666  #6B7280  #707A8A  #777  #795548
#8899AA  #888  #8B5CF6  #999
#A0AABC  #A16207  #A78BFA  #A78BFA0A  #A78BFA10  #A78BFA12  #A78BFA15  #A78BFA18  #A78BFA33
#A855F7  #AAA  #B91C1C  #BBB
#C0C0C0  #C0C0C022  #CCC  #CD7F32  #CD7F3222
#DAA520  #DCFCE7  #DDD
#E0E0E0  #E0F7FA  #E0F7FA22  #E2E5EA  #E5E5E5  #E5E7EB  #E879F9  #E8EBF0  #E8EEF0  #E8F5E9
#EA80FC  #EAB308  #EAF6FA  #EC4899  #EEE  #EEF2F4  #EF4444
#F0F0F0  #F0F2F5  #F0F4F6  #F0F4FF  #F0FDF4  #F0FDF9  #F3F4F6  #F59E0B
#F5F5F5  #F5F7FA  #F5F7FAEE  #F5FAFD  #F8F9FB  #F8FAFB  #F8FBFD  #F97316  #F9F9F9
#FAFBFC  #FEE2E2  #FEF3C7
#FF6B6B  #FF6B6B08  #FF6B6B0A  #FF6B6B10  #FF6B6B12  #FF6B6B15  #FF6B6B18  #FF6B6B20  #FF6B6B33  #FF6B6B44
#FF8A65  #FF8A650A  #FF8A6515  #FF8A6533  #FF8A80  #FFB547  #FFB54720  #FFB54722
#FFD700  #FFD70008  #FFD70015  #FFD70018  #FFD70022  #FFD70033
#FFF  #FFF8E1  #FFFFFF  #FFFFFFCC
```

Variações `+suffix` (8 chars com alpha) podem ser reduzidas usando `T.color + "XX"` em vez de novos literais.
