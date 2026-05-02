<a id="capitulo-41"></a>
# Capítulo 41: Newtype, Builder, RAII

> *"A primitive type is a missed opportunity to teach the compiler about your domain."*
> — Scott Wlaschin, *Domain Modeling Made Functional*

> *"Resource Acquisition Is Initialization is the most important pattern in C++. Rust took it, made it default, and called it Tuesday."*
> — atribuído à comunidade Rust

## 41.1 O Problema da Primitiva Anêmica

Olhe esta assinatura, em qualquer linguagem que você prefira:

```typescript
function transferir(de: number, para: number, valor: number): void
```

```go
func Transferir(de int64, para int64, valor int64) error
```

```java
void transferir(long de, long para, long valor)
```

Pergunta: o que impede um programador, distraído às 17h45 numa sexta-feira, de chamar `transferir(valor, de, para)`? Resposta: **nada**. Os três parâmetros são do mesmo tipo. O compilador não tem como saber que o primeiro é um identificador de origem, o segundo de destino e o terceiro um montante em centavos.

Isso é o que a literatura de design chama de **primitive obsession** — modelar conceitos do domínio com tipos genéricos da linguagem. O custo é silencioso: bugs que compilam, passam em testes superficiais, e quebram em produção quando o argumento errado entra na ordem errada.

Rust dá um remédio quase indecente em sua simplicidade: o **newtype**.

## 41.2 Newtype: Um Tipo Por Conceito

Um *newtype* em Rust é uma *tuple struct* de um único campo:

```rust
struct UserId(u64);
struct PostId(u64);
struct AmountCents(i64);
```

São três tipos *distintos*. Para o compilador, `UserId` e `PostId` não são intercambiáveis, ainda que ambos embrulhem um `u64`. O custo em runtime é **zero**: a representação binária é idêntica à do `u64` cru. O custo em compile time é uma camada de segurança que torna bugs de troca de argumento impossíveis.

```rust
fn deletar_usuario(id: UserId) { /* ... */ }

let post = PostId(42);
deletar_usuario(post);
// erro: expected `UserId`, found `PostId`
```

```mermaid
graph LR
    Domain[Domínio:<br/>UserId, PostId, Email,<br/>AmountCents, OrderId]
    Compiler[Compilador:<br/>tipos distintos]
    Runtime[Runtime:<br/>u64 cru, zero overhead]

    Domain --> Compiler
    Compiler -.zero-cost.-> Runtime

    style Domain fill:#c8e6c9,stroke:#1b5e20
    style Compiler fill:#bbdefb,stroke:#0d47a1
    style Runtime fill:#fff9c4,stroke:#f57f17
```

### 41.2.1 Newtype Como Modelagem de Domínio

A regra prática é simples: **se dois valores nunca devem ser confundidos, eles não devem ter o mesmo tipo**. Um e-mail não é uma string. Um CPF não é uma string. Centavos não são reais. Milissegundos não são segundos.

```rust
struct Email(String);
struct Cpf(String);
struct Milliseconds(u64);
struct Seconds(u64);

fn aguardar(t: Milliseconds) { /* ... */ }

let timeout = Seconds(30);
aguardar(timeout); // erro de compilação
```

A função `aguardar` declara *exatamente* a unidade que aceita. Confundir minutos com segundos custou à NASA o orbitador de Marte em 1999 (US$ 327 milhões). Em Rust, esse bug não compila.

### 41.2.2 Construção Validada: Parse, Don't Validate

O newtype ganha músculo quando combinado com construtores fechados:

```rust
pub struct Email(String);

impl Email {
    pub fn parse(raw: &str) -> Result<Self, EmailError> {
        if !raw.contains('@') {
            return Err(EmailError::MissingAt);
        }
        if raw.len() > 254 {
            return Err(EmailError::TooLong);
        }
        Ok(Email(raw.to_owned()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

#[derive(thiserror::Error, Debug)]
pub enum EmailError {
    #[error("e-mail sem '@'")]
    MissingAt,
    #[error("e-mail excede 254 caracteres")]
    TooLong,
}
```

