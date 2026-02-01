# API Fundamentals — Web API com ASP.NET Core (.NET 6)

Este repositório contém um **projeto de Web API desenvolvido em C# com ASP.NET Core (.NET 6)**, com foco na implementação de um **CRUD completo** utilizando **Entity Framework Core 6** e **SQL Server Express**.

O projeto foi desenvolvido como parte do aprendizado dos fundamentos de Web API, REST, CRUD e ORM com Entity Framework Core.

---

## 📌 Tecnologias utilizadas

* **C#**
* **.NET 6 (ASP.NET Core Web API)**
* **Entity Framework Core 6**
* **SQL Server Express**
* **Swagger (OpenAPI)**

---

## 📁 Estrutura do projeto

```text
api-fundamentals/
│
├── Controllers/
│   └── ContatoController.cs
│
├── Context/
│   └── AgendaContext.cs
│
├── Entities/
│   └── Contato.cs
│
├── Migrations/
│
├── appsettings.json
├── appsettings.Development.json
├── Program.cs
└── api-fundamentals.csproj
```

---

## 📦 Entity Framework Core (EFC)

O **Entity Framework Core** é um ORM (Object-Relational Mapper) que faz a ponte entre:

* as **classes do sistema**
* e as **tabelas do banco de dados**

Ou seja, trabalhamos com **objetos em C#**, e o EF Core se encarrega de converter isso em comandos SQL.

---

## 📥 Instalação dos pacotes

Para este projeto (.NET 6), são utilizadas as versões **6.x** do EF Core.

```bash
dotnet add package Microsoft.EntityFrameworkCore.Design --version 6.0.36
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 6.0.36
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 6.0.36
```

Ferramenta de migração (CLI):

```bash
dotnet tool install --global dotnet-ef --version 6.0.36
```

---

## 🧱 Entidade (Tabela)

Cada classe dentro da pasta **Entities** representa uma tabela no banco de dados.

Exemplo: **Contato.cs**

```csharp
namespace api_fundamentals.Entities
{
    public class Contato
    {
        public int Id { get; set; }
        public string Nome { get; set; }
        public string Telefone { get; set; }
        public bool Ativo { get; set; }
    }
}
```

---

## 🗂️ DbContext

O **DbContext** centraliza todas as informações do banco de dados.

Ele herda de `DbContext` e recebe as configurações via injeção de dependência.

```csharp
using Microsoft.EntityFrameworkCore;
using api_fundamentals.Entities;

public class AgendaContext : DbContext
{
    public AgendaContext(DbContextOptions<AgendaContext> options)
        : base(options) { }

    public DbSet<Contato> Contatos { get; set; }
}
```

### 🔎 Sobre o construtor

* `DbContextOptions` contém as configurações do banco (provider, connection string, etc.)
* `base(options)` repassa essas configurações para o `DbContext`
* O construtor fica vazio porque nenhuma lógica extra é necessária

---

## 🔌 Configuração da conexão com o banco de dados

### Ambiente de desenvolvimento

Arquivo: `appsettings.Development.json`

```json
"ConnectionStrings": {
  "ConexaoPadrao": "Server=localhost\\sqlexpress;Initial Catalog=Agenda;Integrated Security=True"
}
```

#### Explicação:

* `Server=localhost\\sqlexpress` → instância local do SQL Server Express
* `Initial Catalog=Agenda` → nome do banco de dados
* `Integrated Security=True` → autenticação via usuário do Windows

---

### Exemplo em ambiente real (produção)

```json
"ConnectionStrings": {
  "ConexaoPadrao": "Server=meu-servidor;Database=Agenda;User Id=usuario;Password=senha;TrustServerCertificate=True"
}
```

> ⚠️ Em produção, o ideal é **não versionar senhas**, utilizando variáveis de ambiente ou serviços de secret.

---

## ⚙️ Configuração do DbContext no Program.cs

```csharp
builder.Services.AddDbContext<AgendaContext>(
    options => options.UseSqlServer(
        builder.Configuration.GetConnectionString("ConexaoPadrao")
    )
);
```

