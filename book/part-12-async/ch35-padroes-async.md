<a id="capitulo-35"></a>
# Capítulo 35: Padrões Async — Select, Cancellation, Streams

> *"Cancellation is not an event. It is the absence of execution."*
> — Folclore Tokio

> *"In async Rust, dropping is cancelling. That sentence took me a year to fully appreciate."*
> — Niko Matsakis

## 35.1 O Que Este Capítulo Resolve

Os capítulos anteriores explicaram o **modelo** (`Future`, lazy, máquina de estados) e o **runtime** (Tokio, scheduler, primitives de sync). O que falta é o **vocabulário do dia a dia**: como você escreve um servidor que aguarda *qualquer um* de três eventos, como você cancela um trabalho na metade, como você consome um stream de mensagens, como você não cria deadlocks segurando um mutex pelo `await` errado.

Cada um desses padrões tem um análogo em TS e em Go. Cada um tem uma armadilha que o compilador *não pega*. Vamos ver.

## 35.2 `tokio::select!` — Esperar o Primeiro

A primitiva: aguarde N futures e prossiga com o que terminar primeiro. Os outros são **dropados** (cancelados).

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_millis(100)) => {
            println!("100ms ganhou");
        }
        _ = sleep(Duration::from_millis(200)) => {
            println!("200ms ganhou");
        }
    }
    // Saída: "100ms ganhou". O segundo sleep foi DROPADO.
}
```

A sintaxe segue um padrão: `<padrão> = <future> => <handler>`. O macro polla os futures de forma intercalada. Quando um produz `Ready`, o handler correspondente roda; os outros são abandonados.

Compare com TypeScript:

```typescript
const r = await Promise.race([
    sleep(100).then(() => "100ms"),
    sleep(200).then(() => "200ms"),
]);
console.log(`${r} ganhou`);
// Saída: "100ms ganhou". Mas o segundo sleep CONTINUA rodando até terminar.
```

Diferença crucial: `Promise.race` em JS não cancela o perdedor. Ele apenas ignora o resultado. O `setTimeout(200)` continua queimando recursos no event loop.

Em Go, o equivalente é `select` (palavra-chave da linguagem):

```go
select {
case <-time.After(100 * time.Millisecond):
    fmt.Println("100ms ganhou")
case <-time.After(200 * time.Millisecond):
    fmt.Println("200ms ganhou")
}
// O timer perdedor permanece até o GC limpar.
```

Go também não cancela os timers automaticamente. Em Rust, dropar é cancelar — recursos são liberados, sockets fechados, locks devolvidos.

### Padrão: timeout manual

`tokio::time::timeout` é açúcar para um `select!`:

```rust
async fn com_timeout<F, T>(f: F, dur: Duration) -> Result<T, &'static str>
where
    F: Future<Output = T>,
{
    tokio::select! {
        v = f => Ok(v),
        _ = sleep(dur) => Err("timeout"),
    }
}
```

### Padrão: shutdown signal

Servidor que aguarda conexões mas para no Ctrl+C:

```rust
let (shutdown_tx, _) = tokio::sync::broadcast::channel::<()>(1);
let mut shutdown_rx = shutdown_tx.subscribe();

loop {
    tokio::select! {
        Ok((stream, _)) = listener.accept() => {
            tokio::spawn(handle(stream));
        }
        _ = shutdown_rx.recv() => {
            println!("graceful shutdown");
            break;
        }
    }
}
```

Esse é o padrão idiomático em Tokio. Sem `select!`, você precisaria de uma flag atômica e polling.

### Branch guards e `else`

```rust
let mut feito = false;
tokio::select! {
    biased; // tenta na ordem em que aparecem (sem random)
    v = receber_msg(), if !feito => {
        // só ativa se !feito
    }
    _ = sleep(Duration::from_secs(5)) => {
        feito = true;
    }
    else => {
        // todos os branches estão desabilitados
    }
}
```

`biased;` desabilita o random (Tokio embaralha por padrão para evitar starvation). `, if cond` é uma precondição. `else =>` roda quando todas as condições são `false`.

## 35.3 Cancellation Safety — O Bug Que Não Compila Mensagem

A documentação do Tokio dedica páginas a esse tópico. Vale a leitura porque é onde mais bugs de produção async em Rust nascem.

**Definição:** um future é *cancellation safe* se pode ser dropado a qualquer momento sem **perda de dados** ou **estado inconsistente**.

A maior parte dos futures simples é safe. `sleep` é safe. `TcpStream::write_all` é safe. `Mutex::lock` é safe.

Mas alguns não são. O exemplo canônico:

```rust
use tokio::sync::mpsc;

