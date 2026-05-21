# PLAN — Campos checkOut/guests/pets + Priorização inteligente

## Sumário

Adicionar 3 campos opcionais ao formulário de turnover (`agForm`) e refazer a lógica de priorização da tela "🎯 Hoje" (`inspView==="today"`) para usar 3 zonas baseadas em dias até check-in. Atualizar tela "📅 Semana" para mostrar metadados novos. Bump SW no final.

**Compatibilidade:** schema é só adição. Turnovers existentes (sem os campos novos) continuam funcionando — todos os reads usam `ev.field || default`.

**Boa notícia:** `checkOut` **já existe no schema** — é populado por iCal sync (linhas 1013, 1029-1030) e exibido no `weekDetailModal` (linha 3227). Só falta entrada manual no formulário. `guests` e `pets` são realmente novos.

## Pontos exatos de modificação

| Onde | Linha | O que muda | Commit |
|---|---|---|---|
| `agForm` modal — bloco "PRÓXIMO HÓSPEDE" | 4548-4556 | Adicionar 3 inputs novos | B.1 |
| Lógica de priorização (`inspView==="today"`) | 3259-3303 | Reescrever filtros + zonas | B.2 |
| Renderização das zonas (`inspView==="today"`) | 3338-3385 | Substituir 4 cards por 3 zonas | B.2 |
| `renderCard` em "Hoje" | 3307-3336 | Adicionar 🐾/👥X/📅in nos cards | B.2 |
| `renderHouseCard` em "Semana" | 3134-3150 | Adicionar 🐾/👥X/📅in nos cards | B.3 |
| `weekDetailModal` info grid | 3223-3230 | Adicionar guests e pets ao grid | B.3 |
| Service worker cache | 6830 | `sunny-v4` → `sunny-v5` | B.4 |

## COMMIT B.1 — Campos no formulário

### ANTES (linhas 4548-4556, dentro de `(agForm.type==="Turnover"||!agForm.type)&&...`)

```jsx
<div style={{padding:"10px 12px",background:T.bl+"06",borderRadius:8,border:"1px dashed "+T.bl+"30",marginBottom:10}}>
<div style={{fontSize:10,color:T.bl,fontWeight:700,marginBottom:6}}>📅 PRÓXIMO HÓSPEDE {agForm.icalUid&&<span style={{fontSize:9,color:T.gr,marginLeft:4}}>🤖 vindo do iCal</span>}</div>
<div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8}}>
<div><label style={lb}>Check-in</label><DI value={agForm.checkIn||""} onChange={e=>setAgForm({...agForm,checkIn:e.target.value})} style={{...inp,width:"100%"}}/></div>
<div><label style={lb}>Hora Check-in</label><TI value={agForm.checkInTime||""} onChange={e=>setAgForm({...agForm,checkInTime:e.target.value})} style={{...inp,width:"100%"}} placeholder="16:00"/></div>
</div>
{agForm.checkIn===agForm.date&&agForm.date&&<div ...>🔥 SAME DAY ...</div>}
{agForm.stayNights&&<div ...>🛏️ Estadia: ...</div>}
</div>
```

### DEPOIS

