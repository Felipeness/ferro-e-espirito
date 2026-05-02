<a id="capitulo-34"></a>
# Capítulo 34: Tokio — O Runtime de Fato

> *"Tokio is the most widely used runtime, surpassing all other runtimes in usage combined."*
> — Documentação oficial do Tokio

> *"A runtime is a contract between you and the operating system about how to wait."*
> — Carl Lerche, autor original do Tokio

## 34.1 Por Que um Runtime?

No capítulo anterior vimos que em Rust o `Future` trait define o que é um trabalho assíncrono, mas **não define quem o executa**. Um `Future` é inerte. Para algo acontecer, alguém precisa chamar `poll` em loop, cuidar de quando fazer `epoll`/`kqueue`/`io_uring` no SO, distribuir trabalho entre threads, e acordar futures quando IO terminar.

Esse "alguém" é o **runtime**.

```mermaid
graph TB
    subgraph App["Sua aplicação"]
        F1[async fn handler<br/>Future]
        F2[async fn db_query<br/>Future]
        F3[async fn http_call<br/>Future]
    end

    subgraph Tokio["Tokio Runtime"]
        Sch[Scheduler<br/>work-stealing]
        Reactor[Reactor<br/>epoll / kqueue / iocp]
        Pool[Thread pool<br/>N workers]
        Timer[Timer wheel]
        Block[Blocking pool<br/>spawn_blocking]
    end

    subgraph SO["Sistema Operacional"]
        K[Kernel<br/>sockets, files, timers]
    end

    F1 -.poll.- Sch
    F2 -.poll.- Sch
    F3 -.poll.- Sch
    Sch --> Pool
    Sch --> Reactor
    Sch --> Timer
    Sch --> Block
    Reactor <-.events.- K
    Block <-.syscalls.- K

    style F1 fill:#c8e6c9
    style F2 fill:#c8e6c9
    style F3 fill:#c8e6c9
    style Sch fill:#fff9c4
    style Reactor fill:#fff9c4
    style K fill:#ffcdd2
```

Tokio resolve quatro problemas ao mesmo tempo:

1. **Scheduler.** Mantém uma fila de futures prontos para serem polled, distribui em threads, faz work-stealing entre elas.
2. **Reactor.** Conversa com o kernel via `epoll` (Linux), `kqueue` (BSD/macOS), `IOCP` (Windows), ou `io_uring` (Linux moderno) para descobrir quando um socket/arquivo está pronto para IO.
3. **Timer wheel.** Implementa `sleep`, `timeout`, `interval` sem custo por timer (uma estrutura de dados famosa: hashed timing wheel).
4. **Blocking pool.** Para CPU-bound ou syscalls genuinamente bloqueantes, oferece `spawn_blocking` em uma thread pool separada.

Compare com Node.js: o event loop é um runtime escondido na linguagem. Você não escolhe, você não troca, você não desativa. Compare com Go: o scheduler M:N está embutido no runtime do compilador. Tokio é um *crate* que você instala — `cargo add tokio` — e que sua aplicação carrega como qualquer outra dependência.

## 34.2 `#[tokio::main]` e a Construção do Runtime

A entrada mais comum:

```rust
#[tokio::main]
async fn main() {
    println!("oi do mundo async");
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    println!("um segundo depois");
}
```

`#[tokio::main]` é uma macro que reescreve seu `main` para algo equivalente a:

```rust
fn main() {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    runtime.block_on(async {
        println!("oi do mundo async");
        tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        println!("um segundo depois");
    });
}
```

Três coisas acontecem aqui.

**Primeira**, `Runtime::new()` constrói um runtime *multi-threaded* por padrão: N worker threads (um por core lógico, com mínimo de 1), um reactor dedicado, um pool de blocking threads.

**Segunda**, `block_on` é a única forma de cruzar a fronteira sync→async. Ele bloqueia a thread atual até que o future do tipo `async {}` retorne. É o que `main` precisa, porque `main` em Rust é necessariamente síncrono.

