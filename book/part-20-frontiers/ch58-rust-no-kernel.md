<a id="capitulo-58"></a>
# Capítulo 58: Rust no Kernel — Linux e Windows

> *"Rust is the first time in 30 years that we've considered any other language for the kernel."*
> — Linus Torvalds, 2022

> *"Quando o kernel aceita uma linguagem nova, alguma coisa virou."*

## 58.1 O Improvável Aconteceu

Em 2022, Linus Torvalds aceitou no Linux 6.1 o suporte oficial a Rust como segunda linguagem do kernel. Foi a primeira vez em três décadas que o kernel aceitou *qualquer* linguagem além de C.

Isso não foi gesto simbólico. Foi reconhecimento de uma crise: 70% das vulnerabilidades sérias em qualquer kernel grande (Linux, Windows, macOS) são de memória — e nenhuma quantidade de revisão humana, sanitizers, fuzzers ou processo conseguiu reduzir esse número significativamente em 30 anos.

A aposta: se o compilador *recusa* compilar código com classes inteiras desses bugs, talvez consigamos construir kernels onde o ataque de memória deixa de ser o vetor dominante.

## 58.2 Rust for Linux

O projeto **Rust for Linux** começou em 2020, liderado por Miguel Ojeda. A integração tem várias frentes:

- **`kernel` crate**: bindings tipados de subsistemas C do kernel para uso em Rust.
- **Drivers em Rust**: novos drivers podem ser escritos puramente em Rust, falando com os subsistemas via `kernel::*`.
- **Subsystems incrementais**: alguns subsystems estão sendo gradualmente reescritos. Outros expõem pontos de extensão.

Drivers já mainline em Rust (parcial ou totalmente):
- **Apple AGX GPU driver**: Asahi Linux para Macs Apple Silicon.
- **NVIDIA Open GPU Driver**: parcialmente.
- **PuzzleFS**: filesystem experimental.
- **Rust binder driver**: comunicação inter-processo do Android.

## 58.3 As Restrições do Kernel

Rust no kernel não é Rust normal. As restrições obrigatórias:

**1. `no_std` total.** Sem `std`, sem `alloc` padrão. O kernel tem seu próprio allocator (`kernel::alloc`) que retorna `Result` em vez de panicar.

**2. Sem `panic!` em produção.** O kernel não pode parar. Code paths que panicariam em userspace precisam ser provados impossíveis ou tratados com `Result`.

**3. Allocations falíveis.** `Vec::push` não pode panicar em OOM. Em vez disso, kernel usa APIs como `try_push` que retornam `Result`.

**4. Subset de features**. Algumas features de Rust ainda são nightly e o kernel as usa cuidadosamente: `allocator_api`, `coerce_unsized`, `dispatch_from_dyn`, `box_into_inner`. Cada uma tem RFC e processo de estabilização paralelo.

**5. ABI estável com C.** Cada interação com código C existente passa por bindings cuidadosamente revisados.

## 58.4 Por Que o Kernel é o Caso Ideal

Há um paradoxo aparente: o kernel é o lugar mais hostil para uma linguagem high-level, mas também é onde os benefícios de Rust mais se justificam. Por quê?

- **Performance crítica**: kernel não tolera overhead. Rust é zero-cost — sem GC, sem runtime escondido. C-equivalente.
- **Segurança crítica**: bug em kernel é privilege escalation. Toda redução de UB tem valor desproporcional.
- **Vida útil longa**: código de kernel vive décadas. Investir em correção *uma vez* paga por gerações.
- **Concorrência massiva**: cada core, cada interrupt handler é concorrente. Send/Sync de Rust ajuda enormemente.

C foi escolhido nos anos 90 porque era a única opção com performance e portabilidade. Rust é a primeira em 50 anos a igualar isso *e* adicionar segurança.

## 58.5 Windows: A Migração Quieta

Em 2023, David Weston (VP Microsoft Security) anunciou:

> *"We are using Rust to harden the Windows kernel. We have rewritten core kernel libraries in Rust and are shipping them."*

