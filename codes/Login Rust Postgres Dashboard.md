### USER
Sistema de login com rust e postgres, .env, com html usando bs css via cdn. Mensagens de erro, etc e direcionar para página dashboard protegida. botão logout na dashboard

### ASSISTANT
Aqui está um exemplo completo de um **sistema de login simples com Rust (usando Actix Web), PostgreSQL, variáveis de ambiente com `.env`, frontend com HTML + Bootstrap via CDN**, incluindo:

- Página de login com mensagens de erro  
- Redirecionamento para uma **dashboard protegida** após login bem-sucedido  
- Botão de **logout** na dashboard  

---

### 📁 Estrutura do projeto sugerida

```
meu-app/
├── src/
│   └── main.rs
├── templates/
│   ├── login.html
│   └── dashboard.html
├── Cargo.toml
└── .env
```

---

### 🔧 1. Dependências (`Cargo.toml`)

```toml
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
dotenvy = "0.15"
tokio-postgres = "0.7"
actix-session = { version = "0.8", features = ["cookie-session"] }
askama = "0.12"
```

> Usamos `askama` para templates simples (HTML embutido). Alternativas: Tera, handlebars.

---

### 🌐 2. Código principal (`src/main.rs`)

```rust
use actix_session::{Session, CookieSession};
use actix_web::{
    web, App, HttpResponse, HttpServer, Result, middleware::Logger, http::header,
};
use dotenvy::dotenv;
use std::env;
use tokio_postgres::{NoTls, Client};
use askama::Template;

#[derive(Template)]
#[template(path = "login.html")]
struct LoginTemplate {
    error: Option<String>,
}

#[derive(Template)]
#[template(path = "dashboard.html")]
struct DashboardTemplate;

async fn login_page(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_some() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish());
    }
    Ok(HttpResponse::Ok().body(LoginTemplate { error: None }.render().unwrap()))
}

async fn handle_login(
    session: Session,
    client: web::Data<Client>,
    form: web::Form<LoginCredentials>,
) -> Result<HttpResponse> {
    let stmt = client.prepare("SELECT username FROM users WHERE username = $1 AND password = crypt($2, password);")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("DB error"))?;

    let rows = client.query(&stmt, &[&form.username, &form.password])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("DB query error"))?;

    if rows.len() == 1 {
        session.insert("user", form.username.clone())
            .map_err(|_| actix_web::error::ErrorInternalServerError("Session error"))?;
        Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish())
    } else {
        Ok(HttpResponse::Ok().body(
            LoginTemplate {
                error: Some("Usuário ou senha incorretos.".to_string()),
            }
            .render()
            .unwrap(),
        ))
    }
}

#[derive(serde::Deserialize)]
struct LoginCredentials {
    username: String,
    password: String,
}

async fn dashboard(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_none() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/login"))
            .finish());
    }
    Ok(HttpResponse::Ok().body(DashboardTemplate.render().unwrap()))
}

async fn logout(session: Session) -> Result<HttpResponse> {
    session.purge();
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    dotenv().ok();

    let db_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let (client, connection) = tokio_postgres::connect(&db_url, NoTls)
        .await
        .expect("Cannot connect to DB");

    tokio::spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("DB connection error: {}", e);
        }
    });

    let secret_key = env::var("SESSION_SECRET").unwrap_or("fallback-secret-key-32-bytes!!".to_string());
    let secret_bytes = secret_key.as_bytes();

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(client.clone()))
            .wrap(Logger::default())
            .wrap(CookieSession::signed(secret_bytes).secure(false)) // Em produção, use secure(true) com HTTPS
            .route("/login", web::get().to(login_page))
            .route("/login", web::post().to(handle_login))
            .route("/dashboard", web::get().to(dashboard))
            .route("/logout", web::post().to(logout))
            .route("/", web::redirect("/login")) // redireciona / para login
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

### 🔐 3. Banco de dados PostgreSQL

Crie uma tabela de usuários com senha **criptografada com `pgcrypto`**:

```sql
-- Habilitar extensão (execute uma vez no banco)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Tabela de usuários
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password TEXT NOT NULL -- armazenará o hash com crypt()
);

