<a id="capitulo-44"></a>
# Capítulo 44: serde — Serialização Como Arte

> *"Make the boundaries of your program well-typed."*
> — Alexis King, *Parse, Don't Validate*

> *"Serde is what convinced me Rust was serious."*
> — Engenheiro anônimo da Cloudflare, em algum corredor de RustConf

## 44.1 O Problema Que Ninguém Resolveu Direito

Toda linguagem precisa cruzar o abismo entre **bytes na rede** e **valores na memória**. JSON entra, struct sai. YAML entra, config sai. Postgres devolve linhas, código quer objetos. Esse cruzamento — **serialização** e **desserialização** — é a fronteira mais perigosa de qualquer aplicação. É lá que dados não-confiáveis viram tipos confiáveis. É lá que ataques nascem. É lá que erros silenciosos se infiltram.

A maior parte das linguagens trata essa fronteira como uma rampa de skate enferrujada. Você pula, fecha os olhos, espera não cair.

```typescript
// TypeScript — a fronteira invisível
const user = JSON.parse(rawJson);
// user é `any`. Não tem tipo. Pode ter qualquer coisa.
// Pode estar faltando email. Pode ter SQL injection no nome.
// O compilador nada sabe. Você descobre em produção.
console.log(user.email.toLowerCase()); // crash se email não existe
```

```go
// Go — reflection em runtime
type User struct {
    Email string `json:"email"`
    Name  string `json:"name,omitempty"`
}

var u User
err := json.Unmarshal(data, &u) // reflection, runtime cost
// Campo desconhecido? Ignorado silenciosamente.
// Tipo errado? Erro genérico em runtime.
```

```java
// Java/Jackson — verboso, anotações por toda parte
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonIgnoreProperties(ignoreUnknown = true)
public class User {
    @JsonProperty("email_address")
    private String email;
    // 40 linhas de getter/setter/anotação para uma estrutura simples.
}
```

Cada uma dessas soluções tem o mesmo defeito: **a serialização e a deserialização vivem em runtime**. O custo é pago em cada parse. O erro é detectado em cada parse. A correção é gambiarra em cada parse.

Rust olhou para isso e perguntou: *e se a serialização fosse uma propriedade do tipo?* E se o compilador gerasse o código de parsing **junto com o tipo**, sem reflection, sem runtime, sem custo? E se um único `derive` funcionasse para JSON, YAML, TOML, MessagePack, CBOR, e qualquer formato que ainda fosse inventado?

A resposta foi `serde` — e é amplamente considerada a **melhor biblioteca de serialização do mundo**, em qualquer linguagem.

## 44.2 A Arquitetura: Tipos Falam, Formatos Ouvem

A genialidade de `serde` está na separação radical entre **estrutura** e **formato**.

```mermaid
graph LR
    Struct["Seu struct<br/>#[derive(Serialize)]"] --> Trait[Trait Serialize]
    Trait --> JSON[serde_json]
    Trait --> YAML[serde_yaml]
    Trait --> TOML[toml]
    Trait --> Bincode[bincode]
    Trait --> MsgPack[rmp-serde]

    style Trait fill:#c8e6c9,stroke:#1b5e20
    style Struct fill:#bbdefb,stroke:#0d47a1
</content>
```

O struct não sabe que existe JSON. Ele só sabe se descrever a um *Serializer*. O *Serializer* não sabe que existe um struct específico — ele só sabe receber descrições e produzir bytes. **Os dois lados se encontram via traits, em tempo de compilação, sem reflection.**

O resultado é o mesmo struct, o mesmo `derive`, e *qualquer formato*:

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Config {
    host: String,
    port: u16,
    timeout_ms: u64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cfg = Config {
        host: "127.0.0.1".into(),
        port: 8080,
        timeout_ms: 5000,
    };

    // Mesmo struct, três formatos. Zero linhas extras.
    let json = serde_json::to_string(&cfg)?;
    let yaml = serde_yaml::to_string(&cfg)?;
    let toml = toml::to_string(&cfg)?;
    let bin: Vec<u8> = bincode::serialize(&cfg)?;

    Ok(())
}
```

Em Go, você precisaria de tags JSON, tags YAML, tags TOML — cada uma com sua sintaxe. Em Java, anotações Jackson + Snake YAML + outra coisa para TOML. Em Rust, **um derive serve a todos**. Esse é o dividendo da arquitetura: o trait `Serialize` é genérico sobre o destino, e o trait `Deserialize` é genérico sobre a fonte.

## 44.3 Os Atributos: Linguagem Dentro da Linguagem

`serde` exporta uma micro-linguagem declarativa via atributos. Você nunca escreve um parser — você *anota* o tipo, e o derive faz o trabalho.

### 44.3.1 `#[serde(rename = "...")]`