**Terceira**, todo o seu código async vive *dentro* desse `async {}`. Sem o runtime, futures não rodam.

Quando você quer mais controle:

```rust
use tokio::runtime::Builder;

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("meu-app-worker")
        .enable_all() // habilita IO + timers
        .build()
        .unwrap();

    runtime.block_on(async {
        // ...
    });
}
```

Ou single-threaded — útil para testes, embedded, ou quando você sabe que o trabalho não justifica multi-threading:

```rust
let runtime = Builder::new_current_thread()
    .enable_all()
    .build()
    .unwrap();
```

A diferença é fundamental:

| Runtime | Threads | Tipos exigidos | Quando usar |
|---|---|---|---|
| `multi_thread` (default) | N workers | `Send + 'static` | Servidores, alto throughput |
| `current_thread` | 1 | Apenas `'static` | CLIs, testes, single-task |

`current_thread` permite usar tipos `!Send` (como `Rc`, `RefCell`) dentro de tasks, porque tudo roda na mesma thread. Em troca, você não tem paralelismo real — apenas concorrência cooperativa.

## 34.3 `tokio::spawn` e o Contrato `Send + 'static`

Para executar um future em background, sem bloquear o atual:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });

    println!("aguardando...");
    let valor = handle.await.unwrap();
    println!("recebido: {}", valor);
}
```

`tokio::spawn` recebe um `Future`, retorna um `JoinHandle<T>` (você pode `.await` no handle para pegar o resultado, ou descartá-lo). Em outras linguagens:

```typescript
// TypeScript: equivalente conceitual
const promise = (async () => {
    await sleep(1000);
    return 42;
})();
const valor = await promise;
```

```go
// Go: spawn de goroutine + channel para resultado
ch := make(chan int, 1)
go func() {
    time.Sleep(time.Second)
    ch <- 42
}()
valor := <-ch
```

A assinatura completa de `tokio::spawn`:

```rust
pub fn spawn<T>(future: T) -> JoinHandle<T::Output>
where
    T: Future + Send + 'static,
    T::Output: Send + 'static,
```

Aqui mora um detalhe que assusta iniciantes: **`Send + 'static`**.

`'static` significa: o future não pode conter referências para dados na stack do chamador. Por quê? Porque o future pode rodar muito depois de o chamador retornar, talvez em outra thread. Se referenciasse uma variável local, ela já teria sido liberada.

Solução típica: mover dados para dentro do future com `move`:

```rust
let nome = String::from("Felipe");

tokio::spawn(async move {
    println!("oi {}", nome); // `nome` foi movido para dentro do future
});
```

Sem o `move`, o compilador rejeita: `nome` é capturado por referência, e a referência não vive `'static`.

`Send` significa: o future pode ser transferido entre threads. Como o scheduler multi-threaded do Tokio pode mover tasks entre workers, todos os tipos vivos *através de pontos de await* precisam ser `Send`.

Erro clássico:

```rust
use std::rc::Rc;

tokio::spawn(async {
    let rc = Rc::new(5);
    sleep(Duration::from_secs(1)).await;
    println!("{}", rc); // ERRO: Rc não é Send
});
```

`Rc` não é `Send` (sua contagem de referência não é atômica). Se você precisa de compartilhamento entre threads, use `Arc`:

```rust
use std::sync::Arc;

tokio::spawn(async {
    let rc = Arc::new(5);
    sleep(Duration::from_secs(1)).await;
    println!("{}", rc); // OK: Arc é Send + Sync
});
```

Em Go, esse cuidado não existe — todo dado é implicitamente compartilhável, e o data race é em runtime. Em Rust, você paga em sintaxe e ganha em **garantia de compile time** de que data races não acontecem.

## 34.4 Sincronização: `tokio::sync`

Quando múltiplas tasks compartilham estado, você precisa de sincronização. A `std::sync` da biblioteca padrão funciona, mas tem um problema crítico em código async: ela bloqueia a *thread* inteira, não apenas a *task*. Se a thread está rodando outras tasks, todas elas param.