O campo `String` é *privado*. A única forma de obter um `Email` é passar pelo `parse`. Isso significa que **toda função que aceita `Email` pode confiar que o valor já foi validado**. Você nunca mais escreve `if (!isValidEmail(input))` em vinte lugares — a validação acontece *uma vez*, na fronteira.

Esse é o princípio *Parse, don't validate* de Alexis King: transforme dados não-confiáveis em tipos que carregam a prova de sua validade. O newtype é o veículo dessa transformação.

### 41.2.3 A Tentação `Deref` (e Por Que Quase Sempre Resistir)

O Rust permite que um newtype implemente `Deref<Target=u64>`, fazendo com que o valor seja *automaticamente* tratável como o tipo embrulhado:

```rust
use std::ops::Deref;

impl Deref for UserId {
    type Target = u64;
    fn deref(&self) -> &u64 { &self.0 }
}

let id = UserId(42);
let dobrado = *id * 2; // funciona — Deref converte em u64
```

Isso parece conveniente, e quase sempre é uma armadilha. Ao implementar `Deref`, você deu ao chamador o direito de usar `UserId` *como* um `u64`. Toda a segurança que o newtype oferecia evapora: agora `transferir(*origem, *destino, valor)` compila sem queixa.

A regra é áspera: **`Deref` em newtypes deve ser reservado para *smart pointers* genuínos** (como `Box<T>`, `Arc<T>`, `String`→`&str`). Para newtypes de domínio, exponha métodos explícitos:

```rust
impl UserId {
    pub fn new(value: u64) -> Self { Self(value) }
    pub fn value(&self) -> u64 { self.0 }
}
```

A regra de ouro do newtype: o esforço extra de digitar `id.value()` em vez de `*id` é exatamente o esforço que impede o bug.

### 41.2.4 Comparação: TypeScript, Go, Java

| Linguagem | Newtype real? | Custo runtime | Compilador checa? |
|---|---|---|---|
| **Rust** | `struct UserId(u64)` | zero | **sim** |
| **TypeScript** | branded type via intersection | zero | sim (em compile time, apaga em runtime) |
| **Go** | `type UserId int64` | zero | **parcial** (conversão implícita em literais) |
| **Java** | classe wrapper | objeto extra na heap | sim, mas verboso |

TypeScript chega perto com *branded types*:

```typescript
type UserId = number & { readonly __brand: 'UserId' };
type PostId = number & { readonly __brand: 'PostId' };

function makeUserId(n: number): UserId { return n as UserId; }

const u: UserId = makeUserId(1);
const p: PostId = u; // erro de tipo
```

Funciona, mas a marca existe apenas para o type-checker. Em runtime, `UserId` e `PostId` são `number` puro, e nada impede um cast malicioso. Em Rust, o tipo é real até o último byte.

Go tem `type UserId int64`, mas o compilador permite conversões entre tipos derivados do mesmo subjacente em literais e operações aritméticas com surpreendente liberalidade — a barreira é mais frouxa do que a do Rust.

Java exige uma classe inteira (`final class UserId`), com construtor, `equals`, `hashCode`, `toString`. Cada `UserId` é um objeto na heap. O custo conceitual é alto, e por isso quase ninguém faz isso na prática — é por isso que Java está cheio de `transferir(long, long, long)`.

## 41.3 Builder: Construindo Sem Telefone-Sem-Fio

Algumas structs têm muitos campos. A maioria opcionais. Construtores posicionais ficam ilegíveis (`new(true, false, None, 30, "utf-8", None)` — qual campo é qual?). E Rust não tem argumentos nomeados de função.

A solução idiomática é o **builder pattern**.

### 41.3.1 Builder Manual