async fn worker(mut rx: mpsc::Receiver<String>) {
    loop {
        tokio::select! {
            Some(msg) = rx.recv() => {
                processar(msg).await;
            }
            _ = sleep(Duration::from_secs(1)) => {
                println!("heartbeat");
            }
        }
    }
}
```

Esse código está **correto**, porque `mpsc::Receiver::recv()` é cancellation safe — se for dropado antes de receber uma mensagem, a mensagem permanece no canal e a próxima chamada a `recv()` a pega.

Compare com:

```rust
async fn worker_quebrado<R: tokio::io::AsyncRead + Unpin>(mut leitor: R) {
    let mut buf = String::new();
    loop {
        tokio::select! {
            // PERIGO: read_to_string NÃO é cancellation safe
            res = tokio::io::AsyncReadExt::read_to_string(&mut leitor, &mut buf) => {
                processar(&buf).await;
                buf.clear();
            }
            _ = sleep(Duration::from_secs(1)) => {
                println!("heartbeat");
            }
        }
    }
}
```

`read_to_string` lê em um buffer interno até EOF. Se for dropado no meio, **bytes já lidos são perdidos**. O futuro polling começa do zero.

Como saber se um future é cancellation safe? A documentação do Tokio marca explicitamente: cada método na documentação tem uma seção "Cancel safety" indicando se é "safe to cancel" ou "no longer safe" se interrompido.

Regra de ouro: **se um método pode estar no meio de uma transação multi-step (ler até delimitador, escrever várias mensagens), ele provavelmente não é cancellation safe.**

Quando você precisa de algo não-safe em um `select!`, a solução é pin-fora-do-loop:

```rust
let leitura = tokio::io::AsyncReadExt::read_to_string(&mut leitor, &mut buf);
tokio::pin!(leitura);

loop {
    tokio::select! {
        res = &mut leitura => {
            // termina nesse braço, sai do loop
            break;
        }
        _ = sleep(Duration::from_secs(1)) => {
            // o future `leitura` continua de onde parou na próxima iteração
        }
    }
}
```

`tokio::pin!` põe o future em uma localização fixa de memória. `&mut leitura` produz uma referência mutável que pode ser polada várias vezes — preservando estado entre iterações de `select!`.

## 35.4 O Pecado Capital: Mutex Sobre `await`

O bug que mais custa em código async Rust de produção:

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

async fn handler_quebrado(estado: Arc<Mutex<HashMap<String, String>>>) {
    let guard = estado.lock().await;     // adquire lock
    let valor = guard.get("chave").cloned().unwrap_or_default();

    chamada_http_lenta(&valor).await;     // SEGURA O LOCK pelo await inteiro

    println!("{}", valor);
} // só libera o lock aqui
```

Enquanto o `chamada_http_lenta` espera (talvez 5 segundos), **nenhuma outra task pode acessar o mapa**. Se o servidor tem mil conexões, novecentas e noventa e nove ficam paradas.

Pior: se a `chamada_http_lenta` internamente tentar adquirir o mesmo lock (digamos via outro handler), você tem **deadlock**.

Solução: **nunca segure um lock através de `.await`**. Extraia o que precisa, libere, depois faça o trabalho async:

```rust
async fn handler_correto(estado: Arc<Mutex<HashMap<String, String>>>) {
    let valor = {
        let guard = estado.lock().await;
        guard.get("chave").cloned().unwrap_or_default()
    }; // guard sai de escopo aqui, lock liberado

    chamada_http_lenta(&valor).await; // sem lock segurado
    println!("{}", valor);
}
```

O escopo de bloco `{}` é seu amigo. O `guard` morre antes do `await`.

