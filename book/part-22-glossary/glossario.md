<a id="glossario"></a>
# Glossário

> *"Nada em Rust é arbitrário. Mas quase tudo tem nome próprio."*

Este glossário é um índice de consulta rápida. Cada termo aparece com uma definição de uma ou duas frases e uma referência ao capítulo onde ele é tratado em profundidade. Os termos foram organizados em ordem alfabética, ignorando acentos e capitalização. Quando um termo é estrangeiro e tem tradução consagrada em português, ela aparece em parênteses.

## A

| Termo | Definição | Capítulo |
|---|---|---|
| `?` (operador) | Açúcar sintático que propaga `Err`/`None` para o chamador, retornando cedo da função. | [cap. 17](../part-07-collections-and-errors/ch17-result.md) |
| ABI | Application Binary Interface — contrato binário entre código compilado. Rust não tem ABI estável, exceto via `extern "C"`. | [cap. 36](../part-13-unsafe-and-ffi/ch36-ffi.md) |
| Aliasing | Existência de mais de um caminho para o mesmo valor. Rust proíbe aliasing mutável. | [cap. 11](../part-04-ownership/ch11-borrowing.md) |
| `alloc` | Crate da biblioteca padrão que oferece tipos heap (`Box`, `Vec`, `String`) sem exigir `std`. | [cap. 47](../part-16-ecosystem/ch47-no-std.md) |
| Anonymous lifetime (`'_`) | Lifetime que o compilador infere; usada para evitar repetir nomes em assinaturas. | [cap. 26](../part-09-lifetimes-deep/ch26-elision.md) |
| `Any` (trait) | Permite reflexão limitada em runtime via `TypeId`. Última escolha, não primeira. | [cap. 29](../part-10-smart-pointers/ch29-any.md) |
| `Arc<T>` | Atomic Reference Counted — ponteiro de posse compartilhada, thread-safe, com custo de operação atômica. | [cap. 30](../part-10-smart-pointers/ch30-rc-arc.md) |
| Arena | Estratégia de alocação onde objetos vivem o tempo de uma região e são liberados em massa. | [cap. 56](../part-17-performance/ch56-arenas.md) |
| `as` (cast) | Conversão numérica truncante e checada por tipo. Para conversões falíveis prefira `TryFrom`. | [cap. 8](../part-03-types-and-syntax/ch08-numerics.md) |
| `assert!` / `assert_eq!` | Macros de afirmação. Falham em tempo de execução se a condição não vale; usadas em testes e invariantes. | [cap. 19](../part-08-generics-and-traits/ch19-tests.md) |
| Associated function | Função associada a um tipo, sem `self`. Acessada com `Tipo::funcao`. Exemplo: `String::new`. | [cap. 14](../part-05-composite-types/ch14-impl.md) |
| Associated type | Tipo declarado dentro de um trait com `type Item;`. Permite traits genéricos sem parâmetros explícitos no usuário. | [cap. 22](../part-08-generics-and-traits/ch22-associated-types.md) |
| `async` | Palavra-chave que transforma uma função em uma que retorna `impl Future`. Não executa; descreve. | [cap. 33](../part-12-async/ch33-async.md) |
| `await` | Operador que suspende a função até o `Future` estar pronto. Só pode aparecer dentro de `async fn` ou `async {}`. | [cap. 33](../part-12-async/ch33-async.md) |
| Auto trait | Trait implementada automaticamente para tipos cujos campos a satisfazem. Exemplos: `Send`, `Sync`, `Unpin`. | [cap. 32](../part-11-concurrency/ch32-send-sync.md) |

## B

| Termo | Definição | Capítulo |
|---|---|---|
| Binding | Associação entre um nome e um valor. Em Rust, criada com `let`. | [cap. 6](../part-02-foundations/ch06-bindings.md) |
| Borrow (empréstimo) | Acesso temporário a um valor sem tomar posse, expresso com `&` ou `&mut`. | [cap. 11](../part-04-ownership/ch11-borrowing.md) |
| Borrow checker | Componente do compilador que valida regras de empréstimo. É o "antagonista" famoso do Rust. | [cap. 12](../part-04-ownership/ch12-borrow-checker.md) |
| `Box<T>` | Ponteiro inteligente que move um valor para a heap. Posse única, custo zero além do `malloc`. | [cap. 28](../part-10-smart-pointers/ch28-box.md) |
| Branded type / Newtype | `struct Foo(T)` que dá identidade ao valor sem custo, evitando primitive obsession. | [cap. 41](../part-15-patterns-and-idioms/ch41-newtype.md) |

