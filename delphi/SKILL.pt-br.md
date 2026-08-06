---
name: delphi
description: Boas práticas e convenções para projetos Delphi / Object Pascal (VCL e FireMonkey). Use ao escrever ou revisar código Delphi — gerenciamento de memória, exceções, acesso a dados, nomenclatura, organização de units e testes.
---

# Delphi

Skill genérico para escrever código Delphi / Object Pascal seguindo boas práticas, mantendo consistência entre projetos VCL e FireMonkey (FMX).

## Referência Rápida

* Gerenciamento de memória: sempre libere objetos com `try/finally`; veja [Gerenciamento de Memória e Exceções](#gerenciamento-de-memória-e-exceções).
* Nomenclatura: `T` para types, `I` para interfaces, `F` para campos privados; veja [Nomenclatura e Convenções](#nomenclatura-e-convenções).
* Acesso a dados: use parâmetros (`TParams`) em vez de concatenar SQL; veja [Acesso a Dados](#acesso-a-dados) e a [referência de FireDAC](references/data-access.pt-br.md) para conexões, transações e Array DML.
* Organização: uma unit por classe/formulário; lógica de negócio fora de `Forms`; veja [Organização de Units](#organização-de-units).
* Testes: use DUnitX; veja [Testes](#testes) e a [referência de testes](references/testing.pt-br.md) para fixtures, testes parametrizados, mocks e execução em CI.
* Recursos modernos: generics, inline vars, RTTI/atributos; veja [Recursos Modernos](#recursos-modernos-delphi-103).

## Gerenciamento de Memória e Exceções

Sempre libere objetos com `try/finally`, mesmo quando uma exceção pode ocorrer:

```pascal
var
  Lista: TStringList;
begin
  Lista := TStringList.Create;
  try
    Lista.Add('Item 1');
    // usa Lista
  finally
    Lista.Free;
  end;
end;
```

Para coleções de objetos, use `TObjectList<T>` com `OwnsObjects := True` para que a liberação seja automática:

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

Use `try/except` apenas onde a exceção realmente será tratada. Não capture exceções apenas para silenciá-las:

```pascal
try
  Conexao.Open;
except
  on E: EDatabaseError do
  begin
    Log.Error('Falha ao conectar: ' + E.Message);
    raise;
  end;
end;
```

## Nomenclatura e Convenções

* Classes: `TCliente`, `TPedidoService`
* Interfaces: `IClienteRepository`
* Campos privados: `FNome`, `FIdade`
* Constantes: `MAX_TENTATIVAS` ou `cMaxTentativas`
* Métodos e propriedades: PascalCase com verbos claros: `CalcularTotal`, `ValidarCPF`
* Parâmetros e variáveis locais: `camelCase`

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

## Acesso a Dados

Use sempre parâmetros para evitar SQL Injection — nunca concatene valores diretamente na string SQL:

```pascal
Query.SQL.Text := 'SELECT * FROM clientes WHERE id = :id';
Query.ParamByName('id').AsInteger := ClienteId;
Query.Open;
```

Encapsule acesso a dados em uma camada de repositório (DAO), separada da lógica de negócio e da UI:

```pascal
type
  IClienteRepository = interface
    function BuscarPorId(Id: Integer): TCliente;
  end;
```

Veja a [referência de FireDAC](references/data-access.pt-br.md) para conexões nomeadas, pooling, transações, Array DML e uso em threads.

## Organização de Units

* Uma unit por classe ou formulário.
* Na cláusula `uses` da `interface`, inclua apenas o mínimo necessário; mova dependências que só são usadas no corpo para a `uses` da `implementation`.
* Evite lógica de negócio em `Forms`; extraia para Services/Controllers, aplicando MVC/MVP quando fizer sentido.
* Prefira injetar dependências via interfaces, permitindo substituir implementações em testes.

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

// implementação usa units que não precisam estar na interface

end.
```

## Recursos Modernos (Delphi 10.3+)

* Generics: `TList<T>`, `TDictionary<K,V>`, `TObjectList<T>`
* Variáveis inline dentro de blocos: `var i := 10;`
* Atributos e RTTI para serialização: `System.JSON`, `REST.Json`

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

## Testes

Use DUnitX para testes unitários, isolando dependências externas via interfaces e mocks:

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

Veja a [referência de testes](references/testing.pt-br.md) para fixtures (`[Setup]`/`[TearDown]`), testes parametrizados (`[TestCase]`), mocks via interfaces e execução em pipelines de CI.

## Ferramentas

* Gerenciador de pacotes: Boss ou GetIt Package Manager.
* Estilo de código: siga o [Delphi/Object Pascal Coding Standards](https://github.com/omonien/DelphiStandards) da comunidade como referência de formatação e convenções.
* Controle de versão: mantenha arquivos `.dproj`/`.dfm` sob controle de versão, mas evite gerar diffs desnecessários (cuidado com IDE Insight e configurações locais).
