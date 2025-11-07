## You asked:

Sistema de login com rust e postgres, .env, com html usando bs css via cdn. Mensagens de erro, etc e direcionar para página dashboard protegida. botão logout na dashboard

---

## ChatGPT Replied:

Perfeito 👍  
Vamos montar um **projeto completo de login em Rust com PostgreSQL**, com suporte a `.env`, templates HTML com Bootstrap via CDN, mensagens de erro e sessão (login/logout).

* * *

🧩 Estrutura do Projeto
-----------------------

    rust-login/
    ├── Cargo.toml
    ├── .env
    └── src/
        ├── main.rs
        ├── db.rs
        ├── models.rs
        ├── routes.rs
        ├── session.rs
        └── views/
            ├── login.html
            └── dashboard.html
    

* * *

⚙️ Dependências (`Cargo.toml`)
------------------------------

    [package]
    name = "rust-login"
    version = "0.1.0"
    edition = "2021"
    
    [dependencies]
    actix-web = "4"
    actix-session = { version = "0.9", features = ["cookie-session"] }
    actix-rt = "2"
    tokio-postgres = "0.7"
    deadpool-postgres = { version = "0.12", features = ["tokio-runtime"] }
    dotenvy = "0.15"
    serde = { version = "1.0", features = ["derive"] }
    serde_json = "1.0"
    bcrypt = "0.15"
    tera = "1.19"
    

* * *

🌱 `.env`
---------

    DATABASE_URL=postgres://postgres:senha@localhost:5432/rust_login
    SESSION_KEY=uma_chave_bem_segura_aqui
    

* * *

🧠 `src/db.rs`
--------------

    use deadpool_postgres::{Manager, Pool};
    use tokio_postgres::NoTls;
    use std::env;
    
    pub type DBPool = Pool;
    
    pub fn create_pool() -> DBPool {
        let db_url = env::var("DATABASE_URL").expect("DATABASE_URL not set");
        let mgr = Manager::new(db_url, NoTls);
        Pool::builder(mgr).max_size(16).build().unwrap()
    }
    

* * *

👤 `src/models.rs`
------------------

    use serde::Deserialize;
    
    #[derive(Deserialize)]
    pub struct LoginData {
        pub username: String,
        pub password: String,
    }
    

* * *

🔑 `src/session.rs`
-------------------

    use actix_session::Session;
    
    pub fn set_logged_in(session: &Session, username: &str) {
        session.insert("user", username).unwrap();
    }
    
    pub fn is_logged_in(session: &Session) -> bool {
        session.get::<String>("user").unwrap_or(None).is_some()
    }
    
    pub fn logout(session: &Session) {
        session.remove("user");
    }
    

* * *

