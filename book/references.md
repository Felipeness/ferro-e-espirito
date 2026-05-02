# Referências

> Toda obra técnica é uma conversa em curso. Estas são as vozes que ressoam neste livro.

---

## Documentação Oficial

- **[The Rust Programming Language](https://doc.rust-lang.org/book/)** — Steve Klabnik, Carol Nichols, Chris Krycho. O livro canônico, gratuito, atualizado a cada release.
- **[The Rustonomicon](https://doc.rust-lang.org/nomicon/)** — A bíblia do `unsafe`. Não para iniciantes.
- **[Rust by Example](https://doc.rust-lang.org/rust-by-example/)** — Aprender por código.
- **[The Cargo Book](https://doc.rust-lang.org/cargo/)** — Tudo sobre Cargo e o ecossistema.
- **[The Async Book](https://rust-lang.github.io/async-book/)** — Fundamentos de async em Rust.
- **[The Rust Reference](https://doc.rust-lang.org/reference/)** — Especificação semi-formal da linguagem.
- **[The Rustc Dev Guide](https://rustc-dev-guide.rust-lang.org/)** — Como o compilador funciona por dentro.
- **[The Embedded Rust Book](https://docs.rust-embedded.org/book/)** — Bare-metal e microcontroladores.
- **[The Edition Guide](https://doc.rust-lang.org/edition-guide/)** — Migração entre editions (2015/2018/2021/2024).
- **[Standard Library Documentation](https://doc.rust-lang.org/std/)** — A referência mais consultada do mundo Rust.

## Livros Imprescindíveis

- **Programming Rust** (3rd ed., 2024) — Jim Blandy, Jason Orendorff, Leonora Tindall. O *Programming X* da O'Reilly que vale o nome.
- **Rust for Rustaceans** (2021) — Jon Gjengset. Para quem já passou do hello world. Denso, opinativo, brilhante.
- **Rust Atomics and Locks** (2023) — Mara Bos. [Free online](https://marabos.nl/atomics/). Leitura obrigatória se concorrência te interessa.
- **Zero To Production In Rust** (2022) — Luca Palmieri. Construir um SaaS real, capítulo a capítulo.
- **Effective Rust** (2024) — David Drysdale. 35 itens no estilo *Effective Java* de Joshua Bloch.
- **The Little Book of Rust Macros** — Daniel Keep, atualizado por Veykril. [veykril.github.io/tlborm](https://veykril.github.io/tlborm/).
- **Rust in Action** (2021) — Tim McNamara. Systems programming concreto.
- **Hands-On Concurrency with Rust** — Brian Troutwine.
- **Rust Design Patterns** — [rust-unofficial.github.io/patterns](https://rust-unofficial.github.io/patterns/).

## Artigos e Posts Definitivos

- **[Rust 2018 — Two Years of Rust](https://blog.rust-lang.org/2017/12/21/rust-in-2017.html)** e posts subsequentes anuais — Rust Team.
- **[Rust's Two Kinds of 'Assert'](https://matklad.github.io/2020/10/18/lsp-back-end-architecture.html)** — Aleksey Kladov.
- **[Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md)** — pretzelhammer.
- **[Pin and Unpin](https://blog.cloudflare.com/pin-and-unpin-in-rust/)** — Cloudflare Engineering.
- **[Choosing Your Guarantees](https://manishearth.github.io/blog/2015/05/27/wrapper-types-in-rust-choosing-your-guarantees/)** — Manish Goregaokar.
- **[Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/)** — Andrew Gallant (BurntSushi).
- **[The Embedded Rustacean](https://blog.theembeddedrustacean.com/)** — Omar Hiari.
- **[Without boats](https://without.boats/)** — withoutboats, ex-membro do lang team. Reflexões profundas sobre futures, async, design.
- **[Niko Matsakis](https://smallcultfollowing.com/babysteps/)** — Co-líder histórico do lang team. Borrow checker, NLL, Polonius.
- **[Faster than Lime](https://fasterthanli.me/)** — Amos. Tutoriais longos com profundidade rara.

## Para Assistir

- **[Crust of Rust](https://www.youtube.com/playlist?list=PLqbS7AVVErFiWDOAVrPt7aYmnuuOLYvOa)** — Jon Gjengset. Streams de várias horas reimplementando coisas reais do stdlib. Caminho mais curto da intermediação à maestria.
- **[Let's Get Rusty](https://www.youtube.com/@letsgetrusty)** — Bogdan Pshonyak. Tutoriais curtos, didáticos.
- **RustConf** — Talks anuais da conferência principal. YouTube.

## Comunidade

- **[r/rust](https://reddit.com/r/rust)** — Subreddit ativo, gentil, técnico.
- **[Rust Users Forum](https://users.rust-lang.org/)** — Fórum oficial.
- **[Rust Internals Forum](https://internals.rust-lang.org/)** — Para discussões de design de linguagem.
- **[This Week in Rust](https://this-week-in-rust.org/)** — Newsletter semanal indispensável.
- **[Rust Discord](https://discord.gg/rust-lang)** — Real-time community.

## Crates Mencionados (Catálogo Rápido)

| Crate | Domínio | Descrição |
|---|---|---|
| `tokio` | async | Runtime padrão de fato |
| `axum` | web | Framework web do time Tokio |
| `actix-web` | web | Alternativa madura, menos ergonômica |
| `serde` | serialização | A melhor lib de serialização que existe |
| `clap` | CLI | Argument parser declarativo |
| `anyhow` | erros | Error type para applications |
| `thiserror` | erros | Derive de `Error` para libraries |
| `tracing` | observabilidade | Structured logging + spans |
| `sqlx` | banco | Async SQL com queries verificadas em compile-time |
| `criterion` | benchmark | Benchmarking estatístico |
| `proptest` / `quickcheck` | testes | Property-based testing |
| `rayon` | paralelismo | Data parallelism via fork-join |
| `crossbeam` | concorrência | Channels e primitivos avançados |
| `parking_lot` | locks | Mutex/RwLock mais rápidos que std |
| `bytes` | I/O | Buffers eficientes |
| `hyper` | HTTP | HTTP low-level (axum usa internamente) |
| `reqwest` | HTTP cliente | HTTP client ergonômico |
| `bevy` | jogos | ECS engine moderno |
| `embassy` | embedded | Async em microcontroladores |
| `wasm-bindgen` | WASM | Interop Rust ↔ JS |
| `pyo3` | FFI Python | Bindings para Python |
| `napi-rs` | FFI Node | Bindings para Node.js |

## Ferramentas Não-Mencionadas Mas Que Você Vai Querer

- **rustup** — gerenciador de toolchains.
- **rustfmt** — formatador padrão.
- **clippy** — linter genial. Use sempre.
- **rust-analyzer** — IDE backend (LSP). Funciona em VSCode, Vim, Emacs, JetBrains.
- **miri** — interpretador para detectar UB em testes.
- **cargo-audit** — vulnerabilidades em dependências.
- **cargo-edit** — `cargo add/rm/upgrade`.
- **cargo-watch** — re-rodar em mudança.
- **cargo-expand** — expandir macros para inspeção.
- **cargo-llvm-lines** / **cargo-bloat** — entender tempo de compilação.
- **bacon** — background runner com UI minimalista.

## Casos de Estudo Reais

- **[Pingora @ Cloudflare](https://blog.cloudflare.com/pingora-open-source/)** — substituindo NGINX em Rust.
- **[Discord — Why Rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)** — migração de Go.
- **[Dropbox — Rewriting Sync Engine in Rust](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine)**.
- **[Mozilla Servo](https://servo.org/)** — projeto que originou Rust (em hibernação ativa).
- **[Firecracker @ AWS](https://firecracker-microvm.github.io/)** — microVM que roda Lambda.
- **[Linux Kernel — Rust for Linux](https://rust-for-linux.com/)**.

## Inspiração

- **[The Whole and the Part](https://github.com/Felipeness/the-whole-and-the-part)** — Felipe Ness. O livro que inspirou este. Filosofia holonômica aplicada a software.
- **[The Ghost in the Machine](https://en.wikipedia.org/wiki/The_Ghost_in_the_Machine)** — Arthur Koestler. Onde a ideia de holon nasceu.
- **[A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php)** — John Ousterhout. Onde *deep modules* foi articulado.

---

> *"Os mortos governam os vivos."* — Auguste Comte
>
> Em software, os bugs do passado governam as linguagens do futuro. Rust honra seus mortos.

[← Voltar ao sumário](SUMMARY.md)