A API JSON externa usa `snake_case` ou `camelCase`. O Rust idiomático é `snake_case` em campos, mas `PascalCase` em variantes. Sem `rename`, você teria que sacrificar o estilo.

```rust
#[derive(Serialize, Deserialize)]
struct ApiResponse {
    #[serde(rename = "userId")]
    user_id: u64,
    #[serde(rename = "createdAt")]
    created_at: String,
}

// Para o caso comum, renomeie tudo de uma vez:
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct ApiResponseV2 {
    user_id: u64,    // vira "userId" automaticamente
    created_at: String, // vira "createdAt"
}
```

`rename_all` aceita `"camelCase"`, `"snake_case"`, `"kebab-case"`, `"SCREAMING_SNAKE_CASE"`, `"PascalCase"`, `"lowercase"`, `"UPPERCASE"`. Cobre virtualmente toda convenção de API que você vai encontrar.

### 44.3.2 `#[serde(default)]`

Campo ausente no JSON? Em vez de erro, use o `Default::default()` do tipo.

```rust
#[derive(Serialize, Deserialize)]
struct ServerConfig {
    host: String,
    #[serde(default = "default_port")]
    port: u16,
    #[serde(default)] // usa u64::default() = 0
    retries: u64,
}

fn default_port() -> u16 { 8080 }

// JSON: {"host": "localhost"} → port=8080, retries=0
```

Compare com Go: você teria que checar `if u.Port == 0 { u.Port = 8080 }` em cada handler. Com `serde`, o default é parte do *contrato do tipo*.

### 44.3.3 `#[serde(skip)]`, `#[serde(skip_serializing)]`, `#[serde(skip_deserializing)]`

Nem todo campo deve atravessar a fronteira. Senha, sessão interna, cache — esconda do `Serialize`. ID gerado pelo banco — esconda do `Deserialize`.

```rust
#[derive(Serialize, Deserialize)]
struct User {
    id: u64,
    email: String,

    #[serde(skip_serializing)] // nunca sai pelo wire
    password_hash: String,

    #[serde(skip_deserializing)] // nunca chega pelo wire
    last_seen: Option<DateTime<Utc>>,

    #[serde(skip)] // nem entra, nem sai. Cache local.
    cached_avatar: Option<Vec<u8>>,
}
```

Em TypeScript, esconder campo é uma convenção (underscore, tipo separado para resposta). Em `serde`, é um *contrato verificado pelo compilador*. Se você esquecer de marcar o `password_hash` como `skip_serializing` e tentar usá-lo em uma resposta, a divergência fica óbvia no diff.

### 44.3.4 `#[serde(flatten)]`

Quando o JSON é "achatado" mas o seu domínio é hierárquico, `flatten` reconcilia.

```rust
#[derive(Serialize, Deserialize)]
struct Pagination {
    page: u32,
    per_page: u32,
}

#[derive(Serialize, Deserialize)]
struct Listing<T> {
    #[serde(flatten)]
    pagination: Pagination,
    items: Vec<T>,
}

// Serializa como:
// {"page": 1, "per_page": 20, "items": [...]}
// e não como:
// {"pagination": {"page": 1, "per_page": 20}, "items": [...]}
```

`flatten` também captura **campos extras** em um `HashMap`:

```rust
use std::collections::HashMap;
use serde_json::Value;

#[derive(Serialize, Deserialize)]
struct Event {
    event_type: String,
    timestamp: u64,
    #[serde(flatten)]
    extra: HashMap<String, Value>, // tudo que não casou com os outros campos
}
```

Útil para webhooks de terceiros que adicionam campos novos sem aviso. Você não falha — você captura.

### 44.3.5 `#[serde(tag = "...")]`: Discriminated Unions de Verdade

