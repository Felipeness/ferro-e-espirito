# Árvores de Decisão

> Diagramas e regras para escolhas comuns em Rust. Quando estiver em dúvida, comece aqui.

---

## 1. `String` ou `&str`?

```mermaid
graph TB
    Q["Você está armazenando ou recebendo?"]
    Q --> A["Recebendo<br/>como parâmetro"]
    Q --> B["Armazenando<br/>em struct/Vec"]

    A --> C["Aceite &str<br/>(funciona com String<br/>via deref coercion)"]
    B --> D["A struct é dona<br/>do dado?"]

    D --> E["Sim<br/>→ String"]
    D --> F["Não, vive<br/>tanto quanto algo<br/>externo<br/>→ &'a str + lifetime"]
```

**Regra:** parâmetro = `&str`. Campo que possui = `String`. Campo que referencia = `&'a str` (mas quase sempre prefira `String` em código de aplicação).

---

## 2. `Vec<T>`, `Box<[T]>`, `&[T]`, ou `[T; N]`?

| Use | Quando |
|---|---|
| `[T; N]` | Tamanho fixo conhecido em compile time, vai pra stack |
| `Vec<T>` | Tamanho dinâmico, mutável, dono |
| `Box<[T]>` | Tamanho fixo determinado em runtime, dono, sem capacidade extra |
| `&[T]` | Visão imutável (parâmetro idiomático) |
| `&mut [T]` | Visão mutável sem mexer no tamanho |

Default: passe `&[T]` em parâmetros, retorne `Vec<T>` ou armazene `Vec<T>`. Otimize para `Box<[T]>` quando capacidade é desperdício.

---

## 3. `Box<dyn Trait>`, `impl Trait`, ou `<T: Trait>`?

```mermaid
graph TB
    Start["Você precisa de polimorfismo"]
    Start --> Q1["Em runtime você terá<br/>tipos heterogêneos<br/>na mesma coleção?"]

    Q1 --> Yes["Sim<br/>→ Box dyn Trait"]
    Q1 --> No["Não"]

    No --> Q2["É retorno de função<br/>com um único tipo?"]
    Q2 --> Y2["Sim<br/>→ impl Trait<br/>(zero overhead)"]
    Q2 --> N2["Não, é parâmetro"]

    N2 --> Q3["Você quer monomorphization<br/>(velocidade) ou tamanho<br/>de binário (dyn)?"]
    Q3 --> M["Velocidade<br/>→ T: Trait"]
    Q3 --> B["Tamanho<br/>→ dyn Trait"]
```

Regra prática: `<T: Trait>` por default. `impl Trait` em retorno. `Box<dyn Trait>` apenas quando heterogeneidade real em runtime.

---

## 4. `Rc`, `Arc`, `Box`, ou referência?

```mermaid
graph TB
    Q1["Há múltiplos donos<br/>do mesmo valor?"]
    Q1 --> One["Não"]
    Q1 --> Many["Sim"]

    One --> Q2["Você precisa<br/>de heap?"]
    Q2 --> NoHeap["Não → use referência<br/>ou valor direto"]
    Q2 --> YesHeap["Sim → Box T"]

    Many --> Q3["Vai atravessar threads?"]
    Q3 --> Single["Não → Rc T"]
    Q3 --> Multi["Sim → Arc T"]
```

Regra: prefira referências `&T`. Use `Box` para heap. `Rc` para single-thread shared. `Arc` para multi-thread shared. Acumular `Arc<Mutex<T>>` em todo lugar é red flag — repense ownership.

---

## 5. `Mutex`, `RwLock`, ou `Atomic`?

```mermaid
graph TB
    Q["O que você compartilha?"]
    Q --> A["Tipo primitivo<br/>(int, bool, ptr)"]
    Q --> B["Estrutura de dados"]

    A --> AT["Atomic*<br/>(zero-lock, mais rápido)"]
    B --> RW["Quantos escritores<br/>vs leitores?"]

    RW --> Equal["Similar<br/>→ Mutex"]
    RW --> ReadHeavy["Muito mais leitura<br/>→ RwLock"]
```

