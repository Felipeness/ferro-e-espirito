# Cheat Sheet — Rust para Quem Vem de TS, Go ou C

> Tabelas de tradução direta. Use como referência rápida quando o reflexo de outra linguagem te trair.

---

## TypeScript → Rust

### Variáveis e Tipos

| TypeScript | Rust |
|---|---|
| `const x = 5` | `let x = 5;` |
| `let x = 5` | `let mut x = 5;` |
| `const X = 5 as const` | `const X: i32 = 5;` |
| `type ID = string` | `type Id = String;` |
| `type Status = 'A' \| 'B' \| 'C'` | `enum Status { A, B, C }` |
| `interface User { name: string }` | `struct User { name: String }` |
| `class User { name: string }` | `struct User { name: String } impl User { ... }` |
| `null` / `undefined` | `Option<T>` (`Some(x)` / `None`) |
| `Promise<T>` | `Future<Output = T>` |

### Funções e Closures

| TypeScript | Rust |
|---|---|
| `function foo(x: number): number { return x + 1 }` | `fn foo(x: i32) -> i32 { x + 1 }` |
| `const foo = (x: number) => x + 1` | `let foo = \|x: i32\| x + 1;` |
| `async function foo() { ... }` | `async fn foo() { ... }` |
| `await x` | `x.await` |
| `try { f() } catch (e) { ... }` | `match f() { Ok(v) => ..., Err(e) => ... }` |
| `throw new Error("...")` | `return Err(...);` ou `panic!("...")` |

### Coleções

| TypeScript | Rust |
|---|---|
| `[1, 2, 3]` (Array) | `vec![1, 2, 3]` (Vec) |
| `["a"][0]` | `vec!["a"][0]` (panica se OOB; use `.get(0)`) |
| `Map<K, V>` | `HashMap<K, V>` |
| `Set<T>` | `HashSet<T>` |
| `arr.map(x => x + 1)` | `arr.iter().map(\|x\| x + 1).collect::<Vec<_>>()` |
| `arr.filter(x => x > 0)` | `arr.iter().filter(\|x\| **x > 0).collect::<Vec<_>>()` |
| `arr.reduce((a, b) => a + b, 0)` | `arr.iter().fold(0, \|a, b\| a + b)` |
| `JSON.parse(s)` | `serde_json::from_str(&s)?` |
| `JSON.stringify(x)` | `serde_json::to_string(&x)?` |

### Pegadinhas Comuns Vindas de TS

- **Não há coerção implícita**: `"1" + 1` é erro de compilação. Bom.
- **Não há `null`**: `Option<T>` é o substituto. Use `?` para propagar.
- **Strings não são UTF-16**: são UTF-8. `s.len()` é em bytes, não chars.
- **`==` é `PartialEq`/`Eq`**: padrão é equality estrutural; precisa derivar.
- **Closures capturam por referência por default**, mas borrow checker força você a tornar isso explícito.

---

## Go → Rust

### Variáveis e Tipos

| Go | Rust |
|---|---|
| `var x int = 5` | `let mut x: i32 = 5;` |
| `x := 5` | `let x = 5;` |
| `const X = 5` | `const X: i32 = 5;` |
| `type Id int` | `struct Id(i32);` (newtype) |
| `type User struct { Name string }` | `struct User { name: String }` |
| `nil` (em ponteiro/interface) | `Option<T>` |

### Concorrência

| Go | Rust |
|---|---|
| `go f(x)` | `tokio::spawn(async move { f(x).await })` |
| `go f(x)` (sync) | `std::thread::spawn(move \|\| f(x))` |
| `chan T` | `tokio::sync::mpsc::channel::<T>(N)` |
| `make(chan T, 100)` | `tokio::sync::mpsc::channel::<T>(100)` |
| `c <- x` | `tx.send(x).await?` |
| `<-c` | `rx.recv().await?` |
| `select { ... }` | `tokio::select! { ... }` |
| `sync.Mutex` | `std::sync::Mutex` ou `tokio::sync::Mutex` |
| `sync.RWMutex` | `std::sync::RwLock` ou `tokio::sync::RwLock` |
| `context.Context` cancelation | `CancellationToken` ou drop de Future |

### Erros e Interfaces

| Go | Rust |
|---|---|
| `func f() (T, error)` | `fn f() -> Result<T, E>` |
| `if err != nil { return ..., err }` | `f()?` |
| `errors.Is(err, ErrTarget)` | `matches!(err, MyError::Target { .. })` |
| `interface Foo { Bar() }` | `trait Foo { fn bar(&self); }` |
| `defer f()` | `Drop` trait (RAII automático) |

### Pegadinhas Comuns Vindas de Go

- **Não há nil em maps/slices**: tipos sempre válidos.
- **Channels têm semântica diferente**: `mpsc` em Rust é "multi-producer, single-consumer"; para mpmc use `crossbeam` ou `tokio::sync::broadcast`.
- **Rust não tem reflection runtime**: faça via traits e generics.
- **Closures não são interfaces**: são `Fn`/`FnMut`/`FnOnce` — escolha errada gera erro de compilação.
- **Generics em Rust são monomorphized**: builds maiores, runtime mais rápido.