## C

| Termo | Definição | Capítulo |
|---|---|---|
| `Cargo` | Gerenciador de pacotes e build system do Rust. Resolve dependências, compila, testa, publica. | [cap. 17](../part-06-modules-and-crates/ch17-cargo.md) |
| `Cell<T>` | Container de mutabilidade interior por cópia/troca; `Copy` ou `Default`. Não é `Sync`. | [cap. 31](../part-10-smart-pointers/ch31-cell-refcell.md) |
| Closure | Função anônima que pode capturar o ambiente. Implementa `Fn`, `FnMut` ou `FnOnce`. | [cap. 21](../part-08-generics-and-traits/ch21-closures.md) |
| Coerção | Conversão implícita feita pelo compilador, restrita a casos seguros (deref, unsizing, lifetime). | [cap. 9](../part-03-types-and-syntax/ch09-coercion.md) |
| Coherence (regra do orphan) | Restrição que impede dois crates de implementarem a mesma trait para o mesmo tipo. | [cap. 23](../part-08-generics-and-traits/ch23-coherence.md) |
| `Copy` | Marker trait para tipos que podem ser duplicados por cópia bit a bit. Implica `Clone`. | [cap. 10](../part-04-ownership/ch10-ownership.md) |
| `core` | Subconjunto de `std` que não depende de alocação nem do sistema operacional. Base do `no_std`. | [cap. 47](../part-16-ecosystem/ch47-no-std.md) |
| Crate | Unidade de compilação do Rust. Pode ser binária (`main.rs`) ou biblioteca (`lib.rs`). | [cap. 16](../part-06-modules-and-crates/ch16-crates.md) |

## D

| Termo | Definição | Capítulo |
|---|---|---|
| Dangling reference | Referência para memória já liberada. Em Rust, impedida em tempo de compilação. | [cap. 12](../part-04-ownership/ch12-borrow-checker.md) |
| Data race | Acesso concorrente onde pelo menos um é mutável e não há sincronização. Rust proíbe via `Send`/`Sync`. | [cap. 32](../part-11-concurrency/ch32-send-sync.md) |
| `Default` | Trait que fornece um valor padrão via `Tipo::default()`. Útil em construtores e `derive`. | [cap. 22](../part-08-generics-and-traits/ch22-default.md) |
| `Deref` / `DerefMut` | Traits que permitem que `&T` se comporte como `&U`. Base de smart pointers. | [cap. 28](../part-10-smart-pointers/ch28-deref.md) |
| `derive` | Atributo que pede ao compilador para gerar implementações triviais de traits comuns. | [cap. 14](../part-05-composite-types/ch14-derive.md) |
| `Drop` | Trait com método `drop(&mut self)` chamado quando o valor sai de escopo. Análogo a destrutores. | [cap. 13](../part-04-ownership/ch13-drop.md) |
| `dyn Trait` | Tipo de objeto-trait dinâmico, com vtable. Usado quando o tipo concreto não é conhecido. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |
| DST (Dynamically Sized Type) | Tipo cujo tamanho não é conhecido em tempo de compilação. Exemplos: `[T]`, `str`, `dyn Trait`. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |

## E