Default: `Mutex`. `RwLock` quando leitura é 10x+ mais frequente que escrita. `Atomic*` para flags e contadores. `parking_lot::Mutex` se perf importa.

---

## 6. `Result`, `Option`, ou `panic!`?

| Situação | Escolha |
|---|---|
| Operação pode falhar de forma esperada | `Result<T, E>` |
| Valor pode estar ausente sem ser erro | `Option<T>` |
| Programa em estado impossível (invariante quebrada) | `panic!` |
| Index out of bounds em valor controlado por dev | `panic!` (via `[]`) |
| Valor user-provided que pode ser inválido | `Result` |
| Default de campo opcional | `Option` |

Regra moral: `panic!` é para "bug do programador", `Result` é para "erro do mundo".

---

## 7. Síncrono ou async?

```mermaid
graph TB
    Q1["Você faz I/O<br/>em alguma chamada?"]
    Q1 --> No["Não → síncrono.<br/>Não introduza async."]
    Q1 --> Yes["Sim"]

    Yes --> Q2["Quantas operações<br/>I/O concorrentes?"]
    Q2 --> Few["Poucas (10-100)<br/>→ threads + sync"]
    Q2 --> Many["Muitas (1000+)<br/>→ async + Tokio"]
```

Regra: async tem custo cognitivo e de compile time. Use quando você *precisa* de muitas operações concorrentes I/O-bound. Para CPU-bound, threads. Para I/O moderado, threads.

---

## 8. `thiserror` ou `anyhow`?

| Use | Quando |
|---|---|
| `thiserror` | Você está escrevendo uma **library**. Outros vão querer fazer match no erro. |
| `anyhow` | Você está escrevendo uma **application**. Você só quer logar/reportar. |

Combine ambos: lib expõe erro com `thiserror`, app consome via `anyhow::Result` e propaga com `?`.

---

## 9. Newtype ou type alias?

```mermaid
graph TB
    Q["Você quer prevenir confusão<br/>entre tipos similares?"]
    Q --> Yes["Sim<br/>(UserId vs PostId)"]
    Q --> No["Não, só dar nome legível"]

    Yes --> NT["Newtype<br/>struct UserId(u64);"]
    No --> TA["Type alias<br/>type UserId = u64;"]
```

`type` é só açúcar — mesma representação, sem barreira de tipos. `struct UserId(u64)` cria barreira: você não pode passar `PostId` onde `UserId` é esperado. Quase sempre vale o boilerplate.

---

## 10. Mover, emprestar imutável, ou mutável?

| Quero | Assinatura |
|---|---|
| Que a função consuma o valor (transferindo posse) | `fn foo(x: T)` |
| Que a função leia o valor sem modificar nem consumir | `fn foo(x: &T)` |
| Que a função modifique o valor sem consumi-lo | `fn foo(x: &mut T)` |
| Trabalhar com slice em vez de coleção específica | `fn foo(x: &[T])` em vez de `&Vec<T>` |
| Aceitar `String` ou `&str` | `fn foo(x: impl AsRef<str>)` ou `&str` |

Regra: comece sempre com `&T`. Promova para `&mut T` se precisar modificar. Use `T` (move) só quando a função é a sucessora natural do dono (ex: `into_iter`, `into_string`).

---

## 11. `From` ou `TryFrom`?

```mermaid
graph TB
    Q["A conversão pode falhar?"]
    Q --> Yes["Sim → TryFrom (retorna Result)"]
    Q --> No["Não → From (infalível)"]
```

`From` automaticamente dá `Into`. Implemente `From` sempre que possível — o ecossistema espera isso.

---

## 12. `Iterator` adapter ou `for` loop?

| Use | Quando |
|---|---|
| `iter().map(...).filter(...).collect()` | Transformações funcionais, lazy, encadeadas |
| `for x in v` | Side effects, controle de fluxo complexo, early break com state |

Não há penalidade de performance — iterators são zero-cost. A escolha é estilística e de clareza.

---

[← Voltar ao sumário](../SUMMARY.md) | [Glossário ←](glossario.md) | [Cheat Sheet →](cheat-sheet.md)
