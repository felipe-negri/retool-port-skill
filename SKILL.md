---
name: retool-to-next
description: Migrar um app Retool para páginas Next.js (App Router). Usar quando receber um export JSON do Retool (ou pedido para substituir/migrar uma ferramenta interna do Retool) e for preciso reconstruir a funcionalidade num app Next. Decodifica o export, inventaria queries/widgets, classifica tudo e guia a reconstrução.
---

# Retool → Next.js

Porta o que o app Retool **faz**, não o layout nem o estilo. O JSON é spec de comportamento, não de design.
Adaptado da skill `retool-migration` do civitai (alvo original: SvelteKit); as partes específicas deles foram removidas.

## Setup (uma vez)

```bash
cd <pasta-da-skill>/scripts && npm install   # pg + transit-js
```

O export **não é JSON puro**: `page.data.appState` é string **transit-js**. Não tente ler o arquivo cru; use `extract.mjs`.

## 0. Descobrir as convenções do projeto-alvo (antes de qualquer código)

Não assuma stack. Leia no repo Next: router (app/ vs pages/), ORM/query builder (Prisma, Drizzle, Kysely…),
lib de UI (shadcn/Tailwind?), auth/roles, padrão de mutação (Server Actions vs route handlers) e CLAUDE.md/AGENTS.md.
Siga o que já existe; se faltar algo decisivo (auth, onde fica o DB), **pergunte**.

## 1. Inventariar

```bash
node scripts/extract.mjs "<export.json>"            # inventário completo
node scripts/extract.mjs "<export.json>" --queries  # só SQL
node scripts/extract.mjs "<export.json>" --json     # com dataBindings por widget
```

**Nunca commitar o export cru** — costuma ter `Authorization: Bearer …` hardcoded. Guarde fora do repo e commite só o inventário gerado (tem SQL, sem config de auth). Se um token apareceu, trate como vazado.

As queries são *a maior parte* da spec, **não toda**. O resto está nos widgets:

- **Opções de dropdown** (`_labels` estático, query ligada ao widget, ou uma query `Function`) são capacidade. Porte a *fonte*, não um subconjunto visto em screenshot.
- 🔴 **Toda query `Function` deve ser aberta.** É onde o Retool guarda listas sem tabela (motivos, mensagens canônicas, flags). O extractor mostra só o nome — parece cola, não é.
- 🔴 **Leia a expressão `data` do widget; não infira o binding por nome plausível.** Um select pode depender de outro campo (`{{ action.value === 'a' ? ListA.value : ListB.value }}`) enquanto existe uma query de nome óbvio ligada a nada.
- `{{ }}` pode conter JS arbitrário: leia como dica de fluxo de dados, não traduza literalmente.
- Algo que o usuário descreve e não está no inventário? `grep -io "texto[^\"]\{0,60\}" export.json` antes de concluir que não existe.

## 2. Classificar TODA query antes de construir — sem port parcial

Um app = um slice inteiro. Retool continua no ar até o port estar completo; port parcial faz o usuário confiar na ferramenta errada.

| Bucket | Significado |
| --- | --- |
| **port** | Funcionalidade real. Construir. |
| **equivalent** | Coberto por outra forma aqui — dizer qual. |
| **plumbing** | Cola do Retool sem significado de servidor (`State`, `Timer`, agrupamento de tabela). |
| **superseded** | v1 com v2 presente, ou duplicata. Nomear a vencedora. |
| **blocked** | Falta chave/sistema/decisão. Dizer exatamente o que desbloqueia. |

Query fora de bucket = não portada por acidente. Commite a classificação ao lado do inventário.
"Quais queries estão em uso" não sai do export: **pergunte**, mas silêncio ≠ permissão pra descartar (bucket `port`).
Comentário solto em ticket ("talvez isso seja outro app") é pergunta aberta, não decisão.

## 3. Mapear resources do Retool

| Resource Retool | Aqui |
| --- | --- |
| Postgres/MySQL (leitura) | réplica/cliente read-only do ORM do projeto |
| Postgres (escrita) | cliente de escrita, **só via camada de serviço server-side** |
| `retool_db` | banco próprio do app (ver abaixo) |
| REST resource / `REST-WithoutResource` | route handler existente, ou chamada server-side ao serviço — **leia a URL antes de classificar** |
| Query "workflow"/Function JS | função TS no servidor (ou constante tipada se for só lista) |