| Termo | Definição | Capítulo |
|---|---|---|
| Edition | Marco de versão do Rust que pode introduzir mudanças não compatíveis. Atualmente: 2015, 2018, 2021, 2024. | [cap. 17](../part-06-modules-and-crates/ch17-cargo.md) |
| Elision (de lifetimes) | Conjunto de regras que permitem omitir lifetimes em assinaturas comuns. | [cap. 26](../part-09-lifetimes-deep/ch26-elision.md) |
| `enum` | Tipo soma. Em Rust, enums são algébricos: variantes podem carregar dados. | [cap. 15](../part-05-composite-types/ch15-enums.md) |
| `Error` (trait) | Trait padrão para tipos de erro, exigindo `Display`, `Debug` e fonte opcional. | [cap. 17](../part-07-collections-and-errors/ch17-error-trait.md) |
| Exhaustive match | Verificação do compilador de que todo `match` cobre todos os casos possíveis. | [cap. 15](../part-05-composite-types/ch15-match.md) |
| `extern "C"` | Declaração de função com convenção de chamada C, base de FFI. | [cap. 36](../part-13-unsafe-and-ffi/ch36-ffi.md) |

## F

| Termo | Definição | Capítulo |
|---|---|---|
| FFI (Foreign Function Interface) | Interoperação com código de outras linguagens, geralmente via ABI C. | [cap. 36](../part-13-unsafe-and-ffi/ch36-ffi.md) |
| `Fn`, `FnMut`, `FnOnce` | Hierarquia de traits implementadas por closures conforme capturam o ambiente. | [cap. 21](../part-08-generics-and-traits/ch21-closures.md) |
| `From` / `Into` | Traits para conversão infalível entre tipos. `Into` é derivado automaticamente de `From`. | [cap. 22](../part-08-generics-and-traits/ch22-from-into.md) |
| Future | Trait que representa um valor que estará pronto eventualmente. Base do `async`. | [cap. 33](../part-12-async/ch33-future.md) |

## G

| Termo | Definição | Capítulo |
|---|---|---|
| Generics | Parametrização de tipos e funções por tipos. Resolvidos por monomorfização. | [cap. 20](../part-08-generics-and-traits/ch20-generics.md) |
| GAT (Generic Associated Type) | Tipo associado parametrizado por lifetimes ou tipos. Permite traits expressivas como `LendingIterator`. | [cap. 22](../part-08-generics-and-traits/ch22-gats.md) |

## H

| Termo | Definição | Capítulo |
|---|---|---|
| HashMap | Mapa hash da `std::collections`. Por padrão usa SipHash, resistente a HashDoS. | [cap. 18](../part-07-collections-and-errors/ch18-collections.md) |
| Heap | Região de memória de alocação dinâmica. Em Rust, acessada via `Box`, `Vec`, `String`. | [cap. 5](../part-02-foundations/ch05-stack-heap.md) |
| HRTB (Higher-Ranked Trait Bound) | Bound da forma `for<'a> Fn(&'a T)`, permitindo lifetimes universalmente quantificadas. | [cap. 27](../part-09-lifetimes-deep/ch27-hrtb.md) |

## I

| Termo | Definição | Capítulo |
|---|---|---|
| `impl` | Bloco que adiciona métodos ou implementa traits para um tipo. | [cap. 14](../part-05-composite-types/ch14-impl.md) |
| `impl Trait` (em retorno) | Tipo opaco existencial. Esconde o tipo concreto, sem custo de vtable. | [cap. 24](../part-08-generics-and-traits/ch24-impl-trait.md) |
| `impl Trait` (em argumento) | Açúcar para um parâmetro genérico anônimo. | [cap. 24](../part-08-generics-and-traits/ch24-impl-trait.md) |
| Interior mutability | Padrão que permite mutar dados através de uma referência imutável, com checagem em tempo de execução. | [cap. 31](../part-10-smart-pointers/ch31-cell-refcell.md) |
| Iterator | Trait com método `next() -> Option<Self::Item>`. Base de toda iteração idiomática. | [cap. 21](../part-08-generics-and-traits/ch21-iterators.md) |

## K

| Termo | Definição | Capítulo |
|---|---|---|
| Kind (de tipo) | Classificação de tipos por arity e shape. Rust não expõe isso diretamente, mas GATs e HKTs encostam no conceito. | [cap. 22](../part-08-generics-and-traits/ch22-gats.md) |

## L