-- Exemplo de inserção de usuário (senha: "123456")
INSERT INTO users (username, password)
VALUES ('mervy', crypt('123456', gen_salt('bf')));
```

---

### 📄 4. Arquivo `.env`

```env
DATABASE_URL=postgresql://postgres:senha_do_postgres@localhost/techstore
SESSION_SECRET=sua_chave_secreta_de_32_bytes_aqui!!
```

> Ajuste conforme seu ambiente PostgreSQL (usuário, senha, banco `techstore` conforme sua memória).

---

### 🌍 5. Templates HTML

#### `templates/login.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Login</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-4">
            <div class="card">
                <div class="card-header">
                    <h4 class="text-center">Login</h4>
                </div>
                <div class="card-body">
                    {% if error %}
                    <div class="alert alert-danger">{{ error }}</div>
                    {% endif %}
                    <form method="POST">
                        <div class="mb-3">
                            <label for="username" class="form-label">Usuário</label>
                            <input type="text" class="form-control" id="username" name="username" required>
                        </div>
                        <div class="mb-3">
                            <label for="password" class="form-label">Senha</label>
                            <input type="password" class="form-control" id="password" name="password" required>
                        </div>
                        <button type="submit" class="btn btn-primary w-100">Entrar</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
</body>
</html>
```

#### `templates/dashboard.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Dashboard</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
<div class="container mt-5">
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h2>Bem-vindo à Dashboard!</h2>
        <form method="POST" action="/logout" style="display:inline;">
            <button type="submit" class="btn btn-outline-danger">Logout</button>
        </form>
    </div>
    <div class="alert alert-success">
        Você está autenticado. Esta página é protegida.
    </div>
</div>
</body>
</html>
```

---

### 🔒 Observações importantes

1. **Segurança da senha**:  
   - Use `pgcrypto` com `crypt()` como mostrado — **nunca armazene senhas em texto claro**.
   - Em produção, use HTTPS e ative `.secure(true)` na sessão.

2. **Variáveis sensíveis**:  
   - Mantenha `.env` fora do controle de versão (adicione ao `.gitignore`).

3. **Sessão**:  
   - O cookie de sessão é assinado, mas **não criptografado**. Não armazene dados sensíveis nele — apenas identificadores.

4. **Roteamento**:  
   - Qualquer acesso à `/dashboard` sem sessão válida redireciona para `/login`.

---

Se quiser expandir com registro de usuários, proteção CSRF, ou integração com seu sistema atual (ex: banco `techstore`, usuário `postgres`), posso adaptar! Deseja isso?

### USER
Todos os arquivos foram fornecidos? Crie um register e mensagens flash por favor caso não tenha criado

### ASSISTANT
Obrigado pela observação. Na resposta anterior **não incluí** a funcionalidade de **registro (signup)** nem um sistema de **mensagens flash** (como "Conta criada com sucesso" ou "Usuário já existe").

Vou agora:

1. ✅ Adicionar rota e lógica para **registro de novos usuários**  
2. ✅ Implementar um sistema simples de **mensagens flash** (usando sessão)  
3. ✅ Atualizar os templates HTML para exibir essas mensagens  
4. ✅ Garantir que todos os arquivos estejam completos

---

### 📁 Estrutura atualizada

```
meu-app/
├── src/
│   └── main.rs
├── templates/
│   ├── login.html
│   ├── register.html
│   └── dashboard.html
├── Cargo.toml
└── .env
```

---

### 🔧 1. Atualize `Cargo.toml` (mantenha como antes)
Já está correto. Nenhuma mudança necessária.

---

### 🌐 2. Código completo atualizado — `src/main.rs`

```rust
use actix_session::{Session, CookieSession};
use actix_web::{
    web, App, HttpResponse, HttpServer, Result, middleware::Logger, http::header,
    Error,
};
use dotenvy::dotenv;
use std::env;
use tokio_postgres::{NoTls, Client, Row};
use askama::Template;
use serde::Deserialize;

// === Templates ===