Essa configuração:

* Registra o `AgendaContext` no **container de injeção de dependência**
* Define o **SQL Server** como provider
* Utiliza a connection string definida nos arquivos `.json`

O `DbContext` tem ciclo de vida **Scoped** (uma instância por requisição HTTP).

---

## 🔁 Migrations

As migrations fazem o mapeamento das classes para estrutura de banco de dados.

### Criar migration

```bash
dotnet ef migrations add CriacaoTabelaContato
```

> Nesse momento, nada ainda foi alterado no banco.

### Aplicar no banco

```bash
dotnet ef database update
```

O EF Core:

* Cria as tabelas
* Ajusta tipos (ex: `string` → `varchar`)
* Mantém histórico das alterações

---

## 🌐 Controllers e Endpoints

### Create (POST)

```csharp
[HttpPost]
public IActionResult Create(Contato contato)
{
    _context.Add(contato);
    _context.SaveChanges();

    return CreatedAtAction(
        nameof(ObterPorId),
        new { id = contato.Id },
        contato
    );
}
```

---

### Read — Obter por ID (GET)

```csharp
[HttpGet("{id}")]
public IActionResult ObterPorId(int id)
{
    var contato = _context.Contatos.Find(id);

    if (contato == null)
        return NotFound();

    return Ok(contato);
}
```

---

### Update (PUT)

```csharp
[HttpPut("{id}")]
public IActionResult Atualizar(int id, Contato contato)
{
    var contatoBanco = _context.Contatos.Find(id);

    if (contatoBanco == null)
        return NotFound();

    contatoBanco.Nome = contato.Nome;
    contatoBanco.Telefone = contato.Telefone;
    contatoBanco.Ativo = contato.Ativo;

    _context.Contatos.Update(contatoBanco);
    _context.SaveChanges();

    return Ok(contatoBanco);
}
```

---

### Delete (DELETE)

```csharp
[HttpDelete("{id}")]
public IActionResult Deletar(int id)
{
    var contatoBanco = _context.Contatos.Find(id);

    if (contatoBanco == null)
        return NotFound();

    _context.Contatos.Remove(contatoBanco);
    _context.SaveChanges();

    return NoContent();
}
```

---

### Read — Obter por nome (GET)

```csharp
[HttpGet("ObterPorNome")]
public IActionResult ObterPorNome(string nome)
{
    var contato = _context.Contatos
        .Where(x => x.Nome.Contains(nome));

    return Ok(contato);
}
```

---

## 🔗 CRUD x Verbos HTTP

| Operação | Verbo HTTP |
| -------- | ---------- |
| Create   | POST       |
| Read     | GET        |
| Update   | PUT        |
| Delete   | DELETE     |

* Uma mesma rota pode responder a verbos diferentes
* Cada endpoint deve refletir corretamente a ação do CRUD

---

## 💾 SaveChanges()

O método `SaveChanges()`:

* Converte alterações em SQL
* Executa comandos no banco
* Confirma a transação

Sem ele, **nenhuma modificação é persistida**.

---

## 📖 Swagger

O projeto utiliza **Swagger por padrão** (ASP.NET Core .NET 6).

Execute:

```bash
dotnet watch run
```

Acesse:

```
https://localhost:<porta>/swagger
```

---

## 🎯 Objetivo educacional

* Consolidar os fundamentos de Web API na prática
* Demostrar o uso correto de REST e CRUD
* Realizar integração com banco de dados via Entity Framework Core
* Utilizar Migrations e versionamento de schema
* Estar sempre de acordo com as boas práticas em ASP.NET Core

---

## 👩‍💻 Autora

**Heloisa Giacometti**  
Estudante de Ciência da Computação  
Interesse em desenvolvimento backend

---

📌 *Projeto desenvolvido durante estudos em Web API na plataforma DIO, com adaptações e organização próprias para fins de aprendizado e portfólio.*