Tokio oferece versões async-aware:

### `tokio::sync::Mutex` — exclusão mútua

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

let contador = Arc::new(Mutex::new(0));

let c1 = contador.clone();
tokio::spawn(async move {
    let mut guard = c1.lock().await; // .await: outras tasks rodam enquanto espera
    *guard += 1;
});
```

**Quando usar:** quando você precisa segurar o lock através de um `.await`. Se for uma seção crítica curta sem await, prefira `std::sync::Mutex` (mais rápido).

**Armadilha grave:** segurar um `Mutex` através de um `.await` pode causar deadlock se outra task no mesmo runtime precisar do mesmo lock. Veremos no próximo capítulo.

### `tokio::sync::RwLock` — múltiplos leitores, um escritor

```rust
use tokio::sync::RwLock;

let cache = Arc::new(RwLock::new(HashMap::new()));

// Leitura: várias tasks em paralelo
let r = cache.read().await;
println!("{:?}", r.get("chave"));

// Escrita: exclusiva
let mut w = cache.write().await;
w.insert("chave".to_string(), "valor".to_string());
```

**Quando usar:** caches read-heavy onde escritas são raras.

### `tokio::sync::mpsc` — multi-producer, single-consumer channel

```rust
use tokio::sync::mpsc;

let (tx, mut rx) = mpsc::channel::<String>(100); // buffer de 100

// Vários produtores
for i in 0..10 {
    let tx = tx.clone();
    tokio::spawn(async move {
        tx.send(format!("msg {}", i)).await.unwrap();
    });
}
drop(tx); // último handle: senão rx.recv() nunca retorna None

while let Some(msg) = rx.recv().await {
    println!("recebi: {}", msg);
}
```

**Quando usar:** filas de trabalho, distribuição de eventos, comunicação entre tasks. É o canal mais usado em produção.

### `tokio::sync::oneshot` — comunicação 1-para-1, uso único

```rust
use tokio::sync::oneshot;

let (tx, rx) = oneshot::channel::<u32>();

tokio::spawn(async move {
    // ... computar algo ...
    let _ = tx.send(42); // pode falhar se rx foi dropado
});

match rx.await {
    Ok(valor) => println!("recebi {}", valor),
    Err(_) => println!("sender foi dropado"),
}
```

**Quando usar:** "computa algo e me devolve uma vez". É o building block de `JoinHandle`, request/response patterns, sinais de cancelamento.

### `tokio::sync::broadcast` — multi-producer, multi-consumer

```rust
use tokio::sync::broadcast;

let (tx, _) = broadcast::channel::<String>(16);

let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

tokio::spawn(async move {
    while let Ok(msg) = rx1.recv().await {
        println!("rx1: {}", msg);
    }
});
tokio::spawn(async move {
    while let Ok(msg) = rx2.recv().await {
        println!("rx2: {}", msg);
    }
});

tx.send("hello".to_string()).unwrap(); // ambos recebem
```

**Quando usar:** publish/subscribe, eventos de domínio onde N consumidores precisam ver tudo, shutdown notifications.

**Cuidado:** se um consumidor é lento e o buffer enche, ele recebe `RecvError::Lagged` e pula mensagens. Não é um channel de "garantia de entrega".

### `tokio::sync::watch` — última-mensagem-vence

```rust
use tokio::sync::watch;

let (tx, mut rx) = watch::channel::<u64>(0);

tokio::spawn(async move {
    while rx.changed().await.is_ok() {
        let valor = *rx.borrow();
        println!("config mudou: {}", valor);
    }
});

