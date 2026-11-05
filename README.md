# ClaudeCODEX

Método para desenvolver com o **Claude Code** e revisar com o **Codex**, cada um
no seu papel: o Owner decide, o Claude Code implementa e conduz as rodadas, e o
Codex revisa de forma independente pela CLI local, com a assinatura do ChatGPT e
sem API. É o método usado na revisão da v1.1.0 do `mvp_eng_dados_1`, escrito para
ser reaproveitado.

Não é skill nem plugin: é um **prompt** e um **kit** de ferramentas simples.

## Como usar num projeto novo

Clone em `~/Projetos/claudecodex` — é onde o prompt procura o kit:

```bash
git clone https://github.com/alencardoug/ClaudeCODEX ~/Projetos/claudecodex
```

1. Cole o conteúdo de [`PROMPT_ClaudeCodeX.md`](PROMPT_ClaudeCodeX.md) no primeiro
   prompt do Claude Code, no projeto novo. Ele está em inglês e foi escrito para o
   agente executar, não para leitura humana.
2. O próprio prompt manda o Claude Code copiar o [`kit/`](kit/) para o projeto (§2)
   e pedir o seu OK para cada arquivo.

## O que há aqui

| Caminho | Para quê |
|---|---|
| `PROMPT_ClaudeCodeX.md` | O protocolo: papéis, como chamar o Codex (`codex` e `codex2`, `gpt-6.1-sol`, esforço `high`), lotes e rodadas, tratamento de achados, critério de parada, o modelo de prompt de rodada e os defeitos recorrentes |
| `kit/revisao/dossie.py` | Gera o `REVISAO.md` da rodada 1: escopo com SHA fixo, mapa declaração × derivado, saída literal dos comandos de verificação e as seções que só o autor preenche; `--conferir` recusa dossiê incompleto |
| `kit/revisao/comandos.txt` · `ambiente.md` | Os comandos de verificação do projeto e a tabela do que a revisão precisa encontrar de pé |
| `kit/secrets/secrets_review.py` · `segredos_tratados.yml` | Revisão de segredos nos arquivos rastreados e, com `--historico`, em todo o histórico; o registro dos achados já tratados |
| `kit/AGENTS.md` | O que o Codex lê: papel, o que pode e o que não pode rodar, formato do relatório |
| `kit/Makefile.fragment` | Os alvos `check-offline`, `check`, `secrets-history`, `review-dossier` e `review-check` |
| `docs_para_portfolio/` | Material de portfólio |

## Requisitos

- Claude Code e a CLI do Codex (`codex`), com login por ChatGPT nas contas que
  forem usadas (`~/.codex` e, opcionalmente, `~/.codex_conta2`).
- Python 3.10+ e `git`; PyYAML só para `secrets_review.py --historico`.

## Histórico

Este repositório se chamava `astramax` e abrigava o *AstraMax Reviewer* (uma skill de
revisão), substituído por este método e renomeado para `ClaudeCODEX` em 05/11/2026. O
conteúdo anterior continua no histórico do `git`; o endereço antigo redireciona para este.