Esta é a arma secreta. Em quase toda API de evento (webhooks Stripe, GitHub, Slack), o JSON traz um campo `type` que diz qual variante é qual:

```json
{"type": "payment", "amount": 100, "currency": "BRL"}
{"type": "refund", "amount": 100, "reason": "duplicate"}
```

Em TypeScript, você escreve uma união discriminada manual e um *type guard* para cada variante. Em Go, você desserializa para `map[string]interface{}`, lê o `"type"`, e *re-serializa* para o tipo correto. Em `serde`, é uma anotação:

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type", rename_all = "lowercase")]
enum WebhookEvent {
    Payment { amount: u64, currency: String },
    Refund { amount: u64, reason: String },
    Chargeback { amount: u64, dispute_id: String },
}

fn handle(event: WebhookEvent) {
    match event {
        WebhookEvent::Payment { amount, currency } => process_payment(amount, &currency),
        WebhookEvent::Refund { amount, reason } => process_refund(amount, &reason),
        WebhookEvent::Chargeback { amount, dispute_id } => dispute(amount, &dispute_id),
    }
    // O compilador exige exhaustividade. Adicionou variante nova? Quebra a build.
}
```

Variações:

- `#[serde(tag = "type", content = "data")]` — o tag fica num campo, o payload em outro: `{"type": "payment", "data": {...}}`.
- `#[serde(untagged)]` — sem tag explícito, `serde` tenta cada variante até uma casar (ordem importa, é mais lento).

Compare com TypeScript:

```typescript
type WebhookEvent =
    | { type: "payment"; amount: number; currency: string }
    | { type: "refund"; amount: number; reason: string };

function handle(event: WebhookEvent) {
    switch (event.type) {
        case "payment": /* ... */ break;
        case "refund": /* ... */ break;
        // sem `default: never`, esquecer uma variante passa silenciosamente
    }
}
```

TS faz o trabalho — *se* você escrever o código com cuidado. Rust *exige* exhaustividade no `match`. Adicione `Chargeback` no enum, e cada handler que falhar em tratá-lo deixa de compilar. **Quebra a build, não a produção.**

### 44.3.6 `#[serde(with = "...")]`

Quando o formato externo não bate com o tipo interno, mas você não quer renunciar ao tipo. Datas são o exemplo canônico: o wire usa string ISO 8601, mas você quer `chrono::DateTime<Utc>`.

```rust
use chrono::{DateTime, Utc};

#[derive(Serialize, Deserialize)]
struct LogEntry {
    message: String,
    #[serde(with = "chrono::serde::ts_seconds")]
    timestamp: DateTime<Utc>,
}

// JSON: {"message": "ok", "timestamp": 1700000000}
// Tipo: DateTime<Utc> de verdade, com todas as operações de chrono.
```

Você também pode escrever seu próprio módulo `with`:

```rust
mod hex_bytes {
    use serde::{Deserializer, Serializer, Deserialize};

    pub fn serialize<S: Serializer>(bytes: &[u8], s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&hex::encode(bytes))
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<Vec<u8>, D::Error> {
        let s = String::deserialize(d)?;
        hex::decode(&s).map_err(serde::de::Error::custom)
    }
}

#[derive(Serialize, Deserialize)]
struct Block {
    #[serde(with = "hex_bytes")]
    hash: Vec<u8>,
}
```

`with` é a porta de escape sem custo: o tipo interno permanece preciso, a representação externa permanece padrão.

## 44.4 Quando Derive Não Basta: Impl Manual

Para 95% dos casos, `derive` resolve. Para os 5% restantes — quando a estrutura externa simplesmente não tem mapeamento direto para a interna — você implementa `Serialize`/`Deserialize` à mão. O contrato é claro:

```rust
use serde::{Serializer, Serialize};
use serde::ser::SerializeStruct;

struct Money {
    cents: i64,
}

impl Serialize for Money {
    fn serialize<S: Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        // Externamente: string formatada em reais.
        s.serialize_str(&format!("R$ {:.2}", self.cents as f64 / 100.0))
    }
}
```

Para `Deserialize` o ritual é maior — você implementa um `Visitor` que descreve cada formato aceito. É verboso, mas é a *única* parte da experiência `serde` que custa concentração. E só acontece nos cantos exóticos.