tx.send(1).unwrap();
tx.send(2).unwrap();
tx.send(3).unwrap();
// O receiver pode ver apenas 3 (mensagens intermediárias podem ser perdidas)
```

**Quando usar:** distribuir config, status, ou estado *atual* — onde só o último valor importa. Se você não consome rápido, mensagens são sobrescritas (intencionalmente).

Resumo prático:

| Primitive | Cardinalidade | Uso típico |
|---|---|---|
| `Mutex` | 1 escrita por vez | Estado mutável compartilhado |
| `RwLock` | N leituras OU 1 escrita | Cache read-heavy |
| `mpsc` | N → 1 | Worker queues |
| `oneshot` | 1 → 1, single-shot | Request/response, sinais |
| `broadcast` | N → N, fan-out | Pub/sub, eventos |
| `watch` | N → N, latest wins | Config, status |

Comparação com Go: `mpsc` ≈ `chan T` com buffer; `oneshot` ≈ `chan T` com capacidade 1 usado uma vez; `broadcast` não tem equivalente direto (Go obriga você a fazer fan-out manual); `watch` ≈ `sync.Map` com atomic broadcast.

## 34.5 Tempo: `tokio::time`

Operações de tempo são primeira classe.

```rust
use tokio::time::{sleep, timeout, interval, Duration};

// Pausa a task atual
sleep(Duration::from_millis(500)).await;

// Limita um future por tempo
match timeout(Duration::from_secs(5), operacao_lenta()).await {
    Ok(resultado) => println!("ok: {:?}", resultado),
    Err(_) => println!("timeout"),
}

// Repete um trabalho periódico
let mut tick = interval(Duration::from_secs(1));
loop {
    tick.tick().await;
    println!("um tick por segundo");
}
```

`interval` é diferente de `loop { sleep(1s); }` — ele compensa atrasos. Se uma iteração demora 1.2s, o próximo tick não espera mais 1s, ele tenta voltar para o ritmo de 1Hz. Comportamento configurável via `MissedTickBehavior`.

Compare com Go:

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
for range ticker.C {
    fmt.Println("tick")
}
```

A semântica é parecida. A diferença é que em Rust o `interval` é uma `Stream` (capítulo 35), o que permite combiná-lo com `select!`, `take`, `filter`.

## 34.6 IO Async: `tokio::fs` e `tokio::net`

A pergunta natural: por que existe `tokio::fs` se já existe `std::fs`?

Resposta: porque **operações de arquivo bloqueiam a thread**. Não no sentido de "esperar IO" (que é o que async resolve), mas no sentido de "o kernel não tem API async para arquivos em quase nenhum sistema". Um `read()` em um arquivo grande pode bloquear o processo inteiro.

`tokio::fs` resolve isso de forma pragmática: as operações são executadas no **blocking pool** (uma thread pool separada do scheduler). Sua task async fica pendente até o IO completar; outras tasks no scheduler continuam rodando.

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let conteudo = fs::read_to_string("config.toml").await?;
    fs::write("backup.toml", &conteudo).await?;
    Ok(())
}
```

`tokio::net` é diferente: sockets têm APIs async reais no kernel (`epoll`, `kqueue`, `IOCP`). Aqui o ganho é genuíno — milhares de conexões em uma única thread.

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("escutando em :8080");

    loop {
        let (mut stream, addr) = listener.accept().await?;
        println!("conexão de {}", addr);

        tokio::spawn(async move {
            let mut buf = [0u8; 1024];
            loop {
                match stream.read(&mut buf).await {
                    Ok(0) => return, // EOF
                    Ok(n) => {
                        if stream.write_all(&buf[..n]).await.is_err() {
                            return;
                        }
                    }
                    Err(_) => return,
                }
            }
        });
    }
}
```

Esse é um echo server completo. Cada conexão entra em sua própria task. Cem mil conexões custam cem mil structs `Future` (alguns KB cada), não cem mil threads.

Compare com Node.js:

```typescript
import net from "node:net";

const server = net.createServer((socket) => {
    socket.on("data", (chunk) => socket.write(chunk));
});
server.listen(8080);
```

