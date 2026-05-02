<a id="capitulo-25"></a>
# Capítulo 25: Traits Cotidianos — Display, Debug, Clone, Copy, Drop

> *"Aprenda dez traits da std e você escreve metade do Rust idiomático."*

> *"`#[derive(Debug, Clone, PartialEq)]` é a versão Rust de `public class` em Java — tão comum que parece sintaxe."*

## 25.1 A Família de Traits que Aparece em Todo Código

Todo programa Rust não-trivial usa algumas traits compulsivamente. Conhecê-las profundamente economiza horas de leitura de documentação. Este capítulo é um catálogo prático com a justificativa por trás de cada uma.

A boa notícia: a maioria pode ser **derivada** automaticamente com `#[derive(...)]`. O compilador escreve a implementação por você, eliminando o boilerplate que em Java/C++ ocuparia dezenas de linhas.

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Usuario {
    id: u64,
    nome: String,
    ativo: bool,
}
```

Esse `#[derive]` gera, em compile-time, implementações corretas de seis traits. Em Java, isso seria `equals`, `hashCode`, `toString`, copy constructor — facilmente 100 linhas. Em Rust, uma linha.

## 25.2 Display — Saída para o Usuário

`Display` produz a representação textual destinada a humanos. Usada com `{}` em `println!`, `format!`, `write!`:

```rust
use std::fmt;

struct Preco { centavos: u64 }

impl fmt::Display for Preco {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let reais = self.centavos / 100;
        let cent  = self.centavos % 100;
        write!(f, "R$ {},{:02}", reais, cent)
    }
}

fn main() {
    let p = Preco { centavos: 1599 };
    println!("{}", p); // R$ 15,99
}
```

Ponto crítico: **`Display` não pode ser derivada**. O compilador não tem como adivinhar como você quer formatar — formato é decisão semântica. Você sempre escreve manualmente.

Comparação:

```typescript
// TypeScript
class Preco {
  constructor(private centavos: number) {}
  toString(): string {
    const r = Math.floor(this.centavos / 100);
    const c = this.centavos % 100;
    return `R$ ${r},${c.toString().padStart(2, "0")}`;
  }
}
```

```go
// Go
type Preco struct{ Centavos uint64 }
func (p Preco) String() string {
    return fmt.Sprintf("R$ %d,%02d", p.Centavos/100, p.Centavos%100)
}
```

Em Go, implementar `fmt.Stringer` é estrutural — basta ter o método. Em Rust, é nominal — você declara `impl Display`. A vantagem é que o compilador checa que a assinatura está exata.

## 25.3 Debug — Saída para o Programador

`Debug` é para inspeção. Com `{:?}` para uma linha, `{:#?}` para pretty-print:

```rust
#[derive(Debug)]
struct Pedido {
    id: u32,
    itens: Vec<String>,
}

fn main() {
    let p = Pedido { id: 7, itens: vec!["livro".into(), "café".into()] };
    println!("{:?}",  p); // Pedido { id: 7, itens: ["livro", "café"] }
    println!("{:#?}", p);
    // Pedido {
    //     id: 7,
    //     itens: [
    //         "livro",
    //         "café",
    //     ],
    // }
}
```

`Debug` quase sempre é derivada. A regra implícita da comunidade: **toda struct/enum pública deve implementar Debug**. Custa nada, ajuda muito em logs e pânicos.

`assert_eq!` e amigos usam `Debug` para mostrar valores no fail. Sem `Debug`, mensagens de erro de teste viram inúteis.

## 25.4 PartialEq vs Eq — Por Que Dois?

`PartialEq` define `==` e `!=`. `Eq` é uma trait marcadora *sem métodos* que afirma a equivalência ser **total**.

Equivalência total significa: reflexiva (`a == a`), simétrica (`a == b → b == a`), transitiva (`a == b ∧ b == c → a == c`).

Por que isso importa? `f64` viola. `NaN != NaN`. Não é reflexiva. Por isso:

```rust
impl PartialEq for f64 { ... }   // existe
impl Eq for f64                  // NÃO existe
```

Se você tentar usar `f64` como chave de `HashMap`, falha — `HashMap<K, V>` exige `K: Eq + Hash`. O sistema de tipos te impede de fazer algo que silenciosamente quebraria em runtime (chave de hash com NaN é veneno).