```jsx
<div style={{padding:"10px 12px",background:T.bl+"06",borderRadius:8,border:"1px dashed "+T.bl+"30",marginBottom:10}}>
<div style={{fontSize:10,color:T.bl,fontWeight:700,marginBottom:6}}>📅 PRÓXIMO HÓSPEDE {agForm.icalUid&&<span style={{fontSize:9,color:T.gr,marginLeft:4}}>🤖 vindo do iCal</span>}</div>
<div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:8}}>
<div><label style={lb}>Check-out (hóspede anterior)</label><DI value={agForm.checkOut||""} onChange={e=>setAgForm({...agForm,checkOut:e.target.value})} style={{...inp,width:"100%"}}/></div>
<div><label style={lb}>Check-in (próximo hóspede)</label><DI value={agForm.checkIn||""} onChange={e=>setAgForm({...agForm,checkIn:e.target.value})} style={{...inp,width:"100%"}}/></div>
<div><label style={lb}>Hora Check-in</label><TI value={agForm.checkInTime||""} onChange={e=>setAgForm({...agForm,checkInTime:e.target.value})} style={{...inp,width:"100%"}} placeholder="16:00"/></div>
<div><label style={lb}>Hóspedes</label><NI value={agForm.guests||""} onChange={e=>setAgForm({...agForm,guests:e.target.value===""?undefined:Math.max(0,parseInt(e.target.value)||0)})} style={{...inp,width:"100%"}}/></div>
<div style={{gridColumn:"1/3",display:"flex",alignItems:"center",gap:8,padding:"6px 0"}}>
<label style={{display:"flex",alignItems:"center",gap:6,cursor:"pointer",fontSize:11,color:T.tx,fontWeight:600}}>
<input type="checkbox" checked={!!agForm.pets} onChange={e=>setAgForm({...agForm,pets:e.target.checked})} style={{width:16,height:16,cursor:"pointer",accentColor:T.ac}}/>
🐾 Próximo check-in tem pets
</label>
</div>
</div>
{agForm.checkIn===agForm.date&&agForm.date&&<div ...>🔥 SAME DAY ...</div>}
{agForm.stayNights&&<div ...>🛏️ Estadia: ...</div>}
</div>
```

**Notas:**
- Reusa componentes existentes `DI` (date), `TI` (text), `NI` (number).
- Grid mantém 2 colunas, fica em 3 linhas + 1 linha de checkbox full-width.
- `guests` sanitizado para inteiro ≥ 0 ou `undefined`; nunca string nem negativo.
- `pets` é boolean limpo (`!!agForm.pets` no checked).
- Layout responsivo: o `gridTemplateColumns:"1fr 1fr"` existente continua válido; em mobile vira coluna única via media query global.

**Validação:** o handler de save no botão "💾 Salvar" (linha 4567) não precisa mudar — `setAgForm({...agForm, ...})` já carrega os campos novos pra `uAg`.

## COMMIT B.2 — Priorização inteligente em "🎯 Hoje"

### ANTES (lógica resumida das linhas 3260-3303)

- Filtra eventos de hoje, amanhã, depois de amanhã (3 dias fixos)
- Enriquece com `isSameDay`, `noCleaner`, `alreadyInspected`, prioridade numérica
- Renderiza 4 seções: Top 10 prioritárias + Hoje + Amanhã + Depois de amanhã

### DEPOIS (nova lógica)

```js
{inspView==="today"&&(()=>{
  const today=getLocalDate();
  // Janela: turnovers nos próximos 14 dias (cobre as 3 zonas)
  const horizon=getLocalDateOffset(14);
  const allEvents=agEvents.filter(ev=>ev.date&&ev.date>=today&&ev.date<=horizon&&ev.house&&ev.house!=="—"&&(ev.type==="Turnover"||!ev.type||ev.type==="Deep Clean"));
  
  // Helpers
  const daysBetween=(a,b)=>Math.round((new Date(b+"T12:00")-new Date(a+"T12:00"))/86400000);
  
  // Enriquecer
  const enrich=(ev)=>{
    const noCleaner=!ev.employees||ev.employees.length===0;
    const alreadyInspected=inspections.some(i=>i.house===ev.house&&i.date===ev.date);
    const lastInsp=inspections.filter(i=>i.house===ev.house).sort((a,b)=>(b.date||"").localeCompare(a.date||""))[0];
    const houseObj=db.find(p=>p.n===ev.house);
    const status=ev.cleanStatus||"pending";
    const daysToCheckIn=ev.checkIn?daysBetween(today,ev.checkIn):null;
    const daysToTurnover=daysBetween(today,ev.date);
    // Dias desde última limpeza
    const daysSinceLastClean=lastInsp&&lastInsp.date?daysBetween(lastInsp.date,today):null;
    // Zona: urgência baseada em check-in
    let zone;
    if(daysToCheckIn!==null&&daysToCheckIn<=1)zone="urgente";
    else if(daysToCheckIn!==null&&daysToCheckIn>=2&&daysToCheckIn<=4)zone="recomendado";
    else zone="adiantar"; // checkIn >=5 OU sem checkIn
    return {...ev,noCleaner,alreadyInspected,lastInsp,houseObj,status,daysToCheckIn,daysToTurnover,daysSinceLastClean,zone};
  };
  
  const enriched=allEvents.map(enrich);
  const zUrgente=enriched.filter(e=>e.zone==="urgente").sort((a,b)=>(a.daysToCheckIn??99)-(b.daysToCheckIn??99));
  const zRecom=enriched.filter(e=>e.zone==="recomendado").sort((a,b)=>(a.daysToCheckIn??99)-(b.daysToCheckIn??99));
  const zAdiantar=enriched.filter(e=>e.zone==="adiantar").sort((a,b)=>(a.daysToTurnover||0)-(b.daysToTurnover||0));
  
  // KPIs ajustados pra nova realidade
  const kpiUrgente=zUrgente.length;
  const kpiRecom=zRecom.length;
  const kpiAdiantar=zAdiantar.length;
  const kpiNoCleaner=enriched.filter(e=>e.noCleaner&&e.zone!=="adiantar").length;
  const kpiPets=enriched.filter(e=>e.pets&&e.zone!=="adiantar").length;
  const kpiInsp=inspections.filter(i=>i.date===today).length;
  
  // (renderCard ajustado pra mostrar guests/pets/daysToCheckIn/daysSinceLastClean)
  ...
})()}
```