```rust
pub struct HttpClient {
    base_url: String,
    timeout: std::time::Duration,
    user_agent: String,
    max_retries: u32,
}

pub struct HttpClientBuilder {
    base_url: Option<String>,
    timeout: Option<std::time::Duration>,
    user_agent: Option<String>,
    max_retries: Option<u32>,
}

impl HttpClientBuilder {
    pub fn new() -> Self {
        Self { base_url: None, timeout: None, user_agent: None, max_retries: None }
    }

    pub fn base_url(mut self, url: impl Into<String>) -> Self {
        self.base_url = Some(url.into());
        self
    }

    pub fn timeout(mut self, t: std::time::Duration) -> Self {
        self.timeout = Some(t);
        self
    }

    pub fn user_agent(mut self, ua: impl Into<String>) -> Self {
        self.user_agent = Some(ua.into());
        self
    }

    pub fn max_retries(mut self, n: u32) -> Self {
        self.max_retries = Some(n);
        self
    }

    pub fn build(self) -> Result<HttpClient, BuildError> {
        Ok(HttpClient {
            base_url: self.base_url.ok_or(BuildError::MissingBaseUrl)?,
            timeout: self.timeout.unwrap_or(std::time::Duration::from_secs(30)),
            user_agent: self.user_agent.unwrap_or_else(|| "rust-app/1.0".into()),
            max_retries: self.max_retries.unwrap_or(3),
        })
    }
}
```

Uso:

```rust
let client = HttpClientBuilder::new()
    .base_url("https://api.example.com")
    .timeout(std::time::Duration::from_secs(10))
    .build()?;
```

Cada método consome `self` e devolve `Self` — o estilo *consuming builder*. Cada chamada move o builder, evitando aliases e mutações compartilhadas. O `build()` final faz validação e produz o objeto imutável.

### 41.3.2 Builder Com `derive_builder`

Escrever todo isso à mão para cada struct é tedioso. A crate `derive_builder` gera o builder via macro:

```rust
use derive_builder::Builder;

#[derive(Builder, Debug)]
#[builder(setter(into), build_fn(error = "BuildError"))]
pub struct HttpClient {
    base_url: String,
    #[builder(default = "std::time::Duration::from_secs(30)")]
    timeout: std::time::Duration,
    #[builder(default = "\"rust-app/1.0\".into()")]
    user_agent: String,
    #[builder(default = "3")]
    max_retries: u32,
}
```

O atributo `setter(into)` permite passar `&str` em vez de `String`. `default` define valores caso o campo seja omitido. O builder gerado se chama `HttpClientBuilder` automaticamente.

Para builders com checagem em compile time de campos obrigatórios, a crate `typed-builder` aplica o type-state pattern (próximo capítulo): chamar `.build()` sem definir um campo obrigatório falha *na compilação*.

### 41.3.3 Comparação: Java Builder

Java popularizou o builder via Joshua Bloch (*Effective Java*). O custo é dolorido:

```java
public class HttpClient {
    private final String baseUrl;
    private final Duration timeout;
    private final String userAgent;
    private final int maxRetries;

    private HttpClient(Builder b) {
        this.baseUrl = b.baseUrl;
        this.timeout = b.timeout;
        this.userAgent = b.userAgent;
        this.maxRetries = b.maxRetries;
    }

    public static class Builder {
        private String baseUrl;
        private Duration timeout = Duration.ofSeconds(30);
        private String userAgent = "java-app/1.0";
        private int maxRetries = 3;

        public Builder baseUrl(String v) { this.baseUrl = v; return this; }
        public Builder timeout(Duration v) { this.timeout = v; return this; }
        public Builder userAgent(String v) { this.userAgent = v; return this; }
        public Builder maxRetries(int v) { this.maxRetries = v; return this; }

        public HttpClient build() {
            if (baseUrl == null) throw new IllegalStateException("baseUrl required");
            return new HttpClient(this);
        }
    }
}
```

Cada nova struct exige uma classe `Builder` aninhada com campos espelhados, setters, validação manual e construtor privado. Lombok (`@Builder`) automatiza, mas adiciona dependência e magia de bytecode. Em Rust, `#[derive(Builder)]` é uma linha — sem reflexão, sem runtime, sem surpresa.

## 41.4 RAII: Recurso É Posse

