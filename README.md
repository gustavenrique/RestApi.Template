# Brand-Api-Template
Esta é uma template, que pode ser usada como base para APIs potencialmente complexas e monstruosas.

Como testar:
```bash
git clone https://github.com/gustavenrique/Api.Template.git

dotnet run --project Api.Template/src/Brand.Template.Api/Brand.Template.Api.csproj

# Abrir http://localhost:5150/docs
```

# Libraries
Estas são as principais libraries externas usadas para lidar com problemas genéricos:

- Logging: [Serilog](https://serilog.net/)
- Auto-mapping: [Mapster](https://github.com/MapsterMapper/Mapster)
- Persistence: [Dapper](https://github.com/DapperLib/Dapper)
- Resilience: [Polly](https://github.com/App-vNext/Polly?tab=readme-ov-file#polly)
- Code Analysis: [SonarAnalyzer](https://github.com/SonarSource/sonar-dotnet)
- Testing:
  - [XUnit](https://xunit.net/)
  - [NSubstitute](https://github.com/nsubstitute/NSubstitute?tab=readme-ov-file#nsubstitute)
  - [FluentAssertions](https://fluentassertions.com/)
- Outros:
  - [MediatR](https://github.com/jbogard/MediatR?tab=readme-ov-file#mediatr)
  - [Refit](https://github.com/reactiveui/refit?tab=readme-ov-file#refit-the-automatic-type-safe-rest-library-for-net-core-xamarin-and-net)

# Secret Management

O arquivo `Directory.Build.props` possui a prop de UserSecretsId. A intenção de tal prop é buscar as credenciais de algum Azure Key Vault, que contenha todas as secrets necessárias. Para tudo funcionar, o diretório `~/appdata/roaming/microsoft/usersecrets/d0e79c52-7784-4098-933b-5eabfaebe774` precisa ter o arquivo `secrets.json` com as seguintes propriedades:

```json
{
  "KeyVault:Url": "https://xpto.vault.azure.net/",
  "KeyVault:TenantId": "00000000-0000-0000-0000-000000000000",
  "KeyVault:ClientId": "00000000-0000-0000-0000-000000000000",
  "KeyVault:ClientSecret": "secret-xpto"
}
```

Vale mencionar que o Client, representando um Service Principal, por exemplo, precisa ter a access policy configurada para acesso ao Get e List do key vault.

# Health check
O health check deve verificar a disponibilidade de todos os serviços externos usados pela API,
desde bancos e APIs, até serviços de service bus. Portanto, sempre que fizermos uma adição/exclusão de serviços externos consumidos,
também deve ser atualizada a configuração de health check, localizada em `src/Brand.Template.Api/DependencyInjection.cs`, no método `AddHealthChecking`.

O endpoint que expõe os dados de health check é o `/_health`. Outrossim, vale ressaltar que o mesmo pode ser consumido através da UI
encontrada em `/dashboard` (apenas existente fora do stage de Production).

# Versionamento
Ao realizar uma manutenção na API, caso esteja sendo feita alguma 'breaking change', ela deve ser realizada numa versão diferente, 
de modo a facilitar rollback e/ou evitar bugs inesperados. Alguns exemplos de cenários são:

- Mudança de contratos de HTTP request e/ou response
- Alteração de status codes retornados
- Ajuste em path de endpoint

# Metrics
Partindo da premissa de que o Prometheus seria como repositório de métricas, as métricas da aplicação, 
incluindo health check e consumo de computing resources, são expostas através do meta endpoint `/_metrics`.
Ou seja, é este o endpoint que deve ser configurado no servidor do Prometheus, tal como ilustrado no prometheus.yaml abaixo:

```yml
global:
    scrape_interval: 10s

scrape_configs:
    - job_name: 'api-xpto-job'
      metrics_path: /_metrics
      static_configs:
        - targets: ['api-xpto:8080']
```

# Estrutura
O projeto como um todo foi estruturado com base em conceitos, princípios e patterns de Clean Architecture, Vertical Slice Architecture e Domain-Driven Design (DDD), 
visando promover, respectivamente, desacoplamento, coesão e manuteniblidade.

Cada camada, teoricamente, deve ter responsabilidades bem claras e pouco acopladas às outras. De modo geral, cada camada normalmente terá algumas ou todas as seguintes caracaterísticas:
- Divisão por features
- Contém diretório 'Common', para recursos compartilhados entre diferentes features
- Responsabilidade própria de injeção de dependência
- Classe de configuração própria (Settings.cs), baseada no appsettings.json

## Presentation
Expõe a aplicação para agentes externos. Nesse caso, através de endpoints HTTP

```
📂---src
|   📂---Brand.Xpto.Api
|   |   |   Brand.Xpto.Api.csproj
|   |   |   DependencyInjection.cs
|   |   |   Dockerfile
|   |   |   Program.cs
|   |   |   Settings.cs
|   |   📂---Controllers
|   |   📂---Filters
|   |   📂---Middlewares
```

## Application
A camada de aplicação deve orquestrar os domain models e, eventualmente, fazer uso dos domain services. Desse modo, ela é responsável 
majoritariamente por assuntos de aplicação, como comunicação com agentes externos através de abstrações, mas também pode acabar contendo uma ou outra lógica de negócio.
```
|   📂---Brand.Xpto.Application
|   |   |   Brand.Xpto.Application.csproj
|   |   |   DependencyInjection.cs
|   |   |   Settings.cs
|   |   📂---Common
|   |   📂---Foo
|   |       |   FooMapper.cs
|   |       📂---Abstractions
|   |       📂---Dtos
|   |       📂---Errors
|   |       📂---Services
|   |           |   BarService.cs
|   |           |   BazService.cs
```

## Domain
Responsável por concentrar a maioria das lógicas de negócio, dentro das domain models e domain services
```
|   📂---Brand.Xpto.Domain
|   |   |   Brand.Xpto.Domain.csproj
|   |   📂---Common
|   |   📂---Foo
|   |       📂---Abstractions
|   |           |   IBarRepository.cs
|   |           |   IBazRepository.cs
|   |       📂---Events
|   |       📂---Models
|   |           |   Foo.cs
|   |           📂---Entities
|   |           📂---ValueObjects
```

## Infrastructure/Persistence/DataAccess
Implementa o consumo de serviços externos
```
|   📂---Brand.Xpto.Infra
|   |   |   DependencyInjection.cs
|   |   |   Brand.Xpto.Infra.csproj
|   |   |   Settings.cs
|   |   📂---Common     
|   |   📂---Foo
|   |       |   IFooApiClient.cs
|   |       |   FooMapper.cs
|   |       📂---Dtos
|   |       📂---Repositories
|   |               BarRepository.cs
|   |               BazRepository.cs
```

## Shared Kernel
Concentra **recursos internos, relacionados ao Domain**, mas compartilhados entre diferentes 'bounded contexts'.
Na prática, terá domain models e abstrações de negócios que podem ser compartilhadas entre as aplicações do Brand, os quais 
poderiam ser migrados para uma lib interna eventualmente.
```
|   📂---Brand.SharedKernel
|       |   DependencyInjection.cs
|       |   Brand.SharedKernel.csproj
|       📂---Abstractions
|       |   |   AggregateRoot.cs
|       |   |   Entity.cs
|       |   |   ValueObject.cs
|       |   📂---Events
|       |   |       IDomainEvent.cs
|       |   |       IDomainEventDispatcher.cs
|       |   |       IDomainEventBrand.cs
|       |   📂---ResultType
|       |           Result.cs
|       |           ResultReason.cs
|       |           ResultT.cs
|       📂---Services
|               DomainEventDispatcher.cs
```

## Tests
- Os testes devem ser separados em uma pasta além da `src\`
- No caso de **testes unitários**, deve haver um projeto de teste para cada camada testada
- Cada projeto de teste deve preferencialmente seguir uma estrutura de pastas parecida com a da aplicação
```
📂---tests
    📂---Brand.Xpto.Api.Tests.Integration
    📂---Brand.Xpto.Application.Tests.Subcutaneous
    📂---Brand.Xpto.Application.Tests.Unit 
    📂---Brand.Xpto.Domain.Tests.Unit
    📂---Brand.SharedKernel.Tests.Unit
        |   GlobalUsings.cs
        |   Brand.SharedKernel.Tests.Unit.csproj
        📂---Abstractions
        |   |   EntityTests.cs
        |   |   ValueObjectTests.cs
        |   📂---ResultType
        |   |       ResultTests.cs
        |   |       ResultTTests.cs
``` 