### Cards: novos metadados a mostrar

Cada card adicionalmente exibe (dentro do bloco já existente após `<div style={{fontSize:14,fontWeight:800...}}>{ev.house}</div>`):

```jsx
<div style={{fontSize:10,color:T.dm,marginTop:3,display:"flex",gap:8,flexWrap:"wrap"}}>
  {ev.daysToCheckIn!==null&&<span>📅in {ev.daysToCheckIn===0?"hoje":ev.daysToCheckIn===1?"amanhã":ev.daysToCheckIn+"d"}</span>}
  {ev.guests&&<span>👥 {ev.guests}</span>}
  {ev.pets&&<span style={{color:T.or}}>🐾 pets</span>}
  {ev.daysSinceLastClean!==null&&<span>🧹 há {ev.daysSinceLastClean}d</span>}
  {ev.employees&&ev.employees.length>0&&<span>👤 {ev.employees.join(", ")}</span>}
</div>
```

(Substitui a linha existente `{ev.employees&&...}` para consolidar metadados.)

### Renderização das zonas

Substitui as 4 seções atuais (Top 10 + Hoje + Amanhã + Depois) por 3 zonas:

```jsx
return <>
  {/* Header + KPIs (ajustados) */}
  ...
  
  {/* 🔴 URGENTE */}
  {zUrgente.length>0&&<Card style={{padding:14,borderLeft:"4px solid "+T.rd}}>
    <div style={{fontSize:13,fontWeight:800,marginBottom:4,color:T.rd}}>🔴 URGENTE — Check-in ≤ 1 dia ({zUrgente.length})</div>
    <div style={{fontSize:10,color:T.dm,marginBottom:10}}>Same-day ou check-in amanhã. Não dá pra adiar.</div>
    {zUrgente.map((ev,idx)=>renderCard(ev,idx))}
  </Card>}
  
  {/* 🟡 RECOMENDADO */}
  {zRecom.length>0&&<Card style={{padding:14,borderLeft:"4px solid "+T.or}}>
    <div style={{fontSize:13,fontWeight:800,marginBottom:4,color:T.or}}>🟡 RECOMENDADO — Check-in em 2-4 dias ({zRecom.length})</div>
    <div style={{fontSize:10,color:T.dm,marginBottom:10}}>Janela ideal pra limpar. Priorize sem deixar pro último momento.</div>
    {zRecom.map((ev,idx)=>renderCard(ev,idx))}
  </Card>}
  
  {/* 🟢 PODE ADIANTAR */}
  {zAdiantar.length>0&&<Card style={{padding:14,borderLeft:"4px solid "+T.gr}}>
    <div style={{fontSize:13,fontWeight:800,marginBottom:4,color:T.gr}}>🟢 PODE ADIANTAR — Check-in ≥ 5 dias ou sem data ({zAdiantar.length})</div>
    <div style={{fontSize:10,color:T.dm,marginBottom:10}}>Sem pressão de tempo. Use pra adiantar se sobrar capacidade.</div>
    {zAdiantar.map((ev,idx)=>renderCard(ev,idx))}
  </Card>}
  
  {/* Empty state */}
  {zUrgente.length===0&&zRecom.length===0&&zAdiantar.length===0&&<Card style={{textAlign:"center",padding:40}}>
    <div style={{fontSize:48,marginBottom:14}}>🌴</div>
    <div style={{fontSize:14,fontWeight:700,marginBottom:6}}>Nenhum turnover nos próximos 14 dias</div>
    <div style={{fontSize:11,color:T.dm}}>Verifique se a agenda está atualizada</div>
  </Card>}
</>;
```

