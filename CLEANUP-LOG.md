# CLEANUP-LOG.md — Pós-redesign sidebar/header

**Branch:** `feature/cleanup-post-redesign` (a partir de `00e78e2`, merge do redesign em `main`)
**Sessões:** iniciada 2026-05-20, pausada 2026-05-20

## Status por fase

| Fase | Status | Hash | Notas |
|---|---|---|---|
| **FASE 0** — análise + REDESIGN-LOG.md | ✅ concluída | — | mapeamento do header antigo (linhas 1680–1758): logo + dark/⚙️/🚪 + painel `{ss && …}` (Empresa/Acessos/Backup + 2 modais). Decisões aprovadas (sidebar 1º item config, tabs internas Empresa/Acessos/Backup, ícone ⚙️, JSX antes do Dashboard, visibilidade admin-only). |
| **FASE 1** — criar view `configuracoes` | ✅ commitada | **`8eecd63`** | `+94/-3`. allTabs/SB_GROUPS/ROLES.admin.tabs adicionam `"configuracoes"`. Avatar dropdown ganha item ⚙️ (admin-only). View renderiza Card com 3 tabs internas (Empresa/Acessos/Backup) + 2 modais copiados. Handlers preservados 1:1. Babel ok (897.772 chars). Testado local: save empresa OK, convite/listagem usuários OK, exportar/importar backup OK. |
| **FASE 2** — remover header antigo | ❌ **revertida** | — | tentativa aplicou 3 edits parciais (header sticky + parte do `{ss && …}` + state `ss`) mas deixou ÓRFÃOS: Gerenciar Acessos + 2 modais + Backup buttons row + tag de fechamento `</div></div>}`. Babel quebrou em JSX linha 1724 col 3772 (= arquivo linha ~1750). `git restore index.html` reverteu pra `8eecd63`. Working tree limpo. |
| **FASE 3** — remover toggle dark duplicado (Dashboard ~1775) | ⏳ pendente | — | depende da FASE 2 (toggle inline do Dashboard só deve sair quando o header novo for a única fonte). |
| **FASE 4** — remover seletor de idioma 🌐 | ⏳ pendente | — | escopo: botão, dropdown PT/EN/ES, state `lang`/`langOpen`, e decidir sobre `localStorage["lang"]`. |
| **FASE 5** — CLEANUP-LOG final + merge `main` | ⏳ pendente | — | depende de FASE 2/3/4. |

## Aprendizado da FASE 2 revertida

**Causa raiz:** o `{ss && <div>…</div>}` é um bloco grande (~78 linhas) e contém os 2 modais que foram copiados 1:1 pra view nova na FASE 1. Os modais duplicados impedem usar anchors do conteúdo dos modais no Edit tool (não-únicos). Tentei Edit B parcial (removeu só do início até "Salvar Empresa") esperando completar com edits subsequentes — mas isso deixou o JSX sintaticamente inválido entre passos e o working tree foi visto pelo Eduardo no browser.

**Regra pra próxima sessão:** FASE 2 deve ser **uma operação atômica** que remove TUDO de uma vez (header sticky `<div>` + `{ss && <div>…</div>}` inteiro + state `[ss,setSs]`). Validação babel parser DEVE rodar antes do commit. Se a operação não couber em um único `Edit`, usar script Node ad-hoc com `indexOf(startStr)` + `indexOf(endStr, startIdx)`.

**Anchors já validados pra próxima tentativa:**
- Start único: `marginTop:14,padding:"12px 14px",background:T.c2` (só aparece em 1 lugar no arquivo — painel Gerenciar Acessos antigo)
- Header sticky start: `linear-gradient(135deg,#111833,#0A0F1E)":"linear-gradient(135deg,#FFFFFF,#F0F4FF)` (parte do `<div>` do header sticky)
- End único: o backup buttons row termina com `</button></div></div>}` precedido de `Relatório Completo (CSV)` (essa string aparece **duas vezes** — header antigo + view nova; remover **só a primeira ocorrência**).

## Prompt cirúrgico sugerido pra retomar FASE 2

> Em `feature/cleanup-post-redesign`, remove o **header sticky antigo** + o **bloco `{ss && <div>…</div>}` inteiro** + o **state `[ss,setSs]`** em uma operação só. Use script Node ad-hoc com `indexOf` se necessário — não parcialize. Anchors: header sticky começa em `<div style={{background:darkMode?"linear-gradient(135deg,#111833,#0A0F1E)"`, painel ss começa em `{ss&&<div style={{background:T.cd,padding:"14px 18px"`, bloco termina (primeira ocorrência) em `Relatório Completo (CSV)</button></div></div>}`. State em `const[ss,setSs]=useState(false);`. **Antes de commitar**: rodar `sunny-validate.cjs` e confirmar babel parser OK. Se falhar, `git restore index.html` e replanejar.

## Arquivos untracked nesta sessão

- `REDESIGN-LOG.md` — da sessão anterior (redesign), decisão de commit/descarte adiada.
- `CLEANUP-LOG.md` — este arquivo. Não commitar ainda.

## Restrições absolutas que continuam valendo

1. Não toca em conteúdo de outras telas
2. Não mexe em handlers de Firebase/save/load
3. Não bumpa service worker (`sunny-v4` permanece)
4. Babel parser **obrigatório antes de cada commit**
5. Não push sem OK explícito do Eduardo
