# Convencoes Git

- Repositorio: ajuste-google-drive
- Empresa: pv
- BasePath: C:\codes
- ModeloRepositorio: mono-repositorio
- Provedor remoto padrao: github

## Branches

1. Formato recomendado: `tipo/escopo-descricao-curta`.
2. Tipos permitidos: `feature`, `fix`, `docs`, `chore`, `refactor`, `test`.
3. Usar minusculas, numeros e hifens.
4. Incluir chamado quando ajudar rastreabilidade.

## Commits

1. Formato recomendado: `tipo(escopo): resumo curto`.
2. O resumo deve dizer o que mudou.
3. Referenciar chamado no corpo quando aplicavel.
4. Evitar mensagens genericas como `update`, `fix`, `alteracoes` ou `wip`.

## Bootstrap

1. Usar `maintain-git` para criar, preparar e sincronizar este repositorio quando necessario.
2. Conferir `git status` antes de branch, commit, pull, merge, rebase ou push.
3. Nao executar commit ou push na `main`/`master`; criar branch de trabalho antes.
4. Nao executar operacoes destrutivas sem pedido explicito.