Para tipos que não envolvem floats, derive ambas:

```rust
#[derive(PartialEq, Eq)]
struct Id(u64);
```

Em Java, `equals` é uma função única. Não há distinção entre parcial e total. Resultado: `Float.NaN.equals(Float.NaN)` retorna `true` na JVM (violando IEEE 754) para "consistência com hashCode". Rust escolheu honestidade matemática em vez de conveniência. Em troca, você precisa pensar.

## 25.5 PartialOrd vs Ord — Mesmo Padrão

`PartialOrd` define `<`, `<=`, `>`, `>=`. `Ord` afirma ordem total — todo par de elementos é comparável.

`f64` tem `PartialOrd` mas não `Ord`. `NaN` não é menor, igual, nem maior que nada. Comparações com `NaN` retornam `None` em `partial_cmp`.

```rust
let a: f64 = 1.0;
let b: f64 = f64::NAN;

a.partial_cmp(&b);  // None — não comparáveis
// a.cmp(&b);       // não compila — f64 não é Ord
```

Para ordenar `Vec<f64>`, você precisa decidir o que fazer com `NaN`:

```rust
v.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
// ou usar f64::total_cmp (estabilizado), que define ordem total artificial
v.sort_by(|a, b| a.total_cmp(b));
```

Tipos inteiros, strings, enums simples têm `Ord` e podem ser ordenados direto:

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Versao { major: u32, minor: u32, patch: u32 }
// O derive ordena lexicograficamente: major, depois minor, depois patch.
```

A ordem dos campos importa no derive. Inverta os campos, inverta a ordem do sort.

## 25.6 Hash — Para Coleções com Lookup

`Hash` permite usar o tipo como chave de `HashMap` ou elemento de `HashSet`. Quase sempre derivada:

```rust
use std::collections::HashMap;

#[derive(PartialEq, Eq, Hash)]
struct Cpf(String);

fn main() {
    let mut m: HashMap<Cpf, String> = HashMap::new();
    m.insert(Cpf("123".into()), "Felipe".into());
}
```

Contrato implícito: **se `a == b`, então `hash(a) == hash(b)`**. O derive respeita automaticamente. Se você implementar manualmente um e não o outro, e quebrar o contrato, `HashMap` corrompe silenciosamente.

Em Java, é o mesmo contrato (`equals`/`hashCode`), e a fonte de uma das categorias mais comuns de bug em código corporativo. Em Rust, derive os dois juntos sempre que possível e evite a tentação.

## 25.7 Clone — Cópia Explícita

`Clone` define cópia profunda. Sempre explícita — você chama `.clone()`:

```rust
#[derive(Clone)]
struct Doc { titulo: String, paginas: Vec<String> }

fn main() {
    let a = Doc { titulo: "X".into(), paginas: vec!["p1".into()] };
    let b = a.clone();          // duplicação profunda
    println!("{}", a.titulo);   // a continua válido (não foi movido)
}
```

Cloning em Rust é uma decisão consciente. Custou aloc no heap? Você sabe — está escrito ali. Compare TS:

```typescript
// TypeScript: cópia... rasa? profunda? você precisa saber.
const a = { titulo: "X", paginas: ["p1"] };
const b = { ...a };               // shallow — paginas é compartilhado
const c = structuredClone(a);     // deep, ES2022
```

Em TS é fácil errar e compartilhar referência sem querer. Em Rust, `clone()` é deep, `&` é referência, e o compilador não te deixa confundir.

## 25.8 Copy — Cópia Implícita por Bit-Copy

`Copy` é uma trait marcadora que diz "duplicar este valor é apenas copiar bits". Tipos `Copy` se duplicam em atribuição, sem `move`:

```rust
let x: i32 = 5;
let y = x;     // Copy: x continua válido
println!("{}", x);