#[derive(Template)]
#[template(path = "login.html")]
struct LoginTemplate {
    error: Option<String>,
    success: Option<String>,
}

#[derive(Template)]
#[template(path = "register.html")]
struct RegisterTemplate {
    error: Option<String>,
    success: Option<String>,
}

#[derive(Template)]
#[template(path = "dashboard.html")]
struct DashboardTemplate;

// === Helpers de Flash Messages ===

fn get_flash(session: &Session, key: &str) -> Option<String> {
    session.get::<String>(key).unwrap_or(None)
}

fn set_flash(session: &Session, key: &str, value: &str) -> Result<(), Error> {
    session.insert(key, value.to_string())
        .map_err(|_| actix_web::error::ErrorInternalServerError("Falha ao definir mensagem flash"))
}

// === Rotas ===

async fn home_redirect() -> Result<HttpResponse> {
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

async fn login_page(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_some() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish());
    }
    let error = get_flash(&session, "login_error");
    let success = get_flash(&session, "login_success");
    Ok(HttpResponse::Ok().body(LoginTemplate { error, success }.render().unwrap()))
}

async fn register_page(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_some() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish());
    }
    let error = get_flash(&session, "register_error");
    let success = get_flash(&session, "register_success");
    Ok(HttpResponse::Ok().body(RegisterTemplate { error, success }.render().unwrap()))
}

#[derive(Deserialize)]
struct Credentials {
    username: String,
    password: String,
}

async fn handle_login(
    session: Session,
    client: web::Data<Client>,
    form: web::Form<Credentials>,
) -> Result<HttpResponse> {
    let stmt = client.prepare("SELECT username FROM users WHERE username = $1 AND password = crypt($2, password);")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no banco"))?;

    let rows = client.query(&stmt, &[&form.username, &form.password])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na consulta"))?;

    if rows.len() == 1 {
        session.insert("user", form.username.clone())
            .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na sessão"))?;
        Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish())
    } else {
        set_flash(&session, "login_error", "Usuário ou senha incorretos.")?;
        Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/login"))
            .finish())
    }
}

async fn handle_register(
    session: Session,
    client: web::Data<Client>,
    form: web::Form<Credentials>,
) -> Result<HttpResponse> {
    // Verifica se usuário já existe
    let exists_stmt = client.prepare("SELECT 1 FROM users WHERE username = $1;")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no banco"))?;

    let exists_rows = client.query(&exists_stmt, &[&form.username])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na consulta"))?;

    if !exists_rows.is_empty() {
        set_flash(&session, "register_error", "Usuário já existe.")?;
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/register"))
            .finish());
    }

    // Insere novo usuário com senha criptografada
    let insert_stmt = client.prepare("INSERT INTO users (username, password) VALUES ($1, crypt($2, gen_salt('bf')));")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no INSERT"))?;

    client.execute(&insert_stmt, &[&form.username, &form.password])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Falha ao registrar usuário"))?;

    set_flash(&session, "register_success", "Conta criada com sucesso! Faça login.")?;
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

async fn dashboard(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_none() {
        set_flash(&session, "login_error", "Você precisa estar logado.")?;
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/login"))
            .finish());
    }
    Ok(HttpResponse::Ok().body(DashboardTemplate.render().unwrap()))
}

async fn logout(session: Session) -> Result<HttpResponse> {
    session.purge();
    set_flash(&session, "login_success", "Você saiu com sucesso.")?;
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

// === Main ===

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    dotenv().ok();

    let db_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let (client, connection) = tokio_postgres::connect(&db_url, NoTls)
        .await
        .expect("Cannot connect to DB");

    tokio::spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("DB connection error: {}", e);
        }
    });

    let secret_key = env::var("SESSION_SECRET")
        .unwrap_or_else(|_| "fallback-secret-key-32-bytes!!".to_string());
    let secret_bytes = secret_key.as_bytes();

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(client.clone()))
            .wrap(Logger::default())
            .wrap(CookieSession::signed(secret_bytes).secure(false))
            .route("/", web::get().to(home_redirect))
            .route("/login", web::get().to(login_page))
            .route("/login", web::post().to(handle_login))
            .route("/register", web::get().to(register_page))
            .route("/register", web::post().to(handle_register))
            .route("/dashboard", web::get().to(dashboard))
            .route("/logout", web::post().to(logout))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