🧭 `src/routes.rs`
------------------

    use actix_web::{web, HttpResponse, Responder};
    use actix_session::Session;
    use tera::Tera;
    use bcrypt::verify;
    use crate::{db::DBPool, models::LoginData, session};
    
    pub async fn get_login(tmpl: web::Data<Tera>) -> impl Responder {
        let s = tmpl.render("login.html", &tera::Context::new()).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn post_login(
        tmpl: web::Data<Tera>,
        form: web::Form<LoginData>,
        pool: web::Data<DBPool>,
        session: Session,
    ) -> impl Responder {
        let client = pool.get().await.unwrap();
        let row = client
            .query_opt("SELECT username, password FROM users WHERE username=$1", &[&form.username])
            .await
            .unwrap();
    
        let mut ctx = tera::Context::new();
    
        if let Some(row) = row {
            let stored_hash: String = row.get("password");
            if verify(&form.password, &stored_hash).unwrap() {
                session::set_logged_in(&session, &form.username);
                return HttpResponse::Found()
                    .append_header(("Location", "/dashboard"))
                    .finish();
            }
        }
    
        ctx.insert("error", "Usuário ou senha incorretos");
        let s = tmpl.render("login.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn get_dashboard(
        tmpl: web::Data<Tera>,
        session: Session,
    ) -> impl Responder {
        if !session::is_logged_in(&session) {
            return HttpResponse::Found().append_header(("Location", "/login")).finish();
        }
    
        let mut ctx = tera::Context::new();
        ctx.insert("username", &session.get::<String>("user").unwrap().unwrap());
        let s = tmpl.render("dashboard.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn logout(session: Session) -> impl Responder {
        session::logout(&session);
        HttpResponse::Found().append_header(("Location", "/login")).finish()
    }
    

* * *

🚀 `src/main.rs`
----------------

    mod db;
    mod models;
    mod routes;
    mod session;
    
    use actix_web::{web, App, HttpServer};
    use actix_session::SessionMiddleware;
    use actix_web::cookie::Key;
    use dotenvy::dotenv;
    use tera::Tera;
    use std::env;
    
    #[actix_web::main]
    async fn main() -> std::io::Result<()> {
        dotenv().ok();
        let pool = db::create_pool();
        let secret_key = Key::from(env::var("SESSION_KEY").unwrap().as_bytes());
        let tera = Tera::new("src/views/**/*").unwrap();
    
        println!("🚀 Servidor rodando em http://127.0.0.1:8080");
    
        HttpServer::new(move || {
            App::new()
                .app_data(web::Data::new(pool.clone()))
                .app_data(web::Data::new(tera.clone()))
                .wrap(SessionMiddleware::new(
                    actix_session::storage::CookieSessionStore::default(),
                    secret_key.clone(),
                ))
                .service(web::resource("/login").route(web::get().to(routes::get_login)).route(web::post().to(routes::post_login)))
                .service(web::resource("/dashboard").route(web::get().to(routes::get_dashboard)))
                .service(web::resource("/logout").route(web::get().to(routes::logout)))
        })
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
    }
    

* * *

🧱 Banco de Dados (PostgreSQL)
------------------------------

    CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL
    );
    
    -- Criação de um usuário de teste
    INSERT INTO users (username, password)
    VALUES ('admin', '$2b$12$G.9cz...'); -- hash gerado com bcrypt
    

Para gerar o hash:

    cargo install bcrypt-cli
    bcrypt-cli hash
    

* * *

🎨 `src/views/login.html`
-------------------------

    <!DOCTYPE html>
    <html lang="pt-br">
    <head>
        <meta charset="UTF-8">
        <title>Login</title>
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    </head>
    <body class="bg-light d-flex align-items-center" style="height: 100vh;">
        <div class="container text-center">
            <div class="col-md-4 mx-auto bg-white p-4 rounded shadow">
                <h3>Login</h3>
                {% if error %}
                    <div class="alert alert-danger">{{ error }}</div>
                {% endif %}
                <form method="post" action="/login">
                    <div class="mb-3">
                        <input type="text" name="username" class="form-control" placeholder="Usuário" required>
                    </div>
                    <div class="mb-3">
                        <input type="password" name="password" class="form-control" placeholder="Senha" required>
                    </div>
                    <button type="submit" class="btn btn-primary w-100">Entrar</button>
                </form>
            </div>
        </div>
    </body>
    </html>
    

* * *

🖥️ `src/views/dashboard.html`
------------------------------

    <!DOCTYPE html>
    <html lang="pt-br">
    <head>
        <meta charset="UTF-8">
        <title>Dashboard</title>
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    </head>
    <body class="bg-light">
        <nav class="navbar navbar-dark bg-dark p-3">
            <div class="container-fluid">
                <span class="navbar-brand">Bem-vindo, {{ username }}</span>
                <a href="/logout" class="btn btn-outline-light">Logout</a>
            </div>
        </nav>
        <div class="container mt-4">
            <div class="card p-4 shadow-sm">
                <h4>Área restrita</h4>
                <p>Somente usuários logados podem ver esta página.</p>
            </div>
        </div>
    </body>
    </html>
    

* * *

▶️ Executar
-----------

    cargo run
    

Acesse:  
🔗 **http://127.0.0.1:8080/login**

* * *

Deseja que eu adicione **registro de novos usuários (signup)** e **mensagens flash (exibidas uma única vez)** também?  
Posso expandir o exemplo com isso.

---

## You asked:

Sim

---

## ChatGPT Replied:

Excelente 🔥  
Vamos estender o sistema de **login com Rust + PostgreSQL + Actix-web + Tera + Bootstrap** para incluir:

✅ Registro de novos usuários (signup)  
✅ Mensagens flash (mensagens temporárias, exibidas uma vez após redirecionamento)  
✅ Layout base reaproveitável com Tera (`base.html`)

* * *

📦 Estrutura atualizada
-----------------------

    rust-login/
    ├── Cargo.toml
    ├── .env
    └── src/
        ├── main.rs
        ├── db.rs
        ├── models.rs
        ├── routes.rs
        ├── session.rs
        └── views/
            ├── base.html
            ├── login.html
            ├── signup.html
            └── dashboard.html
    

* * *

🧱 Banco de Dados (PostgreSQL)
------------------------------

    CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL
    );
    

* * *

⚙️ Atualize `Cargo.toml`
------------------------

Adicione:

    actix-web-flash-messages = "0.5"
    

(essa crate simplifica mensagens flash com redirecionamentos)

* * *

🧠 `src/models.rs`
------------------

Atualize para incluir os dados de cadastro:

    use serde::Deserialize;
    
    #[derive(Deserialize)]
    pub struct LoginData {
        pub username: String,
        pub password: String,
    }
    
    #[derive(Deserialize)]
    pub struct SignupData {
        pub username: String,
        pub password: String,
        pub confirm_password: String,
    }
    

* * *

🔑 `src/session.rs`
-------------------

Sem mudanças, continua o mesmo:

    use actix_session::Session;
    
    pub fn set_logged_in(session: &Session, username: &str) {
        session.insert("user", username).unwrap();
    }
    
    pub fn is_logged_in(session: &Session) -> bool {
        session.get::<String>("user").unwrap_or(None).is_some()
    }
    
    pub fn logout(session: &Session) {
        session.remove("user");
    }
    

* * *

🧭 `src/routes.rs` (atualizado)
-------------------------------

Aqui adicionamos:

*   `/signup` (cadastro)
    
*   mensagens flash (usando `actix-web-flash-messages`)
    
*   melhorias na renderização dos templates com mensagens
    

    use actix_web::{web, HttpResponse, Responder};
    use actix_session::Session;
    use actix_web_flash_messages::{FlashMessage, IncomingFlashMessages};
    use tera::Tera;
    use bcrypt::{hash, verify};
    use crate::{db::DBPool, models::{LoginData, SignupData}, session};
    
    pub async fn get_login(
        tmpl: web::Data<Tera>,
        flash_messages: IncomingFlashMessages,
    ) -> impl Responder {
        let mut ctx = tera::Context::new();
        let messages: Vec<String> = flash_messages.iter().map(|m| m.content().to_string()).collect();
        if !messages.is_empty() {
            ctx.insert("flash", &messages);
        }
        let s = tmpl.render("login.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn post_login(
        tmpl: web::Data<Tera>,
        form: web::Form<LoginData>,
        pool: web::Data<DBPool>,
        session: Session,
    ) -> impl Responder {
        let client = pool.get().await.unwrap();
        let row = client
            .query_opt("SELECT username, password FROM users WHERE username=$1", &[&form.username])
            .await
            .unwrap();
    
        if let Some(row) = row {
            let stored_hash: String = row.get("password");
            if verify(&form.password, &stored_hash).unwrap() {
                session::set_logged_in(&session, &form.username);
                FlashMessage::info("Login realizado com sucesso!").send();
                return HttpResponse::Found()
                    .append_header(("Location", "/dashboard"))
                    .finish();
            }
        }
    
        FlashMessage::error("Usuário ou senha incorretos.").send();
        HttpResponse::Found().append_header(("Location", "/login")).finish()
    }
    
    pub async fn get_signup(
        tmpl: web::Data<Tera>,
        flash_messages: IncomingFlashMessages,
    ) -> impl Responder {
        let mut ctx = tera::Context::new();
        let messages: Vec<String> = flash_messages.iter().map(|m| m.content().to_string()).collect();
        if !messages.is_empty() {
            ctx.insert("flash", &messages);
        }
        let s = tmpl.render("signup.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn post_signup(
        form: web::Form<SignupData>,
        pool: web::Data<DBPool>,
    ) -> impl Responder {
        if form.password != form.confirm_password {
            FlashMessage::error("As senhas não coincidem.").send();
            return HttpResponse::Found().append_header(("Location", "/signup")).finish();
        }
    
        let client = pool.get().await.unwrap();
        let hashed = hash(&form.password, 12).unwrap();
    
        let res = client
            .execute(
                "INSERT INTO users (username, password) VALUES ($1, $2)",
                &[&form.username, &hashed],
            )
            .await;
    
        match res {
            Ok(_) => {
                FlashMessage::info("Cadastro realizado com sucesso! Faça login.").send();
                HttpResponse::Found().append_header(("Location", "/login")).finish()
            }
            Err(_) => {
                FlashMessage::error("Usuário já existe.").send();
                HttpResponse::Found().append_header(("Location", "/signup")).finish()
            }
        }
    }
    
    pub async fn get_dashboard(
        tmpl: web::Data<Tera>,
        session: Session,
    ) -> impl Responder {
        if !session::is_logged_in(&session) {
            return HttpResponse::Found().append_header(("Location", "/login")).finish();
        }
    
        let mut ctx = tera::Context::new();
        ctx.insert("username", &session.get::<String>("user").unwrap().unwrap());
        let s = tmpl.render("dashboard.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    
    pub async fn logout(session: Session) -> impl Responder {
        session::logout(&session);
        FlashMessage::info("Você saiu da conta.").send();
        HttpResponse::Found().append_header(("Location", "/login")).finish()
    }
    

* * *

🚀 `src/main.rs` (atualizado)
-----------------------------

    mod db;
    mod models;
    mod routes;
    mod session;
    
    use actix_web::{web, App, HttpServer};
    use actix_session::SessionMiddleware;
    use actix_web::cookie::Key;
    use actix_web_flash_messages::{storage::CookieMessageStore, FlashMessagesFramework};
    use dotenvy::dotenv;
    use tera::Tera;
    use std::env;
    
    #[actix_web::main]
    async fn main() -> std::io::Result<()> {
        dotenv().ok();
        let pool = db::create_pool();
        let secret_key = Key::from(env::var("SESSION_KEY").unwrap().as_bytes());
        let tera = Tera::new("src/views/**/*").unwrap();
    
        let message_store = CookieMessageStore::builder(Key::from(&secret_key.clone())).build();
        let flash_framework = FlashMessagesFramework::builder(message_store).build();
    
        println!("🚀 Servidor rodando em http://127.0.0.1:8080");
    
        HttpServer::new(move || {
            App::new()
                .wrap(flash_framework.clone())
                .wrap(SessionMiddleware::new(
                    actix_session::storage::CookieSessionStore::default(),
                    secret_key.clone(),
                ))
                .app_data(web::Data::new(pool.clone()))
                .app_data(web::Data::new(tera.clone()))
                .service(
                    web::resource("/login")
                        .route(web::get().to(routes::get_login))
                        .route(web::post().to(routes::post_login)),
                )
                .service(
                    web::resource("/signup")
                        .route(web::get().to(routes::get_signup))
                        .route(web::post().to(routes::post_signup)),
                )
                .service(web::resource("/dashboard").route(web::get().to(routes::get_dashboard)))
                .service(web::resource("/logout").route(web::get().to(routes::logout)))
        })
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
    }
    

* * *

🎨 Templates
------------

### `src/views/base.html`

    <!DOCTYPE html>
    <html lang="pt-br">
    <head>
        <meta charset="UTF-8">
        <title>{% block title %}Rust Login{% endblock %}</title>
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
    </head>
    <body class="bg-light">
        <div class="container py-5">
            {% if flash %}
                {% for msg in flash %}
                    <div class="alert alert-info alert-dismissible fade show" role="alert">
                        {{ msg }}
                        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
    
            {% block content %}{% endblock %}
        </div>
        <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"></script>
    </body>
    </html>
    

* * *

### `src/views/login.html`

    {% extends "base.html" %}
    {% block title %}Login{% endblock %}
    {% block content %}
    <div class="col-md-4 mx-auto bg-white p-4 rounded shadow">
        <h3 class="mb-3">Login</h3>
        <form method="post" action="/login">
            <div class="mb-3">
                <input type="text" name="username" class="form-control" placeholder="Usuário" required>
            </div>
            <div class="mb-3">
                <input type="password" name="password" class="form-control" placeholder="Senha" required>
            </div>
            <button type="submit" class="btn btn-primary w-100">Entrar</button>
        </form>
        <div class="mt-3">
            <a href="/signup">Criar nova conta</a>
        </div>
    </div>
    {% endblock %}
    

* * *

### `src/views/signup.html`

    {% extends "base.html" %}
    {% block title %}Cadastro{% endblock %}
    {% block content %}
    <div class="col-md-4 mx-auto bg-white p-4 rounded shadow">
        <h3 class="mb-3">Criar Conta</h3>
        <form method="post" action="/signup">
            <div class="mb-3">
                <input type="text" name="username" class="form-control" placeholder="Usuário" required>
            </div>
            <div class="mb-3">
                <input type="password" name="password" class="form-control" placeholder="Senha" required>
            </div>
            <div class="mb-3">
                <input type="password" name="confirm_password" class="form-control" placeholder="Confirmar senha" required>
            </div>
            <button type="submit" class="btn btn-success w-100">Cadastrar</button>
        </form>
        <div class="mt-3">
            <a href="/login">Voltar para login</a>
        </div>
    </div>
    {% endblock %}
    

* * *

### `src/views/dashboard.html`

    {% extends "base.html" %}
    {% block title %}Dashboard{% endblock %}
    {% block content %}
    <nav class="navbar navbar-dark bg-dark p-3 rounded mb-4">
        <span class="navbar-brand">Bem-vindo, {{ username }}</span>
        <a href="/logout" class="btn btn-outline-light">Logout</a>
    </nav>
    <div class="card p-4 shadow-sm">
        <h4>Área restrita</h4>
        <p>Somente usuários logados podem ver esta página.</p>
    </div>
    {% endblock %}
    

* * *

▶️ Executar
-----------

    cargo run
    

Acesse:

*   🔑 http://127.0.0.1:8080/signup → para criar conta
    
*   🔓 http://127.0.0.1:8080/login → para logar
    
*   🧭 http://127.0.0.1:8080/dashboard → área protegida
    

* * *

Deseja que eu adicione também **proteção via middleware (redirect automático se não logado)** em todas as rotas protegidas — em vez de verificar manualmente no handler?  
Isso deixaria o código mais limpo e escalável.

---

## You asked:

Sim

---

## ChatGPT Replied:

Excelente 👌  
Vamos agora tornar o sistema **mais profissional e limpo**, adicionando um **middleware de autenticação** para proteger rotas automaticamente.

Com isso:

*   `/dashboard`, e futuras rotas privadas, só poderão ser acessadas se o usuário estiver logado.
    
*   O middleware redirecionará automaticamente para `/login` se não houver sessão válida.
    
*   O código dos handlers fica mais limpo e coeso.
    

* * *

🔒 Nova Estrutura
-----------------

    rust-login/
    └── src/
        ├── main.rs
        ├── db.rs
        ├── models.rs
        ├── routes.rs
        ├── session.rs
        ├── middleware.rs  <-- novo
        └── views/
    

* * *

🧱 `src/middleware.rs`
----------------------

Middleware simples que intercepta as rotas marcadas como "protegidas":

    use actix_web::{
        dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
        Error, HttpResponse,
    };
    use actix_session::SessionExt;
    use futures_util::future::{ready, LocalBoxFuture, Ready};
    
    pub struct AuthMiddleware;
    
    impl<S, B> Transform<S, ServiceRequest> for AuthMiddleware
    where
        S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
        B: 'static,
    {
        type Response = ServiceResponse<B>;
        type Error = Error;
        type Transform = AuthMiddlewareMiddleware<S>;
        type InitError = ();
        type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
        fn new_transform(&self, service: S) -> Self::Future {
            ready(Ok(AuthMiddlewareMiddleware { service }))
        }
    }
    
    pub struct AuthMiddlewareMiddleware<S> {
        service: S,
    }
    
    impl<S, B> Service<ServiceRequest> for AuthMiddlewareMiddleware<S>
    where
        S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
        B: 'static,
    {
        type Response = ServiceResponse<B>;
        type Error = Error;
        type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
        forward_ready!(service);
    
        fn call(&self, req: ServiceRequest) -> Self::Future {
            // Verifica a sessão
            let session = req.get_session();
            let user = session.get::<String>("user").unwrap_or(None);
    
            // Caminho atual
            let path = req.path().to_string();
    
            // Permite login e signup sem restrição
            if path.starts_with("/login") || path.starts_with("/signup") {
                return Box::pin(self.service.call(req));
            }
    
            // Bloqueia acesso se não estiver logado
            if user.is_none() {
                let res = HttpResponse::Found()
                    .append_header(("Location", "/login"))
                    .finish();
                return Box::pin(async { Ok(req.into_response(res.map_into_left_body())) });
            }
    
            Box::pin(self.service.call(req))
        }
    }
    

* * *

🧭 Atualize `src/main.rs`
-------------------------

Agora registramos o middleware globalmente, e limpamos o código:

    mod db;
    mod models;
    mod routes;
    mod session;
    mod middleware;
    
    use actix_web::{web, App, HttpServer};
    use actix_session::SessionMiddleware;
    use actix_web::cookie::Key;
    use actix_web_flash_messages::{storage::CookieMessageStore, FlashMessagesFramework};
    use dotenvy::dotenv;
    use tera::Tera;
    use std::env;
    use middleware::AuthMiddleware;
    
    #[actix_web::main]
    async fn main() -> std::io::Result<()> {
        dotenv().ok();
        let pool = db::create_pool();
        let secret_key = Key::from(env::var("SESSION_KEY").unwrap().as_bytes());
        let tera = Tera::new("src/views/**/*").unwrap();
    
        let message_store = CookieMessageStore::builder(Key::from(&secret_key.clone())).build();
        let flash_framework = FlashMessagesFramework::builder(message_store).build();
    
        println!("🚀 Servidor rodando em http://127.0.0.1:8080");
    
        HttpServer::new(move || {
            App::new()
                .wrap(flash_framework.clone())
                .wrap(SessionMiddleware::new(
                    actix_session::storage::CookieSessionStore::default(),
                    secret_key.clone(),
                ))
                .wrap(AuthMiddleware) // ⬅️ protege rotas automaticamente
                .app_data(web::Data::new(pool.clone()))
                .app_data(web::Data::new(tera.clone()))
                // Rotas públicas
                .service(
                    web::resource("/login")
                        .route(web::get().to(routes::get_login))
                        .route(web::post().to(routes::post_login)),
                )
                .service(
                    web::resource("/signup")
                        .route(web::get().to(routes::get_signup))
                        .route(web::post().to(routes::post_signup)),
                )
                // Rotas privadas
                .service(web::resource("/dashboard").route(web::get().to(routes::get_dashboard)))
                .service(web::resource("/logout").route(web::get().to(routes::logout)))
        })
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
    }
    

* * *

✨ Simplificação em `routes.rs`
------------------------------

Como agora o middleware bloqueia o acesso às rotas protegidas, o handler de dashboard pode ser simplificado:

    pub async fn get_dashboard(
        tmpl: web::Data<Tera>,
        session: Session,
    ) -> impl Responder {
        let username = session.get::<String>("user").unwrap().unwrap();
        let mut ctx = tera::Context::new();
        ctx.insert("username", &username);
        let s = tmpl.render("dashboard.html", &ctx).unwrap();
        HttpResponse::Ok().body(s)
    }
    

* * *

✅ Funcionamento final
---------------------

| Rota | Acesso | Descrição |
| --- | --- | --- |
| `/signup` | Público | Cria uma nova conta |
| `/login` | Público | Faz login |
| `/dashboard` | Protegido | Redireciona para login se não logado |
| `/logout` | Protegido | Finaliza sessão |

* * *

🧠 Extras possíveis
-------------------

Deseja que eu adicione também:

*   🔐 **Hash automático de senha ao registrar e checagem com Argon2** (mais seguro que bcrypt)?
    
*   🔁 **Lembrar usuário (“keep me logged in”) via cookie persistente**?
    

Esses seriam os próximos passos naturais para tornar o login **nível produção**. Quer que eu adicione essas melhorias?

---