| Termo | Definição | Capítulo |
|---|---|---|
| Lifetime (tempo de vida) | Anotação que descreve por quanto tempo uma referência é válida. Não muda o código gerado. | [cap. 25](../part-09-lifetimes-deep/ch25-lifetimes.md) |
| `let` / `let mut` | Vinculação de nome a valor. Imutável por padrão; `mut` exigido para reatribuir ou mutar. | [cap. 6](../part-02-foundations/ch06-bindings.md) |
| LLVM | Backend de compilação usado por `rustc`. Gera código de máquina otimizado. | [cap. 4](../part-01-genesis/ch04-pipeline.md) |
| LTO (Link-Time Optimization) | Otimização cross-crate aplicada no link. Reduz binário e melhora performance. | [cap. 55](../part-17-performance/ch55-binary-size.md) |

## M

| Termo | Definição | Capítulo |
|---|---|---|
| Macro (declarativa) | Macro definida com `macro_rules!`. Pattern matching sobre tokens, gera código. | [cap. 38](../part-14-macros/ch38-declarative.md) |
| Macro (procedural) | Macro implementada como crate Rust que recebe e devolve `TokenStream`. Base de `derive` customizado. | [cap. 39](../part-14-macros/ch39-procedural.md) |
| `match` | Expressão de pattern matching, exaustiva e com binding. Análoga a `switch` mas mais poderosa. | [cap. 15](../part-05-composite-types/ch15-match.md) |
| MIR (Mid-level IR) | Representação intermediária do compilador onde a maioria das checagens (incluindo borrow check) acontece. | [cap. 4](../part-01-genesis/ch04-pipeline.md) |
| Module | Unidade de namespace dentro de um crate. Declarada com `mod`. | [cap. 16](../part-06-modules-and-crates/ch16-modules.md) |
| Monomorphization | Estratégia de geração de código para genéricos: uma cópia por tipo concreto usado. Custo zero, binário maior. | [cap. 20](../part-08-generics-and-traits/ch20-generics.md) |
| Move | Transferência de posse. Após move, o original deixa de ser usável. | [cap. 10](../part-04-ownership/ch10-ownership.md) |
| MSRV (Minimum Supported Rust Version) | Versão mínima do compilador suportada por um crate, declarada em `Cargo.toml`. | [cap. 47](../part-16-ecosystem/ch47-msrv.md) |
| Mutex | Lock que serializa acesso a dados. `Mutex<T>` em `std::sync`. Pode envenenar em panic. | [cap. 32](../part-11-concurrency/ch32-mutex.md) |

## N

| Termo | Definição | Capítulo |
|---|---|---|
| Newtype | Padrão `struct X(T)` para criar identidade nova sobre um tipo existente. Custo zero. | [cap. 41](../part-15-patterns-and-idioms/ch41-newtype.md) |
| Niche optimization | Otimização do compilador que armazena variantes nulas em bits inutilizados. `Option<&T>` é do mesmo tamanho de `&T`. | [cap. 56](../part-17-performance/ch56-layout.md) |
| NLL (Non-Lexical Lifetimes) | Modelo onde lifetimes seguem o fluxo de dados, não escopos léxicos. Padrão desde 2018. | [cap. 26](../part-09-lifetimes-deep/ch26-nll.md) |
| `no_std` | Atributo que desliga a `std`. Usado em embedded, kernels, WebAssembly. | [cap. 47](../part-16-ecosystem/ch47-no-std.md) |
| Nominal typing | Sistema onde tipos são identificados pelo nome, não pela estrutura. Rust é nominal. | [cap. 7](../part-03-types-and-syntax/ch07-nominal.md) |

## O

| Termo | Definição | Capítulo |
|---|---|---|
| Object safety | Conjunto de regras que decide se uma trait pode virar `dyn Trait`. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |
| `Option<T>` | Enum com variantes `Some(T)` e `None`. Substitui `null` em código seguro. | [cap. 15](../part-05-composite-types/ch15-option.md) |
| Orphan rule | Regra de coerência: você só pode `impl Trait for Type` se ao menos um dos dois estiver no seu crate. | [cap. 23](../part-08-generics-and-traits/ch23-coherence.md) |
| Ownership (posse) | Modelo central do Rust: cada valor tem um e apenas um dono. Quando o dono cai, o valor é dropado. | [cap. 10](../part-04-ownership/ch10-ownership.md) |