O Clippy detecta esse padrão com a lint `await_holding_lock`. Habilite em CI:

```toml
# clippy.toml
disallowed-methods = []
# Cargo.toml
[lints.clippy]
await_holding_lock = "deny"
```

Compare com Go: lá, segurar um `sync.Mutex` durante uma chamada que bloqueia é igualmente ruim, mas o problema é mais visível porque Go não tem `await` — quando você chama uma função que bloqueia, isso é óbvio. Em Rust async, qualquer `.await` é um ponto de yield e o lock vai junto.

## 35.5 `join!`, `try_join!`, `join_all` — Esperar Vários

Quando você quer rodar N futures **em paralelo** e esperar todos:

### `tokio::join!` — número fixo, tipos heterogêneos

```rust
use tokio::join;

async fn buscar() {
    let (usuario, posts, perfil) = join!(
        buscar_usuario(),
        buscar_posts(),
        buscar_perfil(),
    );
    println!("{:?} {:?} {:?}", usuario, posts, perfil);
}
```

`join!` polla os três simultaneamente, retorna quando todos terminam. Tipos podem ser diferentes — é uma tupla de retornos.

### `tokio::try_join!` — short-circuit em erro

```rust
async fn buscar() -> Result<(), Erro> {
    let (u, p, perf) = tokio::try_join!(
        buscar_usuario(),
        buscar_posts(),
        buscar_perfil(),
    )?;
    Ok(())
}
```

Se *qualquer* future retorna `Err`, os outros são **dropados** (cancelados) e o `try_join!` retorna o erro imediatamente. Comparável a `Promise.all` em TS, que rejeita assim que uma promise rejeita.

### `futures::future::join_all` — número dinâmico

```rust
use futures::future::join_all;

async fn buscar_todos(ids: Vec<u32>) -> Vec<Usuario> {
    let futures = ids.into_iter().map(|id| buscar_usuario(id));
    join_all(futures).await
}
```

`join_all` aceita qualquer iterador de futures, retorna `Vec<T>`. Útil quando N é runtime-dependente.

**Cuidado:** sem limite de paralelismo. Para 10.000 IDs, dispara 10.000 requisições simultâneas. Use `buffer_unordered` (próxima seção) quando precisar de bounded parallelism.

### Comparação

| Linguagem | Espera-todos | Espera-todos-ou-falha | Limite |
|---|---|---|---|
| Rust | `join!` / `join_all` | `try_join!` | `buffer_unordered` |
| TypeScript | `Promise.all` (curto-circuita em erro) | mesmo | `p-limit` |
| Go | `sync.WaitGroup` + atomic | `errgroup.Group` | `errgroup` + semaphore |

Note: `Promise.all` em TS sempre curto-circuita; não há equivalente direto a `join!` (espera todos, mesmo se um falhar). Você simula com `Promise.allSettled`.

## 35.6 Streams — Iterators Async

Um `Iterator` produz valores síncronos. Um `Stream` produz valores assíncronos.

```rust
pub trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

Mesma estrutura de `Future`, mas `poll_next` retorna `Option<Item>` — `Some(v)` é o próximo item, `None` é fim.

Em código real, você raramente implementa `Stream`. Você consome:

```rust
use tokio_stream::StreamExt;

async fn processar_msgs(mut stream: impl tokio_stream::Stream<Item = String> + Unpin) {
    while let Some(msg) = stream.next().await {
        println!("{}", msg);
    }
}
```

Ou compõe:

```rust
use futures::stream::{self, StreamExt};

async fn processar_em_paralelo(urls: Vec<String>) {
    stream::iter(urls)
        .map(|url| async move {
            reqwest::get(&url).await.ok()
        })
        .buffer_unordered(10) // até 10 em voo
        .for_each(|resp| async move {
            if let Some(r) = resp {
                println!("{}", r.status());
            }
        })
        .await;
}
```

`buffer_unordered(N)` é o padrão para **bounded parallelism**: pega um stream de futures, executa até N simultaneamente, emite resultados conforme terminam (sem manter ordem). Se quiser ordem, use `buffered(N)`.

Comparações:

```typescript
// TypeScript: AsyncIterator
async function* gerar(): AsyncGenerator<string> {
    yield "a";
    yield "b";
}