O acrônimo RAII vem de C++: *Resource Acquisition Is Initialization*. A ideia é que *adquirir um recurso* (arquivo aberto, conexão de banco, mutex travado) deve ser equivalente a *inicializar uma variável*; e *liberar o recurso* deve acontecer *automaticamente* quando a variável sai de escopo.

Rust não tem RAII como uma escolha de design entre outras. **RAII é o único modelo que Rust oferece**. O traço `Drop` torna isso trivial:

```rust
struct ConexaoBanco {
    socket: TcpStream,
}

impl Drop for ConexaoBanco {
    fn drop(&mut self) {
        eprintln!("fechando conexão");
        // socket fecha automaticamente quando o struct é destruído
    }
}

fn main() {
    let conn = ConexaoBanco { socket: TcpStream::connect("...").unwrap() };
    // ... usar conn ...
} // conn.drop() chamado aqui, garantido, mesmo em panic
```

O destruidor roda **sempre**: no fim do escopo, em retorno antecipado, em `?` que propagou erro, em `panic!` desenrolando o stack. Não há caminho de saída em que o recurso vaze (exceto `std::mem::forget` ou `Box::leak`, deliberadamente).

### 41.4.1 Os Quatro Casos Canônicos

```rust
// Arquivo
{
    let f = std::fs::File::open("dados.txt")?;
    // ler, processar...
} // arquivo fechado aqui, mesmo se houve erro acima

// Mutex guard
{
    let guard = mutex.lock().unwrap();
    *guard += 1;
} // mutex destravado aqui, mesmo se *guard panic

// Conexão de banco (sqlx, diesel, redis)
{
    let conn = pool.get().await?;
    conn.execute(query).await?;
} // conn devolvida ao pool

// Memória heap
{
    let v = vec![1, 2, 3];
} // memória liberada
```

Em cada caso, o recurso vive *exatamente* enquanto a variável vive. Não existe a categoria de bug "esqueci de fechar".

### 41.4.2 Comparação: defer, finally, try-with-resources, RAII

| Linguagem | Mecanismo | Quem garante? | Pitfalls |
|---|---|---|---|
| **C** | `free()` manual | programador | use-after-free, double-free, leak |
| **Go** | `defer` | programador | esquecer o defer, ordem invertida, defer dentro de loop |
| **TS/JS** | `try { } finally { }` | programador | esquecer o finally, async sem await |
| **Java** | `try-with-resources` + `AutoCloseable` | linguagem (limitado) | só funciona com `AutoCloseable`, sintaxe especial |
| **C++** | RAII via destrutor | linguagem | precisa lembrar que existe |
| **Rust** | RAII via `Drop` | linguagem, **default** | praticamente nenhum |

Vamos ver o mesmo programa em quatro linguagens.

```c
// C — três caminhos de saída, três frees
int processar(const char* path) {
    FILE* f = fopen(path, "r");
    if (!f) return -1;
    char* buf = malloc(1024);
    if (!buf) { fclose(f); return -2; }
    if (fgets(buf, 1024, f) == NULL) {
        free(buf);
        fclose(f);
        return -3;
    }
    int r = parse(buf);
    free(buf);
    fclose(f);
    return r;
}
```

```go
// Go — defer, mas você precisa lembrar
func processar(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close() // se você esqueceu, vazou

    buf, err := io.ReadAll(f)
    if err != nil { return err }
    return parse(buf)
}
```

```typescript
// TypeScript — finally, sem garantia de assincronia
async function processar(path: string): Promise<number> {
    const f = await fs.open(path, 'r');
    try {
        const buf = await f.readFile();
        return parse(buf);
    } finally {
        await f.close(); // se você esqueceu o try/finally, vazou
    }
}
```

```java
// Java — try-with-resources, recurso precisa implementar AutoCloseable
int processar(String path) throws IOException {
    try (var f = new BufferedReader(new FileReader(path))) {
        var line = f.readLine();
        return parse(line);
    } // f.close() chamado aqui, automático
}
```

```rust
// Rust — RAII implícito, indistinguível de "não fazer nada"
fn processar(path: &str) -> Result<i32, Erro> {
    let mut f = std::fs::File::open(path)?;
    let mut buf = String::new();
    f.read_to_string(&mut buf)?;
    parse(&buf)
} // f fechado aqui, garantido pelo compilador
```