Equivalente em comportamento. A diferença é que Node tem **uma única thread** rodando JavaScript. Tokio multi_thread tem N threads e distribui tasks entre elas — pode literalmente saturar oito cores. Para um echo server isso não importa; para um servidor que parseia JSON pesado ou faz crypto, importa muito.

## 34.7 CPU-Bound: `spawn_blocking`

Tokio é **otimizado para IO-bound**. Se você fizer trabalho intensivo de CPU em uma task async, vai bloquear o worker thread inteiro. Outras tasks naquele worker ficam paradas até você terminar.

A solução é `spawn_blocking`:

```rust
use tokio::task;

async fn compute_handler() -> u64 {
    // Trabalho pesado de CPU: factoring, parsing, hashing
    let resultado = task::spawn_blocking(|| {
        let mut acc = 0u64;
        for i in 0..1_000_000 {
            acc = acc.wrapping_add(i);
        }
        acc
    }).await.unwrap();

    resultado
}
```

`spawn_blocking` joga a closure em uma thread pool separada (configurável, default 512 threads) reservada para trabalho bloqueante. O scheduler async continua livre.

Regra prática: **se uma operação demora mais de ~100µs sem `.await`, considere `spawn_blocking`**. Compressão, criptografia, parsing JSON gigante, cálculos — tudo isso.

Em Go isso não existe como conceito: o scheduler do Go preempta goroutines automaticamente. Em Node, é o opposite: você usa `worker_threads` para escapar do single-thread. Rust te dá controle explícito.

## 34.8 Outros Runtimes

Tokio domina, mas não é o único.

**`async-std`** tentou ser uma "std assíncrona" — APIs espelhando `std`. Boa ergonomia, mas perdeu tração. A maioria dos crates importantes (reqwest, axum, sqlx, tonic) padronizou em Tokio.

**`smol`** é minimalista — focado em ser pequeno e composável. Útil para CLIs e contextos onde Tokio é overkill. Tem comunidade pequena mas dedicada.

**`embassy`** é o runtime para **embedded**. Sem alocação de heap, sem std, roda em microcontroladores (Cortex-M, RISC-V). Você pode escrever firmware async em um chip de 32KB de RAM. É o tipo de coisa que demonstra a flexibilidade do desenho de Rust async — o mesmo trait `Future` serve do servidor de bilhões de requisições ao termômetro com 16KB.

**`monoio`** e **`glommio`** são runtimes thread-per-core, otimizados para `io_uring` no Linux moderno. Sacrificam portabilidade em troca de throughput máximo. Datacenters muito específicos os usam.

| Runtime | Foco | Quando |
|---|---|---|
| Tokio | General-purpose, servidores | 95% dos casos |
| async-std | API espelhando std | Legado |
| smol | Minimalista | CLIs, sub-libraries |
| embassy | Embedded sem alloc | Firmware, IoT |
| monoio/glommio | Thread-per-core io_uring | Hot path datacenter |

## 34.9 Por Que Tokio Venceu

Há uma lição instrutiva aqui. Tokio não venceu por ser tecnicamente o melhor — várias decisões iniciais foram revisitadas, várias APIs quebraram entre 0.1 e 1.0. Tokio venceu por **três razões sociotécnicas**.

**Patrocínio sério.** AWS contratou Carl Lerche e parte do time. Cloudflare, Discord, Microsoft contribuem. Quando empresas grandes apostam infraestrutura crítica, ecossistema segue.

**Ecossistema se travou cedo.** `hyper` (HTTP) escolheu Tokio. `tonic` (gRPC) escolheu Tokio. `sqlx`, `axum`, `tower`, `tracing` — todos Tokio-first. Em algum momento, escrever um crate "runtime-agnóstico" virou impossível na prática. A comunidade convergiu por gravidade.

**Estabilidade na 1.0.** Desde 2020, Tokio 1.x manteve compatibilidade. Crates podem depender de `tokio = "1"` por anos sem quebrar. Compare com Python `asyncio`, que mudou três vezes em sete anos.