for await (const v of gerar()) {
    console.log(v);
}
```

JavaScript tem `AsyncIterator` desde 2018. Funciona, mas não tem combinators — se quiser bounded parallelism, use bibliotecas externas.

```go
// Go: channels são streams naturais
ch := gerar()
for v := range ch {
    fmt.Println(v)
}
```

Em Go, todo channel é um stream. Combinators? Você escreve à mão.

Em Rust, `Stream` mais `StreamExt` te dá o que TypeScript chama de `Observable` em RxJS — `map`, `filter`, `take`, `skip`, `zip`, `chain`, `merge`, `throttle`, `chunks_timeout`. Tudo composável, tudo zero-cost.

### Padrão: stream de eventos

```rust
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;

let (tx, _) = broadcast::channel::<Evento>(100);

let rx = tx.subscribe();
let mut stream = BroadcastStream::new(rx);

while let Some(Ok(evento)) = stream.next().await {
    aplicar(&evento).await;
}
```

`BroadcastStream` adapta um `broadcast::Receiver` para `Stream`. Agora você pode aplicar `.filter`, `.take(100)`, `.timeout` em eventos.

## 35.7 `AsyncRead` e `AsyncWrite`

Para IO byte-oriented, Tokio expõe traits:

```rust
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,
    ) -> Poll<io::Result<()>>;
}

pub trait AsyncWrite {
    fn poll_write(...) -> Poll<io::Result<usize>>;
    fn poll_flush(...) -> Poll<io::Result<()>>;
    fn poll_shutdown(...) -> Poll<io::Result<()>>;
}
```

Você quase nunca chama `poll_*` diretamente. As extension traits `AsyncReadExt` e `AsyncWriteExt` dão os métodos ergonômicos:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::fs::File;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut f = File::open("entrada.bin").await?;
    let mut buf = Vec::new();
    f.read_to_end(&mut buf).await?;

    let mut out = File::create("saida.bin").await?;
    out.write_all(&buf).await?;
    out.flush().await?;
    Ok(())
}
```

Tudo que implementa `AsyncRead + Unpin` é uma fonte de bytes async — `TcpStream`, `File`, `ChildStdout`, sockets Unix, pipes. Você escreve uma função genérica `async fn parse<R: AsyncRead + Unpin>(r: R)` e ela funciona em qualquer um.

Comparação rápida:

| Conceito | Rust | Node.js | Go |
|---|---|---|---|
| Iterator async | `Stream` + `StreamExt` | `AsyncIterator` | `chan T` + range |
| Byte stream read | `AsyncRead` | `Readable` stream | `io.Reader` (sync) |
| Byte stream write | `AsyncWrite` | `Writable` stream | `io.Writer` (sync) |

Note como Go usa as mesmas interfaces sync para IO async — porque o scheduler torna o IO transparente. Em Rust, async é explícito, então o trait precisa ser explícito também.

## 35.8 Pin: O Mínimo Que Você Precisa Saber Agora

Já mencionamos `Pin` várias vezes. O capítulo 36 dedica trinta páginas. Aqui, o mínimo para o dia a dia.

**Por que Pin existe:** futures gerados por `async fn` podem ser self-referenciais. Mover na memória quebra essas referências. Pin garante que enquanto um valor está sendo usado como future, ele não pode ser movido.

**Onde você esbarra:**

1. **`Pin<Box<dyn Future>>`** — quando você precisa armazenar futures heterogêneos em uma `Vec` ou retornar de uma função sem `impl Trait`:

```rust
type FutureBox = Pin<Box<dyn Future<Output = ()> + Send>>;

fn obter_future() -> FutureBox {
    Box::pin(async {
        sleep(Duration::from_secs(1)).await;
    })
}
```

2. **`tokio::pin!`** — para usar futures não-`Unpin` em loops com `select!`:

```rust
let f = operacao_longa();
tokio::pin!(f);
loop {
    tokio::select! {
        _ = &mut f => break,
        _ = sleep(Duration::from_secs(1)) => println!("ainda esperando"),
    }
}
```