### KPI bar — ajuste

Substitui os 6 KPIs atuais (Top 10 / Pendentes / Feitas / Same-day / Sem cleaner / Bloqueadas) por 6 novos:

```jsx
<div style={{display:"grid",gridTemplateColumns:"repeat(auto-fill,minmax(120px,1fr))",gap:8,marginBottom:12}}>
  <div ...><div>🔴 URGENTE</div><div>{kpiUrgente}</div></div>
  <div ...><div>🟡 RECOMENDADO</div><div>{kpiRecom}</div></div>
  <div ...><div>🟢 PODE ADIANTAR</div><div>{kpiAdiantar}</div></div>
  <div ...><div>Sem cleaner</div><div>{kpiNoCleaner}</div></div>
  <div ...><div>🐾 Com pets</div><div>{kpiPets}</div></div>
  <div ...><div>Inspecionadas hoje</div><div>{kpiInsp}</div></div>
</div>
```

## COMMIT B.3 — Tela "📅 Semana"

### renderHouseCard (linha 3134-3150)

Adicionar badges compactos antes do bloco `<div style={{display:"flex",gap:3,flexWrap:"wrap"...}}>`. Dentro desse bloco que já existe, antes dos badges de status, adicionar:

```jsx
{ev.pets&&<span style={{fontSize:9,color:T.or}}>🐾</span>}
{ev.guests&&<span style={{fontSize:8,color:T.dm,fontWeight:700}}>👥{ev.guests}</span>}
{ev.checkIn&&<span style={{fontSize:8,color:T.bl,fontWeight:700}}>📅in{Math.max(0,Math.round((new Date(ev.checkIn+"T12:00")-new Date(ev.date+"T12:00"))/86400000))}d</span>}
```

### weekDetailModal info grid (linha 3223-3230)

Adicionar 2 entradas extras no grid:

```jsx
{ev.guests&&<div><div style={{fontSize:9,color:T.dm,fontWeight:700,textTransform:"uppercase"}}>Hóspedes</div><div style={{fontSize:13,fontWeight:700,marginTop:2}}>👥 {ev.guests}</div></div>}
{ev.pets&&<div><div style={{fontSize:9,color:T.or,fontWeight:700,textTransform:"uppercase"}}>Pets</div><div style={{fontSize:13,fontWeight:700,marginTop:2,color:T.or}}>🐾 Sim</div></div>}
```

Dentro do grid existente `<div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>` (linha 3224).

## COMMIT B.4 — Bump service worker

Linha 6830 — única mudança no string template do SW:

```diff
-var CACHE='sunny-v4';
+var CACHE='sunny-v5';
```

**Por que separar do B.1-B.3:** o bump força refresh em todos os clientes ativos. Quer fazer só quando tudo testado, e quando você confirmar que a importação no Chrome terminou.

## Funções afetadas (resumo)