A consequência: se você está escrevendo um servidor Rust em produção em 2026, está usando Tokio. Não é dogma — é o que vai te dar comunidade, exemplos no Stack Overflow, integração com tudo.

## 34.10 Anatomia de um Servidor Real

Um exemplo que junta tudo: servidor TCP com graceful shutdown, métricas, e timeout por conexão.

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::sync::broadcast;
use tokio::time::{timeout, Duration};

#[derive(Default)]
struct Metrics {
    conexoes_total: AtomicU64,
    conexoes_ativas: AtomicU64,
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let metrics = Arc::new(Metrics::default());
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    // Listener Ctrl+C
    let st = shutdown_tx.clone();
    tokio::spawn(async move {
        tokio::signal::ctrl_c().await.ok();
        println!("shutdown solicitado");
        let _ = st.send(());
    });

    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    let mut shutdown_rx = shutdown_tx.subscribe();

    loop {
        tokio::select! {
            res = listener.accept() => {
                let (stream, _addr) = res?;
                let m = metrics.clone();
                let mut sd = shutdown_tx.subscribe();
                tokio::spawn(async move {
                    m.conexoes_total.fetch_add(1, Ordering::Relaxed);
                    m.conexoes_ativas.fetch_add(1, Ordering::Relaxed);
                    tokio::select! {
                        _ = handle_conn(stream) => {}
                        _ = sd.recv() => {}
                    }
                    m.conexoes_ativas.fetch_sub(1, Ordering::Relaxed);
                });
            }
            _ = shutdown_rx.recv() => {
                println!("parando o accept loop");
                break;
            }
        }
    }
    Ok(())
}

async fn handle_conn(mut s: tokio::net::TcpStream) {
    let mut buf = [0u8; 1024];
    loop {
        match timeout(Duration::from_secs(30), s.read(&mut buf)).await {
            Ok(Ok(0)) | Ok(Err(_)) | Err(_) => return,
            Ok(Ok(n)) => {
                if s.write_all(&buf[..n]).await.is_err() {
                    return;
                }
            }
        }
    }
}
```

Cinco features de produção em ~50 linhas:

- **Graceful shutdown** via `broadcast` channel.
- **Sinal SIGINT** capturado por `tokio::signal::ctrl_c()`.
- **Timeout por conexão** com `tokio::time::timeout`.
- **Métricas** com `Arc<AtomicU64>` (sem locking).
- **Cancellation** automática quando shutdown é sinalizado — por `select!`.

Faremos `select!` e cancellation em detalhe no próximo capítulo. Por ora, note que o código *parece* síncrono. A complexidade do scheduler, do reactor, do work-stealing, está toda escondida atrás de `.await`.

## 34.11 Resumo

Tokio é **o** runtime async de Rust por convergência de comunidade, não por ser o único possível. Suas ideias centrais:

- **Multi-threaded por padrão**, single-threaded quando você precisa de `!Send`.
- **`spawn` com `Send + 'static`** — o preço da segurança em compile time.
- **`tokio::sync` em vez de `std::sync`** quando o lock cruza um `await`.
- **`tokio::fs` para arquivos** (blocking pool), **`tokio::net` para sockets** (reactor real).
- **`spawn_blocking` para CPU-bound** — não estrague o scheduler.

Comparado a Node.js, Tokio escala para múltiplos cores sem `cluster`. Comparado a Go, exige `Send + 'static` explícito mas não tem GC. Comparado a Python `asyncio`, é estável, performático, e não muda toda versão.

No próximo capítulo, os padrões idiomáticos: `select!`, streams, cancellation safety, e os bugs que custam um sprint inteiro quando você não conhece.

---

> *"Tokio não é magia. É um loop que chama `poll`, um epoll, um pool de threads, e uma timing wheel. Que essa simplicidade rode milhões de conexões é o que torna Rust possível."*

[Próximo: Capítulo 35 — Padrões Async: Select, Cancellation, Streams →](ch35-padroes-async.md)
