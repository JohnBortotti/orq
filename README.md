# orq

Orquestrador de agentes sobre `tmux`, `git worktree` e `systemd`.

CLI fina: cada subcomando é uma ou duas chamadas de tmux, git ou systemd com nome
bonito. A orquestração — grafo, ordem, quem faz o quê — não mora aqui; mora na
skill e no prompt.

**O plano é a fonte de verdade**, e vive fora deste repo:
`notes/projects/orq/index.md` no vault.

## Estado

Fase 1 não implementada. Este repo tem a superfície de comandos e as constantes
que o plano fixa; nenhuma lógica.

## Instalação (quando houver o que instalar)

```bash
ln -s ~/workspace/orq/orq ~/.local/bin/orq
```

Requer `tmux`, `git` e `python3` — todos já presentes. Sem dependência de terceiros.