3. **`Unpin`** — auto-trait que diz "este tipo é seguro para mover livremente". Quase tudo que você usa é `Unpin` (primitivos, `String`, `Vec`, structs simples). Apenas futures gerados por `async fn` e tipos auto-referenciais não são.

**Regra prática:** se o compilador reclamar de `Unpin`, embrulhe com `Box::pin` ou `tokio::pin!`. A maioria dos programas async escapa sem tocar `Pin` diretamente.

## 35.9 Padrão: Race com Cleanup

Você quer fazer duas operações em paralelo. Quando uma terminar, cancelar a outra **e fazer cleanup**.

```rust
use tokio::sync::oneshot;

async fn primeira_que_chegar() -> Resultado {
    let (cancel_tx, cancel_rx) = oneshot::channel::<()>();

    let f1 = async {
        tokio::select! {
            r = trabalho_a() => Some(r),
            _ = cancel_rx => None,
        }
    };
    let f2 = trabalho_b();

    tokio::select! {
        r = f1 => {
            r.unwrap_or_else(|| panic!("f1 cancelado"))
        }
        r = f2 => {
            let _ = cancel_tx.send(()); // sinaliza cleanup para f1
            r
        }
    }
}
```

O `oneshot::channel` aqui é o sinal explícito de cancelamento. Quando `f2` ganha, mandamos `()` por `cancel_tx`; o `cancel_rx` dentro de `f1` resolve, e `f1` aborta limpamente.

Em casos simples, dropar o future é suficiente. Mas se a `trabalho_a` precisa **fazer cleanup** (fechar conexão, escrever log), você precisa de cooperação explícita — porque dropar não roda código async.

Esse é um ponto importante: **destructors em Rust são síncronos**. Se sua task precisa fazer cleanup que envolve await, você não pode confiar em `Drop`. Use canais de cancelamento explícitos ou `tokio_util::sync::CancellationToken`.

## 35.10 `CancellationToken`: O Padrão Hierárquico

Para sistemas grandes, `tokio_util::sync::CancellationToken` é o que se usa:

```rust
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() {
    let raiz = CancellationToken::new();

    for i in 0..10 {
        let tok = raiz.child_token();
        tokio::spawn(async move {
            tokio::select! {
                _ = trabalho(i) => println!("worker {} terminou", i),
                _ = tok.cancelled() => println!("worker {} cancelado", i),
            }
        });
    }

    tokio::signal::ctrl_c().await.ok();
    raiz.cancel(); // cancela TODOS os children em cascata
    sleep(Duration::from_secs(1)).await;
}
```

Cancelar o token raiz cancela toda a árvore. Você pode passar `child_token()` para sub-tasks, formando hierarquias arbitrárias. É o equivalente a `context.Context` em Go — mas em Go é convenção da std, em Rust é uma crate adicional.

## 35.11 Erro Comum: `await` em Mutex Sync da `std`

Variação do problema da seção 35.4. Suponha que você use `std::sync::Mutex` por engano:

```rust
use std::sync::Mutex;
use std::sync::Arc;

async fn handler(estado: Arc<Mutex<u32>>) {
    let guard = estado.lock().unwrap();   // bloqueia a THREAD
    sleep(Duration::from_secs(1)).await;  // a thread está bloqueada
    println!("{}", *guard);
}
```

`std::sync::Mutex::lock()` é síncrono. Se outra thread está segurando o lock, sua thread inteira *para* — junto com todas as outras tasks que ela estava rodando. Pode até ser aceitável se o lock for muito breve (microssegundos). Mas com `await` no meio, é receita para latência terrível.

Variantes do problema:

- `RefCell::borrow` em código async com runtime multi-threaded: panic se duas tasks borrowarem ao mesmo tempo.
- `Rc<T>` sendo movido para uma task `spawn`: erro de compilação por `!Send` (o compilador te salva aqui).

Regra: **em código async, prefira sempre `tokio::sync::*` para qualquer lock que possa cruzar `await`. Use `std::sync::*` apenas para seções críticas estritamente síncronas.**

## 35.12 Resumo Comparativo

Os padrões essenciais e seus equivalentes:

| Padrão | Rust | TypeScript | Go |
|---|---|---|---|
| Esperar primeiro | `tokio::select!` | `Promise.race` | `select { case <-ch: }` |
| Esperar todos | `tokio::join!` / `join_all` | `Promise.all` | `sync.WaitGroup` |
| Esperar todos com erro | `tokio::try_join!` | `Promise.all` | `errgroup.Group` |
| Stream async | `Stream` + `StreamExt` | `AsyncIterator` | `chan T` |
| Cancellation | drop + `CancellationToken` | `AbortController` | `context.Context` |
| Timeout | `tokio::time::timeout` | `Promise.race` + `setTimeout` | `context.WithTimeout` |
| Bounded parallelism | `buffer_unordered(N)` | `p-limit` | semaphore + errgroup |

A coluna do Rust é mais densa em uma só razão: **cancellation é gratuita** (drop é cancelar) **e composição é tipada** (combinators retornam tipos exatos). Isso é o que paga o preço de `Pin`, `Send`, `'static`.

## 35.13 Checklist Antes de Mergear Código Async

Pequena lista mental para revisar PR de código async:

1. **Algum `Mutex`/`RwLock` é segurado através de `await`?** Veja se o lock pode ser solto antes (bloco com `{}`) ou se você precisa de `tokio::sync::Mutex`.
2. **Algum `select!` tem branches com futures não-cancellation-safe?** `read_to_string`, `read_to_end`, qualquer coisa que acumula estado pelo caminho. Se sim, pin fora do loop.
3. **Você está usando `Promise.all`-style sem limite?** `join_all` em mil URLs vai abrir mil conexões. Use `buffer_unordered`.
4. **`spawn` está movendo dados que precisam ser `Send + 'static`?** Compilador pega isso, mas a mensagem pode ser longa. `Arc` em vez de `Rc`, clones de `String` em vez de `&str`, `move` no closure.
5. **CPU-bound no async runtime?** Use `spawn_blocking`. Se um handler demora 50ms sem await, você está bloqueando o worker thread.
6. **Cancellation faz cleanup async?** `Drop` é síncrono. Se você precisa fechar conexão *async* no cancelamento, use `CancellationToken` + `select!` explícito.
7. **Timeouts em **toda** chamada externa?** `tokio::time::timeout` em volta de qualquer HTTP, RPC, query. Sem isso, um serviço lento trava o seu.

Esses sete pontos resolvem 90% dos bugs de código async em produção que vi em PRs de Rust nos últimos anos.

## 35.14 Resumo

Async em Rust é elegante quando você domina o vocabulário. Os padrões essenciais:

- **`select!`** para esperar o primeiro de N eventos. Cancela os perdedores via drop.
- **Cancellation safety** é uma propriedade que você precisa verificar — o compilador não pega.
- **Lock sobre await** é o pecado capital. Solte o guard antes de qualquer `.await`.
- **`join!`/`try_join!`/`join_all`** para paralelismo determinístico. `buffer_unordered` para limitar.
- **`Stream`** é o iterator async. `StreamExt` traz dezenas de combinators.
- **`Pin`** aparece em assinaturas; na prática, `Box::pin` ou `tokio::pin!` resolvem.
- **`CancellationToken`** para cancelamento hierárquico, equivalente a `context.Context` de Go.

Comparado a TypeScript, você ganha cancellation real, paralelismo controlado, tipos exatos. Paga em vocabulário (`Pin`, cancellation safety) e em compile-time errors longos.

Comparado a Go, você ganha previsibilidade de memória, ausência de GC, expressão tipada de "qual future espera qual". Paga na obrigação de pensar em `Send`, `'static`, e cancellation safety.

Os capítulos seguintes vão fundo em `Pin` (capítulo 36), em construir um servidor HTTP completo com Axum (capítulo 37), e em testar código async (capítulo 38). O vocabulário básico, você já tem.

---

> *"Se você terminou estes três capítulos sem enxergar `Pin` como inimigo, e enxergando cancellation safety como uma pergunta natural a se fazer em todo `select!`, você atravessou o vale onde a maioria desiste. O resto é prática."*

[Próximo: Capítulo 36 — Pin, Unpin e Self-References →](ch36-pin-unpin.md)