let s: String = String::from("hi");
let t = s;     // Move: s não vale mais
// println!("{}", s); // erro
```

Quem é `Copy`?

- Tipos primitivos: `i32`, `f64`, `bool`, `char`, `usize`, ...
- Tuplas/arrays *só* se todos os elementos forem Copy.
- Imutabilidade em referências (`&T` é Copy mesmo se `T` não for).
- Structs onde todos os campos são Copy E você derivou `Copy`.

Quem **não pode** ser Copy?

- Qualquer coisa que possua heap: `String`, `Vec`, `Box`, `HashMap`.
- Qualquer coisa que implemente `Drop` (esses dois são mutuamente exclusivos).

Por quê? Se algo aloca recurso (`String` aloca buffer), bit-copy criaria dois donos do mesmo buffer. Quando ambos saíssem de escopo, o buffer seria liberado duas vezes — double free, o pesadelo de C. `Copy` é proibido para esses tipos por construção.

```rust
#[derive(Clone, Copy)]
struct Ponto { x: f64, y: f64 }
// f64 é Copy, então Ponto pode ser Copy.

#[derive(Clone)]            // não derivamos Copy
struct Doc { titulo: String }
// String não é Copy, então Doc não pode ser Copy.
```

Regra de polegar: derive `Copy` para tipos pequenos value-like (point, color, id numérico). Para qualquer coisa com heap, só `Clone`.

## 25.9 Default — Valor Padrão Sensato

`Default::default()` produz "o valor zero" do tipo. Derivada para tipos onde todos os campos têm default:

```rust
#[derive(Default, Debug)]
struct Config {
    porta: u16,         // 0
    host: String,       // ""
    timeout_ms: u32,    // 0
}

fn main() {
    let c: Config = Default::default();
    // ou com syntax field update:
    let c2 = Config { porta: 8080, ..Default::default() };
    println!("{:?}", c2);
}
```

`Default` é a base do "builder pattern" minimalista de Rust: structs com many fields, todos com default, e você só sobrescreve o que precisa via `..Default::default()`.

## 25.10 Drop — Destrutor RAII

`Drop` é o destrutor. Roda automaticamente quando o valor sai de escopo. **Você não chama manualmente** — o compilador chama por você:

```rust
struct ConexaoDB { id: u32 }

impl Drop for ConexaoDB {
    fn drop(&mut self) {
        println!("fechando conexão {}", self.id);
        // ... cleanup real ...
    }
}

fn main() {
    let _c = ConexaoDB { id: 42 };
    println!("trabalhando");
} // ← aqui drop(_c) é chamado automaticamente
// imprime: trabalhando, depois: fechando conexão 42
```

Esse padrão é **RAII** (Resource Acquisition Is Initialization), herdado de C++. O recurso é amarrado ao tempo de vida da variável. Se a variável existe, o recurso existe. Se a variável saiu de escopo, o recurso foi liberado. Sem chance de esquecer `close()`. Sem chance de double-close.

Você não pode chamar `.drop()` manualmente — o compilador te impede:

```rust
let c = ConexaoDB { id: 1 };
c.drop();  // ❌ erro: explicit destructor calls not allowed
drop(c);   // ✅ ok — função `drop` da std consome o valor
```

A função livre `std::mem::drop` simplesmente toma ownership e deixa o valor sair de escopo, disparando o destrutor.

Comparação:

| Linguagem | Liberação de recursos |
|---|---|
| C | `free()` manual; `fopen`/`fclose` manual |
| C++ | RAII via destructor |
| Java | `try-with-resources` (verbose); GC para memória |
| Go | `defer` explícito; GC para memória |
| TypeScript | `using` (TC39 stage 3, ainda raro); GC |
| **Rust** | **RAII automático via `Drop`** |

Rust pegou a melhor parte do C++ (RAII) e descartou a pior (manual memory management).

## 25.11 From / Into — Conversão Idiomática

`From<T>` define como converter `T` em `Self`. `Into<T>` é o inverso. **Você implementa `From`; `Into` é blanket-implementado pra você** automaticamente:

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn main() {
    let c = Celsius(100.0);
    let f: Fahrenheit = c.into();      // usa Into derivada de From
    let f2 = Fahrenheit::from(Celsius(0.0));
    println!("{} {}", f.0, f2.0);
}
```

Você consegue *dois* sentidos a partir de uma única `impl From`. Padrão recomendado: implemente `From`, não `Into`.

`From` também é base de `?` em conversão de erros:

