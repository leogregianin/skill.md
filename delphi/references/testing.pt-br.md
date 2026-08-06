# Testes com DUnitX

Referência aprofundada de DUnitX: fixtures, testes parametrizados, mocks e execução em CI.

## Fixtures e Ciclo de Vida

Use `[Setup]` e `[TearDown]` para preparar e limpar o estado antes/depois de cada teste; `[SetupFixture]`/`[TearDownFixture]` rodam uma vez por classe:

```pascal
[TestFixture]
TClienteServiceTests = class
private
  FService: TClienteService;
public
  [Setup]
  procedure Setup;
  [TearDown]
  procedure TearDown;

  [Test]
  procedure DeveValidarEmailValido;
end;

procedure TClienteServiceTests.Setup;
begin
  FService := TClienteService.Create;
end;

procedure TClienteServiceTests.TearDown;
begin
  FService.Free;
end;

procedure TClienteServiceTests.DeveValidarEmailValido;
begin
  Assert.IsTrue(FService.ValidarEmail('a@b.com'));
end;
```

## Testes Parametrizados

Use `[TestCase]` para rodar o mesmo teste com múltiplos conjuntos de dados:

```pascal
[Test]
[TestCase('CPF valido', '123.456.789-09,True')]
[TestCase('CPF invalido', '111.111.111-11,False')]
procedure DeveValidarCPF(const cpf: string; const esperado: Boolean);
begin
  Assert.AreEqual(esperado, Validador.ValidarCPF(cpf));
end;
```

## Testando Exceções

```pascal
[Test]
procedure DeveLancarExcecaoParaEmailInvalido;
begin
  Assert.WillRaise(
    procedure
    begin
      FService.ValidarEmail('email-invalido');
    end,
    EValidationError
  );
end;
```

## Mocks e Isolamento

DUnitX não inclui um framework de mock nativo. Para isolar dependências, use interfaces e implemente dublês de teste manualmente, ou adote uma biblioteca de mocking como Delphi-Mocks ou Spring4D Mocking:

```pascal
type
  IEmailSender = interface
    procedure Enviar(const Destinatario, Assunto, Corpo: string);
  end;

  TEmailSenderMock = class(TInterfacedObject, IEmailSender)
  public
    ChamadasEnviar: Integer;
    procedure Enviar(const Destinatario, Assunto, Corpo: string);
  end;

procedure TEmailSenderMock.Enviar(const Destinatario, Assunto, Corpo: string);
begin
  Inc(ChamadasEnviar);
end;
```

Injete `IEmailSender` no serviço testado via construtor, permitindo trocar a implementação real pelo mock nos testes.

## Execução em Integração Contínua (CI)

Compile o projeto de testes com `msbuild`/`dcc32` e execute via o console runner do DUnitX (`DUnitX.ConsoleRunner`), que retorna código de saída não-zero em caso de falha — adequado para pipelines de CI:

```bash
msbuild MeuProjetoTestes.dproj /p:Config=Release
MeuProjetoTestes.exe --format:nunit --output:resultados.xml
```

Publique `resultados.xml` (formato NUnit/JUnit) no seu sistema de CI para relatórios de teste.