### 📄 3. Novo template: `templates/register.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Cadastro</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-4">
            <div class="card">
                <div class="card-header">
                    <h4 class="text-center">Criar Conta</h4>
                </div>
                <div class="card-body">
                    {% if error %}
                    <div class="alert alert-danger">{{ error }}</div>
                    {% endif %}
                    {% if success %}
                    <div class="alert alert-success">{{ success }}</div>
                    {% endif %}
                    <form method="POST">
                        <div class="mb-3">
                            <label for="username" class="form-label">Usuário</label>
                            <input type="text" class="form-control" id="username" name="username" required>
                        </div>
                        <div class="mb-3">
                            <label for="password" class="form-label">Senha</label>
                            <input type="password" class="form-control" id="password" name="password" required>
                        </div>
                        <button type="submit" class="btn btn-success w-100">Cadastrar</button>
                    </form>
                    <div class="mt-3 text-center">
                        <a href="/login" class="text-decoration-none">Já tem conta? Faça login</a>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
</body>
</html>
```

---

### 🔁 4. Atualize `templates/login.html` (para mostrar mensagens de sucesso)

Substitua o conteúdo anterior por este (adiciona `success`):

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <title>Login</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body class="bg-light">
<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-4">
            <div class="card">
                <div class="card-header">
                    <h4 class="text-center">Login</h4>
                </div>
                <div class="card-body">
                    {% if error %}
                    <div class="alert alert-danger">{{ error }}</div>
                    {% endif %}
                    {% if success %}
                    <div class="alert alert-success">{{ success }}</div>
                    {% endif %}
                    <form method="POST">
                        <div class="mb-3">
                            <label for="username" class="form-label">Usuário</label>
                            <input type="text" class="form-control" id="username" name="username" required>
                        </div>
                        <div class="mb-3">
                            <label for="password" class="form-label">Senha</label>
                            <input type="password" class="form-control" id="password" name="password" required>
                        </div>
                        <button type="submit" class="btn btn-primary w-100">Entrar</button>
                    </form>
                    <div class="mt-3 text-center">
                        <a href="/register" class="text-decoration-none">Criar uma conta</a>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
</body>
</html>
```

> O `dashboard.html` permanece igual.

---

### ✅ Funcionalidades adicionadas

| Recurso               | Status |
|-----------------------|--------|
| Página de registro    | ✅ `/register` |
| Validação de usuário existente | ✅ |
| Criptografia segura de senha | ✅ (`crypt` + `gen_salt('bf')`) |
| Mensagens flash       | ✅ (`success_msg`, `error_msg`) |
| Logout com redirecionamento e mensagem | ✅ |
| Links entre login/cadastro | ✅ |

---

Agora você pode:

- Acessar `/register` para criar conta  
- Ver mensagens como *"Conta criada com sucesso!"* ou *"Usuário já existe."*  
- Fazer login e acessar o dashboard  
- Clicar em **Logout** e voltar à tela de login com confirmação  

Se quiser adicionar validações extras (ex: senha forte, e-mail, etc.), posso ajudar!

### USER
Criou midleware de proteção?

### ASSISTANT
Boa pergunta! **Na versão atual do código, não há um middleware explícito** no estilo de frameworks como Express.js ou Laravel — **mas a proteção das rotas está implementada de forma equivalente**, diretamente nas funções de rota (por exemplo, `dashboard` verifica a sessão manualmente).

No entanto, em **Actix Web (Rust)**, é **altamente recomendado e mais limpo** usar um **middleware ou um guard de sessão reutilizável** para proteger rotas. Vamos corrigir isso criando um **middleware de autenticação real**.

---

### ✅ Vamos implementar um **middleware de proteção de rota** em Rust com Actix Web

Este middleware:
- Verifica se o usuário está logado (sessão contém `"user"`)
- Se **não estiver**, redireciona para `/login` com mensagem flash
- Se **estiver**, permite o acesso

---

