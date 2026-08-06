# Testing with DUnitX

In-depth DUnitX reference: fixtures, parameterized tests, mocks, and CI execution.

## Fixtures and Lifecycle

Use `[Setup]` and `[TearDown]` to prepare and clean up state before/after each test; `[SetupFixture]`/`[TearDownFixture]` run once per class:

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

## Parameterized Tests

Use `[TestCase]` to run the same test with multiple data sets:

```pascal
[Test]
[TestCase('Valid CPF', '123.456.789-09,True')]
[TestCase('Invalid CPF', '111.111.111-11,False')]
procedure DeveValidarCPF(const cpf: string; const esperado: Boolean);
begin
  Assert.AreEqual(esperado, Validador.ValidarCPF(cpf));
end;
```

## Testing Exceptions

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

## Mocks and Isolation

DUnitX doesn't ship with a native mocking framework. To isolate dependencies, use interfaces and hand-write test doubles, or adopt a mocking library such as Delphi-Mocks or Spring4D Mocking:

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

Inject `IEmailSender` into the tested service via the constructor, so the real implementation can be swapped for the mock in tests.

## Running in Continuous Integration (CI)

Build the test project with `msbuild`/`dcc32` and run it via the DUnitX console runner (`DUnitX.ConsoleRunner`), which returns a non-zero exit code on failure — suitable for CI pipelines:

```bash
msbuild MeuProjetoTestes.dproj /p:Config=Release
MeuProjetoTestes.exe --format:nunit --output:resultados.xml
```

Publish `resultados.xml` (NUnit/JUnit format) to your CI system for test reporting.
