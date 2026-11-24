# retool-port-skill

Skill para agentes de código (Claude Code, Antigravity) que guia a migração de um app **Retool** para código próprio, em **qualquer stack** (Next.js, SvelteKit, Nuxt, Remix, Rails…): decodifica o export, inventaria queries e widgets, classifica tudo e orienta a reconstrução e a verificação.

## Instalar

```bash
# Claude Code
git clone git@github.com:felipe-negri/retool-port-skill.git ~/.claude/skills/retool-port
# Antigravity (workspace)
git clone git@github.com:felipe-negri/retool-port-skill.git <projeto>/.agent/skills/retool-port

cd <pasta-clonada>/scripts && npm install
```

## Usar

Passe o export JSON do Retool ao agente e peça: *"porte este app usando a skill retool-port"*.
O fluxo está em [`SKILL.md`](SKILL.md): convenções do projeto → inventário → classificação de toda query → mapeamento de resources → construção → verificação de cobertura e fidelidade.

## Scripts (`scripts/`)

| Script | Para quê |
| --- | --- |
| `extract.mjs <export.json> [--queries\|--json]` | Decodifica o export (transit-js) em inventário de queries, SQL, widgets, abas e bindings |
| `retool-db.mjs --tables \| --describe <T> \| "<SELECT>"` | Consulta **somente leitura** ao `retool_db` (`RETOOL_DATABASE_URL` em `scripts/.env`) |

> Nunca commite o export cru do Retool: costuma ter tokens `Authorization` hardcoded. Commite só o inventário gerado.

## Licença e atribuição

Derivada de [`retool-migration`](https://github.com/civitai/civitai/tree/main/.claude/skills/retool-migration), © 2022 Civitai, sob **Apache-2.0**. Alterações listadas em [`NOTICE`](NOTICE); texto completo em [`LICENSE`](LICENSE). Sem relação com a Retool Inc.