`{{ }}` vira input real: query params para filtros, campos de formulário para ações. **Nunca interpolar em SQL** — use o builder do ORM ou queries parametrizadas.

⚠️ **Query Retool muitas vezes é REST para o backend, não escrita em tabela.** Classificar só pelo nome da tabela perde tudo que o endpoint faz em volta (sync de índice, notificações, cache, tracking). Antes de chamar algo de "não portado", procure o endpoint/serviço que já existe no backend e **leia o schema (zod/validação), não o nome** — costuma aceitar mais que o Retool enviava e já carrega as permissões por ação.

### Banco `retool_db` (leitura para escopo)

```bash
cp scripts/.env.example scripts/.env  # preencher RETOOL_DATABASE_URL
node scripts/retool-db.mjs --tables
node scripts/retool-db.mjs --describe <Tabela>
node scripts/retool-db.mjs 'SELECT * FROM "<Tabela>" LIMIT 5'
```
Somente leitura: escritas são recusadas antes de conectar. Credencial vem do cofre/Vault da void, nunca hardcoded.
Colunas de autoria (`createdBy`, `updatedBy`) costumam ser texto livre com *nome de exibição*; grave o username do usuário logado e deixe o mapeamento para id como follow-up.

## 4. Construir

- **Rota**: `app/<área>/<slug>/page.tsx` (Server Component carrega dados via serviço) + componentes client só onde há interação.
- **Dados**: serviço em `lib/server/<dominio>.ts`; query builder por padrão, SQL cru só p/ bitmask, funções PG, jsonb/LATERAL.
- **Mutações**: Server Actions (ou route handlers) → mesma camada de serviço; validar input (zod) no limite; `revalidatePath` após escrever.
- **Filtros**: a query string é a fonte da verdade (`searchParams`), nunca estado local.
- **Layout**: container com várias panes = **sub-rotas, uma por pane** (`[tab]`), não uma página longa. Modal = dialog. Pane com `only visible when` é **gate de role/estado que não aparece em nenhuma query** — porte o gate (middleware/checagem server-side, não só esconder no client).
- **Leituras de investigação** devem usar a réplica de leitura, se existir; `write` só com motivo escrito.
- Mantenha os nomes das queries do Retool (`GetHelpers`, `GetImageData`) em nomes de funções: é como o time as chama.
- Registrar a rota na navegação/permissões do app e avisar o usuário se o acesso depende de grant manual.

## 5. Verificar (gate, não formalidade)

1. **Cobertura**: releia a classificação do §2 contra o que foi construído; toda query tem que estar contabilizada.
2. **Ser NOMEADA não é ser COBERTA.** Para cada query leia o **corpo SQL**: o que o WHERE filtra, que colunas seleciona, de onde vem o input. Uma query cujo input é um widget que você não construiu não foi portada, mesmo mapeada para um endpoint (exemplos reais: query que listava itens *já removidos* = o workflow inteiro de restauração; entrada por "lista colada de ids" perdida; um "nukeUser" que só removia imagens mapeado para um "purge all" de blast radius maior).
3. **Fidelidade**: percorra o export query por query e confira se o comportamento existe e bate. Revisões de código comparam o código a ele mesmo e **não abrem o export** — passam limpas sobre uma implementação fiel da coisa errada. Verifique as alegações do review contra o inventário antes de agir.
4. Rodar typecheck, lint e build do projeto.
5. Lacunas que só screenshot fecha: **handlers de evento** (o que cada clique dispara) e **widgets de nível de frame** (toolbar persistente fora do container) não existem no export. Peça screenshots/acesso ao app vivo.

## Gotchas

- Export fica velho no momento em que é tirado; se o comportamento parecer estranho, pergunte em vez de inferir intenção do SQL.
- Operações destrutivas portadas (ban, remover, purgar) merecem confirmação na UI e log de auditoria.
- Um slice por sessão: commite o inventário + classificação no repo para a próxima sessão não depender de contexto.