```rust
fn ler_numero(path: &str) -> Result<i32, MeuErro> {
    let s = std::fs::read_to_string(path)?;  // io::Error → MeuErro via From
    let n = s.trim().parse::<i32>()?;        // ParseIntError → MeuErro via From
    Ok(n)
}
```

Cada `?` faz `From::from` do erro original para `MeuErro`. Implementar `From<io::Error> for MeuErro` e `From<ParseIntError> for MeuErro` torna toda essa cadeia possível. Esse é um dos padrões mais ergonômicos da linguagem.

## 25.12 Iterator — A Trait Fundadora da Biblioteca de Coleções

`Iterator` define iteração externa preguiçosa. Apenas um método obrigatório:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // dezenas de métodos default: map, filter, collect, fold, sum, ...
}
```

Implementando `next`, você ganha de graça:

```rust
struct Contador { atual: u32, max: u32 }

impl Iterator for Contador {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.atual < self.max {
            self.atual += 1;
            Some(self.atual)
        } else {
            None
        }
    }
}

fn main() {
    let total: u32 = Contador { atual: 0, max: 1_000_000 }
        .filter(|n| n % 7 == 0)
        .map(|n| n * n)
        .sum();
    println!("{}", total);
}
```

Iteradores são **lazy** — `filter` e `map` não executam até `sum` puxar. E o compilador inlinha tudo: o código gerado é equivalente a um `for` loop em C, sem alocação intermediária. Capítulo dedicado virá adiante; aqui o foco é só reconhecer `Iterator` como a trait que dá poder a `for x in coisa`.

## 25.13 A Tabela Mestre dos Derives

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
```

Esses são os nove principais. Quando aplicar cada um:

| Derive | Use quando | Não use quando |
|---|---|---|
| `Debug` | sempre que possível em tipos públicos | nunca — derive sempre |
| `Clone` | tipo precisa duplicação explícita | tipo é único por construção (handles, sockets) |
| `Copy` | tipo é pequeno, value-like, sem heap | tem `String`, `Vec`, `Box`, ou implementa `Drop` |
| `PartialEq` | precisa comparar com `==` | comparação não faz sentido (closures, tipos com FP especial) |
| `Eq` | `PartialEq` é também total | tem floats |
| `PartialOrd` | precisa ordenar | ordenação não faz sentido |
| `Ord` | `PartialOrd` é total | tem floats |
| `Hash` | precisa usar como chave de HashMap/HashSet | tem floats; tem campos com semântica especial |
| `Default` | "valor zero" tem sentido | construir o tipo exige escolha de negócio |

## 25.14 Quanto Boilerplate Você Está Economizando

Em Java, a mesma struct exige construtor, getters, e overrides manuais de `equals`, `hashCode` e `toString` — facilmente quarenta linhas. Em Rust:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Usuario { id: u64, nome: String, ativo: bool }
```

Quatro linhas. Lombok aliviou em Java; records (14+) aliviaram mais. Mas em Rust cada derive é declarativo, escolhido individualmente, e o compilador gera código otimizado direto — sem delegar a annotation processors em runtime.

## 25.15 Resumo

- `Display` e `Debug` para formatação humana e para programador.
- `PartialEq`/`Eq`, `PartialOrd`/`Ord` — a versão "parcial" tolera valores não comparáveis (NaN); a "total" não.
- `Hash` para chaves de HashMap; sempre junto de `Eq`.
- `Clone` para cópia explícita profunda; `Copy` para bit-copy implícita.
- `Default` para valor zero sensato.
- `Drop` para RAII — destrutor automático, nunca chamado manualmente.
- `From`/`Into` para conversão idiomática; implemente `From`, ganhe `Into`.
- `Iterator` para iteração externa preguiçosa; um `next` te dá uma biblioteca.
- `#[derive(...)]` é a arma mais subestimada da linguagem.

Com este capítulo, você fechou os fundamentos do sistema de tipos de Rust. Os próximos capítulos vão para concorrência segura, async/await, e o ecossistema — todos eles construídos *em cima* das ideias dos capítulos 22 a 25.

---

> *"`#[derive]` é um agradecimento silencioso do Rust pelo tempo da sua vida."*

[Próximo: Capítulo 26 →](ch26-erros-resultado-opcao.md)