Componentes confirmados em Rust:
- **DWriteCore**: renderização de fontes.
- **Win32k**: subsistema de janelas (parcial).
- **GDI**: graphics device interface (parcial).
- **Bibliotecas de criptografia**.

Microsoft não publica milestones tão visíveis quanto Linux, mas a direção é clara: cada componente novo escrito em Rust, e cada componente antigo sob revisão para reescrita.

## 58.6 Android: Vinte e Um Por Cento

Em 2024, Google reportou que **21% de todo código nativo novo em Android era Rust**. Detalhe importante: durante a migração, a taxa de vulnerabilidades de memória *despencou*.

Citação de Lars Bergstrom, Google:

> *"Android's vulnerabilities have decreased over time as Rust has increased. None of the new code in Rust has been the source of a memory safety CVE."*

Isso é dado, não opinião. Bilhões de dispositivos rodando Android. A correlação entre Rust e queda de CVEs é comprovada estatisticamente.

## 58.7 ChromeOS, Fuchsia e Outros

- **ChromeOS**: Rust em componentes de baixo nível, especialmente VMs (`crosvm`).
- **Fuchsia**: o sistema operacional experimental do Google é primariamente C++, mas tem Rust como cidadão de primeira classe. `Zircon` (microkernel) é C++; muitos serviços ao redor são Rust.
- **Redox OS**: sistema operacional acadêmico inteiramente em Rust. Não produção, mas demonstra viabilidade.
- **Tock OS**: SO embedded em Rust para microcontroladores. Usado em pesquisa e produtos comerciais (Helium hotspots).

## 58.8 O Que o Kernel Precisa Que Rust Ainda Não Dá

Honestidade:

- **`alloc` fallible estável**: em rollout, mas não totalmente estabilizado.
- **GAT em casos avançados**: melhorou muito, mas algumas APIs do kernel ainda batem em limites.
- **`async` no kernel**: experimental. RTIC e Embassy provam que dá para fazer, mas integrar com event loops do kernel é frente aberta.
- **Compile time**: kernel já é grande. Rust adiciona tempo de build. Trabalho em curso (`cranelift` para builds de debug, melhorias em rustc).

Cada um desses tem RFCs ativos. O processo é lento por design — o kernel não pode aceitar features instáveis.

## 58.9 Casos Reais Mensuráveis

| Projeto | Linguagem antiga | Linguagem nova | Resultado |
|---|---|---|---|
| **Cloudflare Pingora** | NGINX (C) | Rust | 1/3 do CPU, 1/3 da RAM, 0 segfaults em 2 anos |
| **Discord Read States** | Go | Rust | 90% redução de latência tail (sem GC) |
| **Microsoft Win32k partial** | C++ | Rust | Classes inteiras de bugs eliminadas |
| **Android nativo novo** | C/C++ | Rust | Zero CVEs de memória até 2024 |
| **Apple AGX driver** | Estaria em C | Rust direto | Reverse engineering bem-sucedido em prazo razoável |

Não são histórias de marketing. São relatórios de equipes que mediram.

## 58.10 O Que Significa Tudo Isso

Quando o sistema operacional aceita uma linguagem nova depois de 30 anos, você está vendo uma virada de era — não em escala de release, mas em escala de geração.

A cada bug eliminado em compile time, a indústria desaprende uma lição amarga. Em 2050, "linguagem de sistemas sem memory safety" pode parecer tão estranho quanto hoje parece "linguagem sem garbage collector para web apps".

Rust acertou o timing histórico. Cinquenta anos depois de C, em vez de mais cinquenta com C, vamos ter cinquenta com Rust — ou com algo derivado das lições que Rust provou possíveis.

---

> *"Quando o kernel sucede, sucede tudo abaixo dele. Rust ganhou o kernel. O resto é tempo."*

[← Capítulo 57 — Embedded e no_std](ch57-embedded-no-std.md) | [Próximo: Capítulo 59 — Compilador por Dentro →](ch59-compilador-internals.md)