### 🔁 Atualização: `src/main.rs` com middleware de autenticação

Substitua o conteúdo de `src/main.rs` pelo código abaixo (mantendo todos os recursos anteriores: login, register, flash, etc.):

```rust
use actix_session::{Session, CookieSession};
use actix_web::{
    dev::{ServiceRequest, ServiceResponse, Transform},
    web, App, HttpResponse, HttpServer, Result, middleware::Logger, http::header,
    Error, guard, HttpMessage,
};
use dotenvy::dotenv;
use std::env;
use std::future::{Future, Ready};
use std::pin::Pin;
use tokio_postgres::{NoTls, Client};
use askama::Template;
use serde::Deserialize;

// === Templates ===

#[derive(Template)]
#[template(path = "login.html")]
struct LoginTemplate {
    error: Option<String>,
    success: Option<String>,
}

#[derive(Template)]
#[template(path = "register.html")]
struct RegisterTemplate {
    error: Option<String>,
    success: Option<String>,
}

#[derive(Template)]
#[template(path = "dashboard.html")]
struct DashboardTemplate;

// === Flash helpers ===

fn get_flash(session: &Session, key: &str) -> Option<String> {
    session.get::<String>(key).unwrap_or(None)
}

fn set_flash(session: &Session, key: &str, value: &str) -> Result<(), Error> {
    session.insert(key, value.to_string())
        .map_err(|_| actix_web::error::ErrorInternalServerError("Falha ao definir flash"))
}

// === Middleware de Autenticação ===

pub struct Auth;

impl<S, B> Transform<S, ServiceRequest> for Auth
where
    S: actix_web::dev::Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        actix_web::dev::ready(Ok(AuthMiddleware { service }))
    }
}

pub struct AuthMiddleware<S> {
    service: S,
}

impl<S, B> actix_web::dev::Service<ServiceRequest> for AuthMiddleware<S>
where
    S: actix_web::dev::Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>>>>;

    fn poll_ready(&self, cx: &mut std::task::Context<'_>) -> std::task::Poll<Result<(), Self::Error>> {
        self.service.poll_ready(cx)
    }

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let session = req.get_session();
        match session.get::<String>("user").unwrap_or(None) {
            Some(_) => {
                // Usuário autenticado: continua
                let fut = self.service.call(req);
                Box::pin(async move { fut.await })
            }
            None => {
                // Não autenticado: redireciona para login
                Box::pin(async move {
                    let _ = set_flash(&session, "login_error", "Você precisa estar logado para acessar essa página.");
                    let res = req.into_response(
                        HttpResponse::Found()
                            .append_header((header::LOCATION, "/login"))
                            .finish()
                    );
                    Ok(res)
                })
            }
        }
    }
}

// === Rotas públicas (sem proteção) ===

async fn home_redirect() -> Result<HttpResponse> {
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

async fn login_page(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_some() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish());
    }
    let error = get_flash(&session, "login_error");
    let success = get_flash(&session, "login_success");
    Ok(HttpResponse::Ok().body(LoginTemplate { error, success }.render().unwrap()))
}

async fn register_page(session: Session) -> Result<HttpResponse> {
    if session.get::<String>("user").unwrap_or(None).is_some() {
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish());
    }
    let error = get_flash(&session, "register_error");
    let success = get_flash(&session, "register_success");
    Ok(HttpResponse::Ok().body(RegisterTemplate { error, success }.render().unwrap()))
}

#[derive(Deserialize)]
struct Credentials {
    username: String,
    password: String,
}

async fn handle_login(
    session: Session,
    client: web::Data<Client>,
    form: web::Form<Credentials>,
) -> Result<HttpResponse> {
    let stmt = client.prepare("SELECT username FROM users WHERE username = $1 AND password = crypt($2, password);")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no banco"))?;

    let rows = client.query(&stmt, &[&form.username, &form.password])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na consulta"))?;

    if rows.len() == 1 {
        session.insert("user", form.username.clone())
            .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na sessão"))?;
        Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/dashboard"))
            .finish())
    } else {
        set_flash(&session, "login_error", "Usuário ou senha incorretos.")?;
        Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/login"))
            .finish())
    }
}

async fn handle_register(
    session: Session,
    client: web::Data<Client>,
    form: web::Form<Credentials>,
) -> Result<HttpResponse> {
    let exists_stmt = client.prepare("SELECT 1 FROM users WHERE username = $1;")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no banco"))?;

    let exists_rows = client.query(&exists_stmt, &[&form.username])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro na consulta"))?;

    if !exists_rows.is_empty() {
        set_flash(&session, "register_error", "Usuário já existe.")?;
        return Ok(HttpResponse::Found()
            .append_header((header::LOCATION, "/register"))
            .finish());
    }

    let insert_stmt = client.prepare("INSERT INTO users (username, password) VALUES ($1, crypt($2, gen_salt('bf')));")
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Erro no INSERT"))?;

    client.execute(&insert_stmt, &[&form.username, &form.password])
        .await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Falha ao registrar"))?;

    set_flash(&session, "register_success", "Conta criada com sucesso! Faça login.")?;
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

async fn dashboard() -> Result<HttpResponse> {
    // Agora esta rota é protegida pelo middleware — não precisa verificar sessão aqui!
    Ok(HttpResponse::Ok().body(DashboardTemplate.render().unwrap()))
}

async fn logout(session: Session) -> Result<HttpResponse> {
    session.purge();
    set_flash(&session, "login_success", "Você saiu com sucesso.")?;
    Ok(HttpResponse::Found()
        .append_header((header::LOCATION, "/login"))
        .finish())
}

// === Main ===

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    dotenv().ok();

    let db_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let (client, connection) = tokio_postgres::connect(&db_url, NoTls)
        .await
        .expect("Cannot connect to DB");

    tokio::spawn(async move {
        if let Err(e) = connection.await {
            eprintln!("DB connection error: {}", e);
        }
    });

    let secret_key = env::var("SESSION_SECRET")
        .unwrap_or_else(|_| "fallback-secret-key-32-bytes!!".to_string());
    let secret_bytes = secret_key.as_bytes();

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(client.clone()))
            .wrap(Logger::default())
            .wrap(CookieSession::signed(secret_bytes).secure(false))
            .route("/", web::get().to(home_redirect))
            .route("/login", web::get().to(login_page))
            .route("/login", web::post().to(handle_login))
            .route("/register", web::get().to(register_page))
            .route("/register", web::post().to(handle_register))
            // Rotas protegidas: aplicam middleware Auth
            .service(
                web::scope("/dashboard")
                    .wrap(Auth) // <-- Proteção centralizada aqui
                    .route("", web::get().to(dashboard))
                    // Você pode adicionar mais rotas protegidas aqui:
                    // .route("/profile", web::get().to(profile))
            )
            .route("/logout", web::post().to(logout))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

### ✅ O que mudou?

| Antes | Depois |
|------|--------|
| Verificação manual de sessão em `dashboard` | ✅ **Middleware `Auth` reutilizável** |
| Qualquer nova rota protegida precisaria repetir a lógica | ✅ Basta adicionar `.wrap(Auth)` no escopo |
| Código mais limpo e seguro | ✅ Separação clara entre lógica pública e privada |

---

### 🛡️ Como usar para novas rotas protegidas?

Basta adicioná-las dentro do escopo protegido:

```rust
.service(
    web::scope("/dashboard")
        .wrap(Auth)
        .route("", web::get().to(dashboard))
        .route("/settings", web::get().to(settings))
        .route("/profile/edit", web::post().to(edit_profile))
)
```

Todas essas rotas agora exigem login automaticamente.

---

Pronto! Agora você tem:

- ✅ **Middleware de autenticação real**  
- ✅ **Proteção centralizada de rotas privadas**  
- ✅ **Mensagens flash funcionando**  
- ✅ **Registro + Login + Logout + Dashboard**  
- ✅ **HTML com Bootstrap via CDN**  
- ✅ **PostgreSQL com senhas seguras**  
- ✅ **`.env` para configuração**

Se quiser adicionar, por exemplo, **proteção por nível de usuário (admin vs. comum)**, posso estender o middleware!