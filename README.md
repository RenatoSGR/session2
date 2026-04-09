# Guia de Estrutura de Pastas — .NET 10

> Guia prático para iniciar um novo projecto .NET 10 com uma estrutura de pastas recomendada, convenções de nomenclatura e boas práticas de organização de código.

---

## Pré-requisitos

| Requisito | Versão mínima |
|---|---|
| [.NET SDK](https://dotnet.microsoft.com/download/dotnet/10.0) | **10.0** |
| Git | qualquer versão recente |
| IDE | Visual Studio 2022 / Rider / VS Code + C# Dev Kit |

Verifique a instalação:

```bash
dotnet --version
# deve devolver algo como 10.0.xxx
```

---

## Estrutura de Solução (Monorepo)

Adoptar uma estrutura monorepo permite manter todo o código, testes, documentação e scripts de build num único repositório Git, facilitando:

- **Consistência**: uma única versão dos pacotes partilhados.
- **Refactoring atómico**: mudanças que abrangem vários projectos são feitas num único commit.
- **CI/CD simplificado**: um pipeline por repositório.

---

## Árvore de Pastas

```
/MyProject                         ← raiz do repositório
│
├── MyProject.sln                  ← ficheiro de solução
│
├── global.json                    ← versão do SDK fixada
├── .editorconfig                  ← regras de estilo de código
├── Directory.Packages.props       ← gestão centralizada de versões NuGet
├── Directory.Build.props          ← propriedades MSBuild partilhadas
│
├── src/                           ← código da aplicação
│   ├── MyProject.Domain/          ← entidades, value objects, interfaces
│   ├── MyProject.Application/     ← casos de uso, DTOs, interfaces de serviços
│   ├── MyProject.Infrastructure/  ← EF Core, repositórios, serviços externos
│   └── MyProject.Api/             ← ASP.NET Core Web API (entrada HTTP)
│
├── tests/                         ← testes automatizados
│   ├── MyProject.Domain.Tests/    ← testes unitários de domínio
│   ├── MyProject.Application.Tests/
│   ├── MyProject.Infrastructure.Tests/
│   └── MyProject.Api.IntegrationTests/
│
├── docs/                          ← documentação de arquitectura
│   ├── architecture-decision-records/   ← ADRs
│   └── diagrams/                  ← diagramas C4, ER, sequência
│
├── build/                         ← scripts de build e helpers de CI
│   ├── build.ps1
│   ├── build.sh
│   └── pipelines/                 ← YAML do GitHub Actions / Azure Pipelines
│
├── samples/                       ← (opcional) projectos de exemplo / demos
│
└── tools/                         ← (opcional) ferramentas locais (dotnet-tools)
    └── .config/
        └── dotnet-tools.json
```

---

## Camadas — Clean Architecture

```
          ┌──────────────────────────────────┐
          │           MyProject.Api          │  ← entrypoint HTTP
          └──────────────────┬───────────────┘
                             │ depende de
          ┌──────────────────▼───────────────┐
          │       MyProject.Application      │  ← orquestra casos de uso
          └───────────┬──────────────────────┘
                      │ depende de
          ┌───────────▼──────────────────────┐
          │        MyProject.Domain          │  ← núcleo de negócio (puro C#)
          └──────────────────────────────────┘
                      ▲
          ┌───────────┴──────────────────────┐
          │     MyProject.Infrastructure     │  ← implementa interfaces do Domain/App
          └──────────────────────────────────┘
```

| Camada | Responsabilidade | Dependências |
|---|---|---|
| **Domain** | Entidades, Value Objects, interfaces de repositório, eventos de domínio | Nenhuma (zero dependências externas) |
| **Application** | Casos de uso (Commands/Queries via MediatR), DTOs, validações (FluentValidation) | Domain |
| **Infrastructure** | EF Core DbContext, repositórios, clientes HTTP, filas, cache | Application, Domain |
| **Api** | Controllers/Minimal APIs, middleware, DI wiring | Application, Infrastructure |

---

## Convenções de Nomenclatura

| Artefacto | Padrão | Exemplo |
|---|---|---|
| Solução | `<NomeProjecto>.sln` | `MyProject.sln` |
| Projecto de produção | `<NomeProjecto>.<Camada>` | `MyProject.Domain` |
| Projecto de testes | `<NomeProjecto>.<Camada>.Tests` | `MyProject.Domain.Tests` |
| Testes de integração | `<NomeProjecto>.<Camada>.IntegrationTests` | `MyProject.Api.IntegrationTests` |
| Namespace raiz | igual ao nome do projecto | `MyProject.Domain` |
| Classes | PascalCase | `OrderService` |
| Interfaces | `I` + PascalCase | `IOrderRepository` |

---

## Criação da Solução e Projectos

### 1. Criar a solução

```bash
dotnet new sln -n MyProject
```

### 2. Criar os projectos (`src/`)

```bash
# Domínio (class library sem framework)
dotnet new classlib -n MyProject.Domain     -o src/MyProject.Domain     -f net10.0

# Aplicação
dotnet new classlib -n MyProject.Application -o src/MyProject.Application -f net10.0

# Infra-estrutura
dotnet new classlib -n MyProject.Infrastructure -o src/MyProject.Infrastructure -f net10.0

# API (ASP.NET Core Web API)
dotnet new webapi   -n MyProject.Api        -o src/MyProject.Api         -f net10.0
```

### 3. Criar os projectos de testes (`tests/`)

```bash
dotnet new xunit -n MyProject.Domain.Tests               -o tests/MyProject.Domain.Tests               -f net10.0
dotnet new xunit -n MyProject.Application.Tests          -o tests/MyProject.Application.Tests          -f net10.0
dotnet new xunit -n MyProject.Infrastructure.Tests       -o tests/MyProject.Infrastructure.Tests       -f net10.0
dotnet new xunit -n MyProject.Api.IntegrationTests       -o tests/MyProject.Api.IntegrationTests       -f net10.0
```

### 4. Adicionar todos à solução

```bash
# Produção
dotnet sln add src/MyProject.Domain/MyProject.Domain.csproj
dotnet sln add src/MyProject.Application/MyProject.Application.csproj
dotnet sln add src/MyProject.Infrastructure/MyProject.Infrastructure.csproj
dotnet sln add src/MyProject.Api/MyProject.Api.csproj

# Testes
dotnet sln add tests/MyProject.Domain.Tests/MyProject.Domain.Tests.csproj
dotnet sln add tests/MyProject.Application.Tests/MyProject.Application.Tests.csproj
dotnet sln add tests/MyProject.Infrastructure.Tests/MyProject.Infrastructure.Tests.csproj
dotnet sln add tests/MyProject.Api.IntegrationTests/MyProject.Api.IntegrationTests.csproj
```

### 5. Adicionar referências entre projectos

```bash
# Application depende de Domain
dotnet add src/MyProject.Application reference src/MyProject.Domain

# Infrastructure depende de Application e Domain
dotnet add src/MyProject.Infrastructure reference src/MyProject.Application
dotnet add src/MyProject.Infrastructure reference src/MyProject.Domain

# Api depende de Application e Infrastructure
dotnet add src/MyProject.Api reference src/MyProject.Application
dotnet add src/MyProject.Api reference src/MyProject.Infrastructure

# Projectos de testes referenciam o projecto correspondente
dotnet add tests/MyProject.Domain.Tests reference src/MyProject.Domain
dotnet add tests/MyProject.Application.Tests reference src/MyProject.Application
dotnet add tests/MyProject.Infrastructure.Tests reference src/MyProject.Infrastructure
dotnet add tests/MyProject.Api.IntegrationTests reference src/MyProject.Api
```

---

## Gestão Centralizada de Pacotes NuGet

Crie `Directory.Packages.props` na **raiz** do repositório para controlar versões num único sítio:

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <!-- Frameworks e runtime -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi"   Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />

    <!-- Aplicação -->
    <PackageVersion Include="MediatR"                        Version="12.*" />
    <PackageVersion Include="FluentValidation"               Version="11.*" />

    <!-- Testes -->
    <PackageVersion Include="xunit"                          Version="2.*" />
    <PackageVersion Include="FluentAssertions"               Version="6.*" />
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.0" />
  </ItemGroup>
</Project>
```

Nos `.csproj` individuais, **omita a versão**:

```xml
<PackageReference Include="MediatR" />
```

---

## Ficheiros de Configuração Globais

### `global.json` — fixar versão do SDK

Coloque na **raiz** do repositório:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestMinor"
  }
}
```

### `.editorconfig` — estilo de código

Coloque na **raiz** do repositório. Exemplo mínimo:

```ini
root = true

[*.cs]
indent_style             = space
indent_size              = 4
end_of_line              = lf
charset                  = utf-8-bom
trim_trailing_whitespace = true
insert_final_newline     = true

# Naming rules
dotnet_naming_rule.interface_should_be_begins_with_i.symbols  = interface
dotnet_naming_rule.interface_should_be_begins_with_i.style    = begins_with_i
dotnet_naming_rule.interface_should_be_begins_with_i.severity = warning
```

### `Directory.Build.props` — propriedades MSBuild partilhadas

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <!-- Analisadores de estilo -->
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisMode>All</AnalysisMode>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="8.*" />
    <PackageReference Include="StyleCop.Analyzers"                  Version="1.2.*" />
  </ItemGroup>
</Project>
```

> **Dica:** Com `Directory.Build.props` na raiz, todas as propriedades acima aplicam-se automaticamente a **todos** os projectos da solução, sem necessitar de repetição em cada `.csproj`.

---

## Comandos Úteis do Dia-a-Dia

```bash
# Restaurar pacotes
dotnet restore

# Build completo
dotnet build

# Executar todos os testes
dotnet test

# Executar a API
dotnet run --project src/MyProject.Api

# Publicar para produção
dotnet publish src/MyProject.Api -c Release -o ./publish
```

---

## Referências

- [Documentação oficial .NET 10](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-10)
- [Clean Architecture com ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/best-practices)
- [Gestão centralizada de pacotes NuGet](https://learn.microsoft.com/nuget/consume-packages/central-package-management)
- [global.json](https://learn.microsoft.com/dotnet/core/tools/global-json)
- [.editorconfig no .NET](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/configuration-files)