| Função | Linha | Mudança | Risco |
|---|---|---|---|
| `agForm` render | 4540-4571 | +3 inputs no bloco PRÓXIMO HÓSPEDE | Baixo — campos opcionais |
| Save handler `agForm` (botão Salvar) | 4567 | Nenhuma — usa spread do agForm | Baixo |
| `inspView==="today"` IIFE | 3259-3386 | Refactor completo da lógica + render | Médio — mas isolado |
| `renderCard` (Today) | 3307-3336 | +1 linha de metadados | Baixo |
| `renderHouseCard` (Week) | 3134-3150 | +3 badges condicionais | Baixo |
| `weekDetailModal` info grid | 3223-3230 | +2 entradas condicionais | Baixo |
| iCal sync handlers (linhas 1013, 1029) | — | Nenhuma | Nenhum — já gravam checkOut |
| AI context builder (~4948) | — | Nenhuma | Nenhum — usa Object scan |
| Service Worker | 6830 | Cache bump | Forçará refresh global |

## Risco de quebrar turnovers existentes

**Baixo.** Resumo das proteções:

- Todos os reads usam `ev.field || default` ou cheques `&&` antes de exibir
- Turnover sem `checkIn` → cai em zona "PODE ADIANTAR" (correto: sem pressão)
- Turnover sem `guests` → badge `👥X` não aparece
- Turnover sem `pets` → badge `🐾` não aparece
- iCal events com `checkOut` já preenchido → input do form mostra esse valor, pode reeditar
- Schema do Firestore é livre — não há validação no `FB.set` que rejeite campos novos

## Plano de validação (testes manuais no browser)

### Após B.1 (campos no formulário)
1. Abrir um turnover existente (sem campos novos) → form abre normal, checkOut/guests/pets vazios
2. Preencher os 3 campos → salvar → reabrir → valores persistiram
3. Editar turnover do iCal (que tem `checkOut` preenchido) → input mostra valor correto, pode editar
4. Mobile width: form responsivo, grid quebra em 1 coluna
5. Validação numérica: tentar digitar "abc" no guests → input rejeita ou sanitiza para 0
6. Sem internet (DevTools offline) → toast "❌ Sem conexão..." aparece (porque `uAg` usa `sv` antigo silencioso — toast de sucesso ainda virá, mas a auditoria já anotou isso para 5.3)

### Após B.2 (priorização inteligente)
1. Abrir tela "🎯 Hoje" → ver 3 zonas com cores e contagens corretas
2. Criar turnover com checkIn = hoje → cai em URGENTE
3. Criar turnover com checkIn = +3 dias → cai em RECOMENDADO
4. Criar turnover sem checkIn → cai em PODE ADIANTAR
5. Cards mostram metadados novos: 📅in Xd, 👥, 🐾, 🧹 há Yd
6. KPIs batem com as zonas
7. Empty state aparece quando nada nos 14 dias

### Após B.3 (semana)
1. Tela "📅 Semana" → badges 🐾/👥/📅in aparecem nos cards quando aplicável
2. Clicar num card → modal mostra grid com guests/pets adicionados
3. Card de turnover sem campos novos → continua igual, sem badges extras

### Após B.4 (SW bump)
1. Reload em janela anônima → service worker antigo descartado, novo cache `sunny-v5` ativo
2. Em janela com sessão aberta → após 1 reload, novo shell carregado

## Estimativa de commits

4 commits conforme spec do usuário:

- **B.1** — ~10 linhas tocadas (modal form). Tempo: 15min.
- **B.2** — ~80 linhas tocadas (refactor de `inspView==="today"`). Tempo: 30-40min.
- **B.3** — ~10 linhas tocadas (renderHouseCard + modal grid). Tempo: 10min.
- **B.4** — 1 caractere alterado (cache bump). Tempo: 2min.

Total: ~100 linhas tocadas no `index.html`. Validação no browser entre cada commit pode levar mais que o tempo de código.

## O que NÃO vou fazer (restrições do usuário)

- Não toco em invoices, folha, financeiro
- Não migra outros handlers para `safe/svT` (isso é a próxima sessão de error handling)
- Não refatora código que vejo "podia melhorar"
- Não renomeia campos existentes (`date`, `checkIn`, etc. ficam como estão)
- Não mexe na lógica do iCal sync — só verifica que continua compatível
- Não adiciona dependências novas (continua tudo em runtime via Babel)

## Pergunta

Aprova esse plano? Se sim, prossigo direto para **COMMIT B.1** (campos no formulário). Espero seu OK explícito.