---

## C → Rust

### Memória e Ponteiros

| C | Rust |
|---|---|
| `int x = 5;` | `let x: i32 = 5;` |
| `int* p = malloc(sizeof(int));` | `let p = Box::new(0_i32);` |
| `free(p);` | (drop automático no fim do escopo) |
| `int arr[10];` | `let arr = [0_i32; 10];` |
| `int arr[N];` (VLA) | `let arr = vec![0_i32; n];` |
| `char* s = "hello";` | `let s: &str = "hello";` |
| `char buf[256];` | `let buf = [0_u8; 256];` |
| `void* ptr` | `*const c_void` (em FFI), generics em Rust seguro |
| `NULL` | `Option<&T>` ou `std::ptr::null()` em FFI |
| `&x` | `&x` |
| `*p` | `*p` (em unsafe se raw pointer) |

### Concorrência

| C (POSIX) | Rust |
|---|---|
| `pthread_create(&t, NULL, f, arg)` | `std::thread::spawn(move \|\| f(arg))` |
| `pthread_join(t, NULL)` | `handle.join().unwrap()` |
| `pthread_mutex_t` | `std::sync::Mutex` |
| `pthread_mutex_lock(&m)` | `let g = m.lock().unwrap();` (drop automático = unlock) |
| `atomic_int` (C11) | `std::sync::atomic::AtomicI32` |

### Strings

| C | Rust |
|---|---|
| `char* s` (mutável, dono) | `String` |
| `const char* s` (imutável, emprestado) | `&str` |
| `strlen(s)` | `s.len()` (UTF-8 byte length) |
| `strcpy(dst, src)` | `dst.push_str(src)` ou `dst.clone_from(&src)` |
| `strcat(dst, src)` | `dst.push_str(src)` |
| `printf("%d\n", x)` | `println!("{}", x)` |
| `sprintf(buf, "...", x)` | `format!("...", x)` |

### Pegadinhas Comuns Vindas de C

- **Não há undefined behavior em safe Rust**: o que UB faz em C, Rust ou compila ou recusa.
- **Buffer overflows são impossíveis em safe Rust**: bound checks por default.
- **Não há `void*`**: use generics ou `dyn Trait`.
- **`memcpy` ainda existe**: `std::ptr::copy` em unsafe; mas `Clone`/`Copy` cobre 99%.
- **`#define` é macros higiênicas**: `macro_rules!` ou proc-macro, sem captura acidental.

---

## Operações Comuns Lado a Lado

### Sort

```typescript
arr.sort((a, b) => a - b);
```

```go
sort.Ints(arr)
```

```c
qsort(arr, n, sizeof(int), cmp);
```

```rust
arr.sort();
```

### HTTP GET

```typescript
const r = await fetch(url);
const json = await r.json();
```

```go
resp, _ := http.Get(url)
defer resp.Body.Close()
var data Foo
json.NewDecoder(resp.Body).Decode(&data)
```

```c
// libcurl ou similar — dezenas de linhas
```

```rust
let json: Foo = reqwest::get(url).await?.json().await?;
```

### Read File

```typescript
const text = await fs.readFile('file.txt', 'utf8');
```

```go
data, _ := os.ReadFile("file.txt")
text := string(data)
```

```c
FILE* f = fopen("file.txt", "r");
// ... 10 linhas de boilerplate
```

```rust
let text = std::fs::read_to_string("file.txt")?;
```

---

## Erros Mentais Mais Comuns

### Vindo de TypeScript

- "Vou só botar `let` em todo lugar" — Rust é mais rigoroso. Cada variável tem tipo deduzido pela primeira atribuição e mutabilidade declarada.
- "Vou usar `null` se valor não existe" — não. `Option<T>` é o caminho.
- "Vou jogar exceção" — não. `Result<T, E>` e `?`.
- "Vou capturar tudo num try-catch" — não. Trate cada erro no tipo.

### Vindo de Go

- "Vou usar `nil` quando precisar de zero-value" — use `Default` ou `Option`.
- "Vou tipar com `interface{}`/`any`" — generics ou `Box<dyn Trait>`.
- "Vou começar 1000 goroutines despreocupadamente" — em Rust você precisa pensar em `Send`/`Sync` e tem que escolher runtime.

### Vindo de C

- "Vou fazer `malloc` direto" — `Box::new`.
- "Vou usar `void*` para genérico" — generics.
- "Vou contar bytes manualmente" — slices têm `len`, e você não passa bytes raw.
- "Vou retornar -1 para erro" — `Result`.
- "Macro do preprocessor resolve" — proc-macro, com tipagem.

---

[← Voltar ao sumário](../SUMMARY.md) | [Glossário ←](glossario.md) | [Árvores de Decisão ←](arvores-decisao.md)
