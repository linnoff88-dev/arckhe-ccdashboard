# 05 — Manter a base atualizada

O painel só é têo novo quanto a última cópia para o banco.

```text
Crie uma tarefa agendada que, [todo dia às 6h / toda segunda às 7h], recopie os dados de
[app 1] e [app 2] para o schema [nome] no meu Supabase, do mesmo jeito que fizemos no piloto.
Só acrescente ou atualize registros; nêo apague nada. No fim de cada execuçêo, grave numa
tabela [schema].execucoes a data, quantos registros entraram e se houve erro.
Adicione ao painel um cartêo "Última atualizaçêo" lendo essa tabela.
```
