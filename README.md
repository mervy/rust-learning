# 🦀 Rust — Guia, História e Recursos

## 📖 História

O **Rust** é uma linguagem de programação moderna focada em **segurança**, **velocidade** e **concorrência**.
Ela nasceu em **2006**, como um projeto pessoal de **Graydon Hoare**, engenheiro da **Mozilla Research**, que queria criar uma linguagem que unisse o desempenho de linguagens de baixo nível (como C e C++) com a segurança e expressividade de linguagens modernas.

A **Mozilla** passou a apoiar oficialmente o projeto em **2009**, e em **2015** foi lançado o **Rust 1.0**, a primeira versão estável.
Desde então, o Rust tem se destacado por sua eficiência em sistemas embarcados, aplicações web (via WebAssembly), ferramentas de linha de comando, e até mesmo em sistemas operacionais.

Hoje, o desenvolvimento é mantido pela **Rust Foundation**, uma organização independente que conta com o apoio de empresas como **Amazon, Microsoft, Google, Huawei e Dropbox**.

---

## ⚙️ Instalação

### 🐧 Linux

```bash
# Atualize os pacotes
sudo apt update && sudo apt install curl build-essential -y

# Baixe e instale o Rust via rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Adicione o cargo ao PATH (caso não adicione automaticamente)
source $HOME/.cargo/env

# Verifique a instalação
rustc --version
cargo --version
```

> 💡 O comando `rustup` gerencia versões do Rust, o compilador `rustc` compila seu código, e o `cargo` é o gerenciador de pacotes e build system da linguagem.

---

### 🪟 Windows

1. Baixe o instalador oficial em:
   👉 [https://win.rustup.rs](https://win.rustup.rs)

2. Execute o instalador e siga as instruções (mantenha as opções padrão).

3. Após a instalação, feche e reabra o terminal (PowerShell ou CMD) e verifique:

```powershell
rustc --version
cargo --version
```

> 💡 No Windows, o `rustup` instala automaticamente o **Cargo**, o **Rust Compiler (rustc)** e o **Clippy** (para linting).

---

## 🚀 Como usar

Crie e rode um projeto simples:

```bash
cargo new hello-rust
cd hello-rust
cargo run
```

> 🦀 O Cargo cria a estrutura completa do projeto, incluindo o `main.rs`, o manifesto `Cargo.toml` e o diretório `src`.

---

## 📚 Recursos e Cheatsheets

Aqui estão alguns dos melhores materiais para consulta rápida e aprendizado contínuo de Rust:

* [Rust — A Beginner Cheat Sheet](https://medium.com/codex/rust-a-beginner-cheat-sheet-8fd7b0ce49de)
* [DevSheets - Rust](https://www.devsheets.io/sheets/rust)
* [Rust Cheat Sheet (DEV.to)](https://dev.to/elmyrockers/rust-cheat-sheet-25i1)
* [Rust Language Cheat Sheet (PDF)](https://cheats.rs/dl/rust_cheat_sheet_a4.pdf)
* [Rust Cheatsheet (phaiax.github.io)](https://phaiax.github.io/rust-cheatsheet/)
* [Rust Cheat Sheet (Zero To Mastery)](https://zerotomastery.io/cheatsheets/rust-cheat-sheet/)
* [Rust SpeedSheet](https://speedsheet.io/s/rust#h53m)
* [Rust cheatsheet (QuickRef)](https://quickref.me/rust.html)
* [Rust Language Cheat Sheet (cheats.rs)](https://cheats.rs/)

---

## 💡 Dica Final

> “O Rust não é apenas sobre evitar bugs — é sobre **pensar diferente** sobre como escrever código seguro e eficiente.”

Se você está começando, experimente o playground online:
👉 [https://play.rust-lang.org](https://play.rust-lang.org)

---

🦀 **Rust — Performance, Reliability, Productivity. Pick Three.**