## P

| Termo | Definição | Capítulo |
|---|---|---|
| Panic | Aborto controlado do thread em condição irrecuperável. Pode unwind ou abort. | [cap. 17](../part-07-collections-and-errors/ch17-panic.md) |
| Pattern | Estrutura usada em `match`, `let`, `if let`, `while let` para desestruturar valores. | [cap. 15](../part-05-composite-types/ch15-patterns.md) |
| `PhantomData<T>` | Marker de tamanho zero que faz o tipo "fingir" possuir um `T`, afetando variance e drop. | [cap. 27](../part-09-lifetimes-deep/ch27-phantomdata.md) |
| `Pin<P>` | Garantia de que o valor apontado não será movido. Necessário para self-referential futures. | [cap. 33](../part-12-async/ch33-pin.md) |

## Q

| Termo | Definição | Capítulo |
|---|---|---|
| `?Sized` | Bound que relaxa a exigência implícita de `Sized`, permitindo aceitar DSTs. | [cap. 24](../part-08-generics-and-traits/ch24-sized.md) |

## R

| Termo | Definição | Capítulo |
|---|---|---|
| RAII (Resource Acquisition Is Initialization) | Padrão onde um recurso é adquirido no construtor e liberado no `Drop`. Base do gerenciamento de recursos em Rust. | [cap. 13](../part-04-ownership/ch13-drop.md) |
| `Rc<T>` | Reference Counted — ponteiro de posse compartilhada single-thread. Mais barato que `Arc`. | [cap. 30](../part-10-smart-pointers/ch30-rc-arc.md) |
| `RefCell<T>` | Mutabilidade interior verificada em tempo de execução. Não é `Sync`. | [cap. 31](../part-10-smart-pointers/ch31-cell-refcell.md) |
| `repr` | Atributo que controla layout de memória de structs e enums. `repr(C)` é base de FFI. | [cap. 36](../part-13-unsafe-and-ffi/ch36-repr.md) |
| `Result<T, E>` | Enum para erro/sucesso, com variantes `Ok(T)` e `Err(E)`. Operador `?` propaga `Err`. | [cap. 17](../part-07-collections-and-errors/ch17-result.md) |
| `RwLock` | Lock que admite muitos leitores ou um escritor. Mais caro que `Mutex` em escrita. | [cap. 32](../part-11-concurrency/ch32-mutex.md) |

## S

| Termo | Definição | Capítulo |
|---|---|---|
| `Send` | Marker trait: o tipo pode ser transferido entre threads com segurança. | [cap. 32](../part-11-concurrency/ch32-send-sync.md) |
| Shadowing | Reuso de um nome com um novo `let`. Não muta — cria binding novo, possivelmente de outro tipo. | [cap. 6](../part-02-foundations/ch06-bindings.md) |
| SIMD | Single Instruction, Multiple Data. Suportado via `std::simd` (nightly) e `std::arch`. | [cap. 56](../part-17-performance/ch56-simd.md) |
| `Sized` | Marker trait implícito em todos os parâmetros de tipo, indicando que o tamanho é conhecido. | [cap. 24](../part-08-generics-and-traits/ch24-sized.md) |
| Slice | Vista contígua de elementos: `&[T]` ou `&mut [T]`. Tem ponteiro e tamanho, sem posse. | [cap. 18](../part-07-collections-and-errors/ch18-slices.md) |
| Smart pointer | Tipo que se comporta como ponteiro mas com lógica adicional (`Box`, `Rc`, `Arc`, `RefCell`). | [cap. 28](../part-10-smart-pointers/ch28-box.md) |
| `Stack` | Região de memória LIFO. Usada para variáveis locais de tamanho fixo. | [cap. 5](../part-02-foundations/ch05-stack-heap.md) |
| `std` | Biblioteca padrão completa, dependendo do SO. Compõe-se de `core` + `alloc` + APIs do sistema. | [cap. 47](../part-16-ecosystem/ch47-no-std.md) |
| `String` | String UTF-8 mutável e dona em heap. Cresce conforme necessário. | [cap. 18](../part-07-collections-and-errors/ch18-string.md) |
| `&str` | Slice imutável de UTF-8. Pode apontar para `String`, literal, ou outra origem. | [cap. 18](../part-07-collections-and-errors/ch18-string.md) |
| `Sync` | Marker trait: `&T` pode ser enviado entre threads com segurança. | [cap. 32](../part-11-concurrency/ch32-send-sync.md) |

