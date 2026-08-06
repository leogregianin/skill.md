# Acesso a Dados com FireDAC

Referência aprofundada de FireDAC: conexões, transações, operações em lote e tratamento de erros.

## Conexões

Prefira configurar conexões via `TFDConnection` com definição de conexão nomeada (Connection Definition), em vez de montar a `ConnectionString` manualmente em cada formulário:

```pascal
FDManager.ConnectionDefFileName := 'FDConnectionDefs.ini';
FDManager.Active := True;

Conexao := TFDConnection.Create(nil);
Conexao.ConnectionDefName := 'MinhaConexaoPrincipal';
Conexao.Connected := True;
```

Use pooling de conexões em aplicações servidor (multi-thread ou multi-request), ativando `Pooled=True` na definição da conexão:

```
[MinhaConexaoPrincipal]
DriverID=FB
Server=localhost
Database=meubanco.fdb
Pooled=True
POOL_MaximumItems=50
```

## Transações

Sempre delimite operações que modificam múltiplas tabelas dentro de uma transação explícita:

```pascal
Conexao.StartTransaction;
try
  QueryPedido.ExecSQL;
  QueryItens.ExecSQL;
  Conexao.Commit;
except
  Conexao.Rollback;
  raise;
end;
```

Evite transações longas — abra, execute as operações necessárias e finalize (`Commit`/`Rollback`) o quanto antes, para não segurar locks no banco.

## Operações em Lote (Array DML)

Para inserir ou atualizar muitos registros, use Array DML em vez de um loop com múltiplos `ExecSQL`:

```pascal
Query.SQL.Text := 'INSERT INTO itens (pedido_id, produto_id, quantidade) VALUES (:pedido_id, :produto_id, :quantidade)';
Query.Params.ArraySize := QtdItens;

for i := 0 to QtdItens - 1 do
begin
  Query.Params[0].AsIntegers[i] := PedidoId;
  Query.Params[1].AsIntegers[i] := Itens[i].ProdutoId;
  Query.Params[2].AsIntegers[i] := Itens[i].Quantidade;
end;

Query.Execute(QtdItens);
```

Isso reduz drasticamente o número de round-trips ao banco comparado a executar um INSERT por vez.

## TFDQuery vs TFDMemTable vs TFDTable

* `TFDQuery`: para consultas SQL específicas, com ou sem parâmetros. Uso mais comum.
* `TFDTable`: acesso direto a uma tabela sem escrever SQL; útil para telas simples de cadastro, mas menos eficiente que `TFDQuery` para consultas complexas.
* `TFDMemTable`: dataset 100% em memória, sem conexão com banco; útil para dados temporários, cache local, ou para desacoplar a UI da fonte de dados (ex.: `TFDMemTable.Data := Query.Data`).

## Tratamento de Erros

Capture exceções específicas do FireDAC para tratar erros de banco de forma diferenciada de outros erros:

```pascal
try
  Conexao.Connected := True;
except
  on E: EFDDBEngineException do
  begin
    Log.Error('Erro de banco: ' + E.Message);
    raise;
  end;
end;
```

## Uso em Threads

Cada thread deve ter sua própria instância de `TFDConnection` — não compartilhe uma conexão entre threads. Use pooling (`Pooled=True`) para que cada thread obtenha uma conexão do pool sem overhead de reconexão constante.