A versão Rust é **a mais curta** e **a mais segura**. Não há `defer`, não há `finally`, não há `try-with-resources` — o destruidor é parte do tipo, não da estrutura de controle.

### 41.4.3 Por Que RAII É Melhor Que `defer`

`defer` é uma feature elegante quando vista isolada. Comparada a RAII, ela tem três fraquezas:

1. **É decisão do chamador, não do tipo.** Se você esquece de escrever `defer f.Close()`, o compilador Go não reclama. Em Rust, *é impossível* esquecer — o destruidor está atrelado ao tipo.

2. **`defer` em loop é uma armadilha clássica.**

```go
for _, path := range arquivos {
    f, _ := os.Open(path)
    defer f.Close() // só roda no fim da função!
    // ...processar...
} // se há 10000 arquivos, há 10000 file descriptors abertos
```

   O `defer` em Go acumula até o fim da *função*, não do bloco. Em Rust, sair do escopo do `for` chama `drop` imediatamente — sem armadilhas.

3. **`defer` não compõe.** Se um struct contém três campos com `Close()`, é responsabilidade do construtor lembrar de cada um. RAII compõe gratuitamente: destruir um struct destrói cada campo na ordem inversa de declaração, recursivamente.

```rust
struct Servico {
    db: ConexaoBanco,    // tem Drop
    cache: ClienteRedis, // tem Drop
    log: ArquivoLog,     // tem Drop
}
// Drop de Servico é gerado pelo compilador.
// Cada campo é destruído na ordem inversa.
// Você não escreve nada.
```

Em Go, você precisaria de uma função `Close()` no `Servico` que chama `Close()` em cada campo, e `defer servico.Close()` em todo lugar onde se cria um. Você vai esquecer.

## 41.5 Quando Combinar os Três

Os três padrões se reforçam:

```rust
// Newtype para domínio
pub struct DatabaseUrl(String);

impl DatabaseUrl {
    pub fn parse(raw: &str) -> Result<Self, UrlError> { /* ... */ }
}

// Builder para configuração
#[derive(Builder)]
pub struct PoolConfig {
    url: DatabaseUrl,           // newtype
    #[builder(default = "10")]
    max_connections: u32,
    #[builder(default = "std::time::Duration::from_secs(30)")]
    timeout: std::time::Duration,
}

// RAII para o recurso
pub struct Pool {
    inner: Arc<InnerPool>,
}

impl Drop for Pool {
    fn drop(&mut self) {
        // fecha conexões, libera handles
    }
}

impl Pool {
    pub fn new(config: PoolConfig) -> Result<Self, PoolError> { /* ... */ }
    pub fn acquire(&self) -> impl Future<Output = Result<Connection, PoolError>> { /* ... */ }
}
```

A função que recebe um `DatabaseUrl` sabe que a URL é válida (newtype). O builder garante que campos opcionais têm padrões e obrigatórios são checados (builder). O pool fecha conexões automaticamente quando sai de escopo (RAII). Cada padrão fecha uma classe de bug.

## 41.6 O Espírito da Coisa

Os três padrões deste capítulo têm um irmão filosófico: **deixar o tipo carregar a invariante**. Em vez de validar um e-mail toda vez, encapsule a validação num tipo `Email` que só pode ser construído por `parse`. Em vez de lembrar de chamar `close`, dê ao tipo um destruidor. Em vez de aceitar parâmetros num construtor de seis argumentos, faça o tipo construir-se passo a passo via builder.

A pergunta-guia: *o que estou pedindo ao programador para lembrar?* Toda resposta a essa pergunta é uma oportunidade de mover a obrigação para o compilador. Rust não inventou nenhum desses padrões. Apenas os tornou tão baratos que não usá-los virou exceção, não regra.

> *"Make illegal states unrepresentable."*
> — Yaron Minsky

[Próximo: Capítulo 42 — Type-State Pattern: Estados Como Tipos →](ch42-type-state.md)
