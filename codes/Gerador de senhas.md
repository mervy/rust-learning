### Exemplo em Rust — gerador de senhas fortes
```rust
//Cargo.tml
[package]
name = "senha_forte"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
```
```rust
//src/main.rs
use rand::rngs::OsRng;
use rand::distributions::{Distribution, Uniform};
use rand::seq::SliceRandom;
use std::env;

fn generate_password(length: usize, avoid_ambiguous: bool) -> String {
    // Conjuntos de caracteres
    let lower = "abcdefghijklmnopqrstuvwxyz";
    let upper = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    let digits = "0123456789";
    let symbols = "!@#$%^&*()-_=+[]{};:,.<>/?";

    // caracteres ambíguos que podemos evitar (opcional)
    let ambiguous = "Il1O0";

    // Constrói o alfabeto final, possivelmente removendo ambíguos
    let mut allowed = String::new();
    allowed.push_str(lower);
    allowed.push_str(upper);
    allowed.push_str(digits);
    allowed.push_str(symbols);

    if avoid_ambiguous {
        allowed.retain(|c| !ambiguous.contains(c));
    }

    // Se o comprimento for muito pequeno, adaptamos para garantir categorias
    let mut rng = OsRng;

    // Se length < 4, ainda assim geramos (mas pode não conter todas as categorias)
    let mut chars: Vec<char> = Vec::with_capacity(length);

    // Garantir pelo menos uma de cada categoria quando possível
    if length >= 1 {
        chars.push(*lower.chars().collect::<Vec<_>>().choose(&mut rng).unwrap());
    }
    if length >= 2 {
        chars.push(*upper.chars().collect::<Vec<_>>().choose(&mut rng).unwrap());
    }
    if length >= 3 {
        chars.push(*digits.chars().collect::<Vec<_>>().choose(&mut rng).unwrap());
    }
    if length >= 4 {
        // escolher símbolo disponível (respeitando ambiguous filter se ativo)
        let syms: String = symbols
            .chars()
            .filter(|c| !avoid_ambiguous || !ambiguous.contains(*c))
            .collect();
        if !syms.is_empty() {
            chars.push(*syms.chars().collect::<Vec<_>>().choose(&mut rng).unwrap());
        }
    }

    // Preenche o restante com caracteres aleatórios do conjunto permitido
    let allowed_chars: Vec<char> = allowed.chars().collect();
    if allowed_chars.is_empty() {
        panic!("Conjunto de caracteres permitido ficou vazio (verifique avoid_ambiguous).");
    }

    let between = Uniform::from(0..allowed_chars.len());
    while chars.len() < length {
        let idx = between.sample(&mut rng);
        chars.push(allowed_chars[idx]);
    }

    // Embaralha para remover padrão previsível
    chars.shuffle(&mut rng);

    chars.into_iter().collect()
}

fn print_usage(program: &str) {
    eprintln!("Uso: {} [comprimento] [--no-ambiguous]", program);
    eprintln!("Exemplos:");
    eprintln!("  {} 16          # gera senha de 16 caracteres", program);
    eprintln!("  {} 12 --no-ambiguous  # evita I l 1 O 0", program);
}

fn main() {
    let args: Vec<String> = env::args().collect();
    let program = &args[0];

    if args.len() == 1 {
        print_usage(program);
        return;
    }

    // Parse básico
    let length_arg = args.get(1).and_then(|s| s.parse::<usize>().ok());
    let length = length_arg.unwrap_or_else(|| {
        eprintln!("Comprimento inválido, usando 16 por padrão.");
        16
    });

    let avoid_ambiguous = args.iter().any(|s| s == "--no-ambiguous" || s == "--no-amb");

    if length == 0 {
        eprintln!("Comprimento deve ser maior que zero.");
        return;
    }

    let password = generate_password(length, avoid_ambiguous);
    println!("{}", password);
}
```
`
Como usar
```bash
cargo run --release -- 16 # gera uma senha de 16 caracteres.
```bash
```bash
cargo run --release -- 12 --no-ambiguous #evita caracteres ambíguos.
```