## 44.5 Zero-Copy Deserialization: A Vitória Invisível

Quando o input é `&str` ou `&[u8]` que dura mais que o resultado, `serde` *não copia* as strings. Ele desserializa para `&str` que aponta para dentro do buffer original.

```rust
#[derive(Deserialize)]
struct LogLine<'a> {
    level: &'a str,
    message: &'a str,
}

let raw = r#"{"level":"info","message":"server started"}"#;
let line: LogLine = serde_json::from_str(raw)?;
// line.level e line.message são fatias do `raw` original.
// Zero alocação. Zero cópia. Borrow checker garante que `raw` não morre antes.
```

Em Go, `json.Unmarshal` aloca uma `string` nova para cada campo string (porque strings em Go são imutáveis e ownership é por GC). Em Java, idem. **Em Rust, parsing pode ser literalmente zero-copy** — e o borrow checker prova que é seguro.

Para um log shipper processando milhões de linhas por segundo, isso é a diferença entre 4 cores saturados e 1 core ocioso.

## 44.6 Erros: Localizados, Tipados, Acionáveis

Quando o parsing falha, `serde_json` te diz exatamente onde:

```rust
let raw = r#"{"port": "not_a_number"}"#;
let cfg: Result<ServerConfig, _> = serde_json::from_str(raw);

// Err(Error("invalid type: string \"not_a_number\", expected u16",
//          line: 1, column: 24))
```

Compare com `JSON.parse` em TypeScript: você ganha `SyntaxError: Unexpected token` se o JSON for inválido, ou *nada* se o JSON for válido mas o conteúdo for errado — o erro só aparece quando você tenta `user.port + 1` e descobre que `port` é uma string.

`serde_json::Error` carrega linha, coluna, e descrição da expectativa. Erros não são uma ideia tardia — são um produto do tipo.

## 44.7 O Custo: Tempo de Compilação

Não há almoço grátis. `serde` tem um custo, e ele é honesto: **proc macros são lentas para compilar**. Um projeto com 200 structs `Serialize, Deserialize` paga alguns segundos a cada `cargo build`. Em projetos enormes (Cargo, rust-analyzer), isso vira minutos.

A comunidade tem soluções:

- `miniserde` — derive sem proc macro, compila rápido, mas só fala JSON e tem menos atributos.
- `serde-generate` — gera código uma vez, comita, sem proc macro em rebuild.
- `nanoserde` — mesma filosofia, ainda menor.

Para a maioria dos projetos, o custo de compilação de `serde` é aceitável. Para a totalidade dos projetos, o custo de runtime de `serde` é zero.

## 44.8 O Veredito

Toda outra linguagem tem uma biblioteca de serialização. Nenhuma tem o que `serde` tem:

| Critério | TS `JSON.parse` | Go `encoding/json` | Java Jackson | Rust `serde` |
|---|---|---|---|---|
| Type-safe | Não (any) | Parcial (struct tags) | Sim, com anotações | Sim, no compilador |
| Reflection runtime | N/A | Sim | Sim | **Não** |
| Zero-copy | Não | Não | Não | **Sim** |
| Formato-agnóstico | Só JSON | Só JSON nativo | Vários, mas anotações por formato | **Um derive, qualquer formato** |
| Discriminated unions | Manual | Manual | `@JsonTypeInfo` | `#[serde(tag = "...")]` |
| Erros localizados | Não | Genéricos | Bons | **Linha + coluna + tipo esperado** |
| Custo de compilação | Zero | Zero | Médio | Médio-alto (proc macro) |
| Custo de runtime | Médio | Médio (reflection) | Médio | **Zero** |

`serde` não é apenas a melhor biblioteca de serialização do Rust. É um exemplo de que, quando você desenha uma linguagem com **traits + derive macros + lifetimes + sem GC**, alguns problemas que pareciam intrinsecamente dinâmicos se tornam *estáticos*. O parsing deixa de ser uma fronteira insegura e vira parte do tipo.

> *"A fronteira da sua aplicação não precisa ser uma rampa enferrujada. Ela pode ser um corredor com paredes."*

[Próximo: Capítulo 45 — Testing: cargo test, criterion, proptest →](ch45-testing.md)