## T

| Termo | Definição | Capítulo |
|---|---|---|
| Tokio | Runtime async mais usado em produção. Multi-thread por padrão, work-stealing. | [cap. 34](../part-12-async/ch34-tokio.md) |
| Trait | Conjunto de métodos abstratos. Análoga a interfaces, com extras (associated types, defaults, bounds). | [cap. 22](../part-08-generics-and-traits/ch22-traits.md) |
| Trait object | Valor de tipo `dyn Trait`. Carrega vtable. Permite polimorfismo dinâmico. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |
| `TryFrom` / `TryInto` | Conversão falível, retornando `Result`. Usada quando `From`/`Into` não cabe. | [cap. 22](../part-08-generics-and-traits/ch22-from-into.md) |
| Turbofish (`::<>`) | Sintaxe `f::<Tipo>()` para informar genéricos quando inferência não basta. | [cap. 20](../part-08-generics-and-traits/ch20-generics.md) |

## U

| Termo | Definição | Capítulo |
|---|---|---|
| UB (Undefined Behavior) | Comportamento que o compilador é livre para tratar como impossível. Em Rust seguro, ausente; em `unsafe`, sua responsabilidade. | [cap. 35](../part-13-unsafe-and-ffi/ch35-unsafe.md) |
| `unsafe` | Bloco ou função onde o programador assume responsabilidades que o compilador não pode verificar. | [cap. 35](../part-13-unsafe-and-ffi/ch35-unsafe.md) |
| `Unpin` | Marker auto trait: tipos que podem ser movidos mesmo dentro de `Pin`. | [cap. 33](../part-12-async/ch33-pin.md) |
| Unsizing | Coerção que transforma `T` em `dyn Trait` ou `[T; N]` em `[T]`. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |

## V

| Termo | Definição | Capítulo |
|---|---|---|
| Variance | Como subtipagem de lifetimes/tipos se propaga por construtores. Pode ser covariante, contravariante, invariante. | [cap. 27](../part-09-lifetimes-deep/ch27-variance.md) |
| `Vec<T>` | Vetor dinâmico em heap. Cresce dobrando capacidade. Suporta slice gratuitamente. | [cap. 18](../part-07-collections-and-errors/ch18-collections.md) |
| Vtable | Tabela de ponteiros para métodos, usada por `dyn Trait` para chamada dinâmica. | [cap. 24](../part-08-generics-and-traits/ch24-dyn.md) |

## W

| Termo | Definição | Capítulo |
|---|---|---|
| `where` | Cláusula que lista bounds em uma posição mais legível que inline. | [cap. 20](../part-08-generics-and-traits/ch20-generics.md) |
| Workspace | Conjunto de crates compilados juntos, com `Cargo.lock` único. Configurado em `[workspace]`. | [cap. 17](../part-06-modules-and-crates/ch17-cargo.md) |

## Z

| Termo | Definição | Capítulo |
|---|---|---|
| ZST (Zero-Sized Type) | Tipo cujo tamanho é zero (`()`, `PhantomData`, structs vazias). Não ocupa memória, mas tem identidade. | [cap. 56](../part-17-performance/ch56-layout.md) |
| Zero-cost abstraction | Princípio de design: abstrações não devem custar mais do que o equivalente escrito à mão. | [cap. 1](../part-01-genesis/ch01-por-que-rust-existe.md) |

---

> *"Aprender Rust é, em parte, aprender o vocabulário com o qual o compilador fala. Quando os termos param de assustar, o compilador deixa de ser adversário e vira interlocutor."*

[Próximo: Árvores de Decisão →](arvores-decisao.md)
