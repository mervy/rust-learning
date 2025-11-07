```rust
//Cargo.toml
[package]
name = "strong-password-generator"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
clap = { version = "4.5", features = ["derive"] }

```
```rust
//src/main.rs
use clap::Parser;
use rand::seq::SliceRandom;
use rand::thread_rng;

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    /// Comprimento da senha (mínimo: 4)
    #[arg(default_value_t = 16)]
    length: usize,
}

fn generate_strong_password(length: usize) -> Result<String, &'static str> {
    if length < 4 {
        return Err("O comprimento da senha deve ser pelo menos 4.");
    }

    let mut rng = thread_rng();

    let uppercase = b"ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    let lowercase = b"abcdefghijklmnopqrstuvwxyz";
    let digits = b"0123456789";
    let symbols = b"!@#$%^&*()_+-=[]{}|;:,.<>?";

    let mut password = Vec::with_capacity(length);

    // Garantir um de cada tipo
    password.push(*uppercase.choose(&mut rng).unwrap());
    password.push(*lowercase.choose(&mut rng).unwrap());
    password.push(*digits.choose(&mut rng).unwrap());
    password.push(*symbols.choose(&mut rng).unwrap());

    // Restante: mistura de todos os caracteres
    let all_chars = [
        uppercase.as_slice(),
        lowercase.as_slice(),
        digits.as_slice(),
        symbols.as_slice(),
    ]
    .concat();

    for _ in 4..length {
        password.push(*all_chars.choose(&mut rng).unwrap());
    }

    // Embaralhar para evitar padrões
    password.shuffle(&mut rng);

    String::from_utf8(password).map_err(|_| "Erro ao converter senha para UTF-8")
}

fn main() {
    let cli = Cli::parse();

    match generate_strong_password(cli.length) {
        Ok(password) => println!("Senha forte gerada: {}", password),
        Err(e) => {
            eprintln!("Erro: {}", e);
            std::process::exit(1);
        }
    }
}
```

### Uso
```bash
cargo run #gera com 16 caracteres
cargo run -- 20 #gera com 20 caracteres
```

Depois de testar com cargo run, você pode compilar uma versão otimizada:

```bash
cargo build --release
```

E executar com:

```bash
./target/release/strong-password-generator 24
```