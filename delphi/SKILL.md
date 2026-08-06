---
name: delphi
description: Best practices and conventions for Delphi / Object Pascal projects (VCL and FireMonkey). Use when writing or reviewing Delphi code — memory management, exceptions, data access, naming, unit organization, and testing.
---

# Delphi

Generic skill for writing Delphi / Object Pascal code following best practices, keeping consistency across VCL and FireMonkey (FMX) projects.

## Quick Reference

* Memory management: always release objects with `try/finally`; see [Memory Management and Exceptions](#memory-management-and-exceptions).
* Naming: `T` for types, `I` for interfaces, `F` for private fields; see [Naming and Conventions](#naming-and-conventions).
* Data access: use parameters (`TParams`) instead of concatenating SQL; see [Data Access](#data-access) and the [FireDAC reference](references/data-access.md) for connections, transactions, and Array DML.
* Organization: one unit per class/form; keep business logic out of `Forms`; see [Unit Organization](#unit-organization).
* Testing: use DUnitX; see [Testing](#testing) and the [testing reference](references/testing.md) for fixtures, parameterized tests, mocks, and CI execution.
* Modern features: generics, inline vars, RTTI/attributes; see [Modern Features](#modern-features-delphi-103).

## Memory Management and Exceptions

Always release objects with `try/finally`, even when an exception may occur:

```pascal
var
  Lista: TStringList;
begin
  Lista := TStringList.Create;
  try
    Lista.Add('Item 1');
    // use Lista
  finally
    Lista.Free;
  end;
end;
```

For collections of objects, use `TObjectList<T>` with `OwnsObjects := True` so release is automatic:

```pascal
var
  Clientes: TObjectList<TCliente>;
begin
  Clientes := TObjectList<TCliente>.Create(True); // OwnsObjects
  try
    Clientes.Add(TCliente.Create);
  finally
    Clientes.Free;
  end;
end;
```

Use `try/except` only where the exception will actually be handled. Don't catch exceptions just to silence them:

```pascal
try
  Conexao.Open;
except
  on E: EDatabaseError do
  begin
    Log.Error('Failed to connect: ' + E.Message);
    raise;
  end;
end;
```

## Naming and Conventions

* Classes: `TCliente`, `TPedidoService`
* Interfaces: `IClienteRepository`
* Private fields: `FNome`, `FIdade`
* Constants: `MAX_TENTATIVAS` or `cMaxTentativas`
* Methods and properties: PascalCase with clear verbs: `CalcularTotal`, `ValidarCPF`
* Parameters and local variables: `camelCase`

```pascal
type
  TCliente = class
  private
    FNome: string;
    FIdade: Integer;
  public
    property Nome: string read FNome write FNome;
    property Idade: Integer read FIdade write FIdade;
    function ValidarCPF(const cpf: string): Boolean;
  end;
```

## Data Access

Always use parameters to avoid SQL Injection — never concatenate values directly into the SQL string:

```pascal
Query.SQL.Text := 'SELECT * FROM clientes WHERE id = :id';
Query.ParamByName('id').AsInteger := ClienteId;
Query.Open;
```

Encapsulate data access in a repository (DAO) layer, separated from business logic and UI:

```pascal
type
  IClienteRepository = interface
    function BuscarPorId(Id: Integer): TCliente;
  end;
```

See the [FireDAC reference](references/data-access.md) for named connections, pooling, transactions, Array DML, and usage in threads.

## Unit Organization

* One unit per class or form.
* In the `interface` section's `uses` clause, include only what's strictly needed; move dependencies only used in the body to the `implementation` section's `uses`.
* Avoid business logic in `Forms`; extract it into Services/Controllers, applying MVC/MVP where it makes sense.
* Prefer injecting dependencies via interfaces, so implementations can be swapped in tests.

```pascal
unit Clientes.Service;

interface

uses
  Clientes.Model;

type
  TClienteService = class
  public
    function Validar(const Cliente: TCliente): Boolean;
  end;

implementation

uses
  System.SysUtils, System.RegularExpressions;

// implementation uses units that don't need to be in the interface

end.
```

## Modern Features (Delphi 10.3+)

* Generics: `TList<T>`, `TDictionary<K,V>`, `TObjectList<T>`
* Inline variables inside blocks: `var i := 10;`
* Attributes and RTTI for serialization: `System.JSON`, `REST.Json`

```pascal
var
  Cliente: TCliente;
  JsonStr: string;
begin
  Cliente := TCliente.Create;
  try
    JsonStr := TJson.ObjectToJsonString(Cliente);
  finally
    Cliente.Free;
  end;
end;
```

## Testing

Use DUnitX for unit tests, isolating external dependencies via interfaces and mocks:

```pascal
[TestFixture]
TCalculadoraTests = class
public
  [Test]
  procedure DeveSomarDoisNumeros;
end;

procedure TCalculadoraTests.DeveSomarDoisNumeros;
begin
  Assert.AreEqual(4, Calculadora.Somar(2, 2));
end;
```

See the [testing reference](references/testing.md) for fixtures (`[Setup]`/`[TearDown]`), parameterized tests (`[TestCase]`), interface-based mocks, and CI pipeline execution.

## Tooling

* Package manager: Boss or GetIt Package Manager.
* Code style: follow the community [Delphi/Object Pascal Coding Standards](https://github.com/omonien/DelphiStandards) as a formatting and convention reference.
* Version control: keep `.dproj`/`.dfm` files under version control, but avoid unnecessary diffs (watch out for IDE Insight and local settings).
