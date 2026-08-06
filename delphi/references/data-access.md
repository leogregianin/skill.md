# Data Access with FireDAC

In-depth FireDAC reference: connections, transactions, batch operations, and error handling.

## Connections

Prefer configuring connections through `TFDConnection` with a named Connection Definition, instead of building the `ConnectionString` manually in every form:

```pascal
FDManager.ConnectionDefFileName := 'FDConnectionDefs.ini';
FDManager.Active := True;

Conexao := TFDConnection.Create(nil);
Conexao.ConnectionDefName := 'MinhaConexaoPrincipal';
Conexao.Connected := True;
```

Use connection pooling in server applications (multi-thread or multi-request) by enabling `Pooled=True` on the connection definition:

```
[MinhaConexaoPrincipal]
DriverID=FB
Server=localhost
Database=meubanco.fdb
Pooled=True
POOL_MaximumItems=50
```

## Transactions

Always wrap operations that modify multiple tables in an explicit transaction:

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

Avoid long-running transactions — open, run the required operations, and finalize (`Commit`/`Rollback`) as soon as possible to avoid holding locks on the database.

## Batch Operations (Array DML)

To insert or update many records, use Array DML instead of looping with multiple `ExecSQL` calls:

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

This drastically reduces the number of database round-trips compared to executing one INSERT at a time.

## TFDQuery vs TFDMemTable vs TFDTable

* `TFDQuery`: for specific SQL queries, with or without parameters. The most common choice.
* `TFDTable`: direct access to a table without writing SQL; useful for simple record-editing screens, but less efficient than `TFDQuery` for complex queries.
* `TFDMemTable`: a 100% in-memory dataset, no database connection; useful for temporary data, local caching, or decoupling the UI from the data source (e.g. `TFDMemTable.Data := Query.Data`).

## Error Handling

Catch FireDAC-specific exceptions to handle database errors differently from other errors:

```pascal
try
  Conexao.Connected := True;
except
  on E: EFDDBEngineException do
  begin
    Log.Error('Database error: ' + E.Message);
    raise;
  end;
end;
```

## Usage in Threads

Each thread must have its own `TFDConnection` instance — never share a connection across threads. Use pooling (`Pooled=True`) so each thread gets a connection from the pool without the overhead of constant reconnection.
