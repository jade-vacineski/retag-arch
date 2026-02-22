# Documentacao Arquitetural

## 1. Visao geral
Este repositorio segue uma arquitetura em 2 aplicacoes principais:
- `FrontEnd`: aplicacao web em Next.js (App Router), responsavel por UX, autenticacao no navegador e consumo da API.
- `BackEnd`: API REST em NestJS, responsavel por autenticacao, regras de precificacao com IA e persistencia em MySQL via Knex.

Objetivo do sistema:
- Simular preco sugerido para itens de brecho com apoio de IA.
- Permitir salvar, listar, editar, excluir e marcar itens como vendidos.
- Suportar usuarios autenticados e uma simulacao limitada para visitantes.

## 2. Arquitetura logica (containers)

```text
[Browser / Next.js FrontEnd]
        |
        | HTTPS (JSON + JWT Bearer)
        v
[NestJS API BackEnd]
   |            |
   |            +--> [OpenAI Responses API] (sugestao de preco)
   |
   +--> [MySQL] (users, items, guest_price_simulations)

Fluxos auxiliares:
BackEnd -> Resend API (opcional) para reset de senha
BackEnd -> Google OAuth2 para login social
```

## 3. FrontEnd (Next.js)

### 3.1 Estrutura
- Base: `FrontEnd/app`
- Paginas principais:
- `FrontEnd/app/page.tsx`: landing, cadastro/login, Google login, forgot password e simulacao rapida.
- `FrontEnd/app/admin/page.tsx`: painel autenticado com listagem, filtros, paginacao, edicao e exclusao.
- `FrontEnd/app/perfil/page.tsx`: perfil do usuario.
- `FrontEnd/app/reset-password/page.tsx`: redefinicao por token.
- Componente central: `FrontEnd/app/components/PriceCalculator.tsx`.

### 3.2 Comunicacao com API
- URL base: `NEXT_PUBLIC_API_URL` (fallback `http://localhost:3000`).
- JWT armazenado em `localStorage` (`auth_token`).
- Header usado em rotas protegidas: `Authorization: Bearer <token>`.

### 3.3 Responsabilidades de UI
- Captura de dados do item e contexto do brecho (faixa de preco/media de canais).
- Simulacao de preco sem persistencia (`POST /items/price`).
- Persistencia do item (`POST /items`) somente para usuario autenticado.
- Painel administrativo com operacoes CRUD e marca de venda.

## 4. BackEnd (NestJS)

### 4.1 Modulos
- `AuthModule`: registro/login, OAuth Google, perfil/conta, forgot/reset password.
- `ItemsModule`: simulacao de preco, cadastro de item, listagem e manutencao de itens.

`AppModule` importa ambos os modulos.

### 4.2 Camadas
- `Controller`: contrato HTTP e validacao de entrada via DTO.
- `Service`: regras de negocio (auth, precificacao, limites de visitante, persistencia).
- `database/knex.ts`: instancia de conexao MySQL.

### 4.3 Middleware/infra global
- CORS habilitado com `origin: true` e `credentials: true`.
- `ValidationPipe` global com:
- `whitelist: true`
- `forbidNonWhitelisted: true`
- `transform: true`

## 5. Dominio e regras de negocio

### 5.1 Precificacao
No `ItemsService`:
1. Valida campos obrigatorios (`name`, `description`, `basePrice`).
2. Para visitantes, aplica limite de 1 simulacao por IP hash na tabela `guest_price_simulations`.
3. Consulta OpenAI Responses API para sugerir preco e racional.
4. Ajusta preco:
- `-15%` quando `curated = false`.
- Acrescimo para cobrir taxa de marketplace (maximo entre canais selecionados).
5. `POST /items/price` retorna simulacao sem salvar.
6. `POST /items` salva no banco e vincula ao `user_id` autenticado.

### 5.2 Autenticacao e conta
No `AuthService`:
- Senha com `scrypt` + salt aleatorio.
- JWT assinado com `JWT_SECRET`, expiracao de 7 dias.
- Login social com Google (OAuth2).
- Recuperacao de senha por token hash (`sha256`) com expiracao de 1 hora.
- Envio de email via Resend (se sem API key, loga link no console).

## 6. Contratos principais de API

### 6.1 Auth (`/auth`)
- `POST /register`
- `POST /login`
- `GET /google`
- `GET /google/callback`
- `POST /forgot-password`
- `POST /reset-password`
- `GET /profile` (JWT)
- `POST /profile` (JWT)
- `POST /account` (JWT)

### 6.2 Items (`/items`)
- `POST /price` (publico, com limite para visitante)
- `POST /` (JWT)
- `GET /` (JWT, paginado)
- `GET /:id` (JWT)
- `PATCH /:id` (JWT)
- `PATCH /:id/sold` (JWT)
- `DELETE /:id` (JWT)

## 7. Modelo de dados

### 7.1 Tabela `users`
Campos principais:
- identidade: `id`, `name`, `email`, `phone`
- autenticacao: `password_hash`, `google_id`
- recuperacao: `password_reset_token_hash`, `password_reset_expires_at`
- contexto de negocio: `average_price_ceiling`, `sales_channels` (JSON em texto)
- auditoria: `created_at`, `updated_at`

### 7.2 Tabela `items`
Campos principais:
- ownership: `user_id`
- item: `name`, `description`, `brand`, `size`, `category`, `condition`, `gender`
- detalhamento por categoria: `clothing_type`, `footwear_type`, `bag_type`
- precificacao: `base_price`, `suggested_price`, `ai_rationale`
- operacao: `curated`, `curation_notes`, `is_sold`
- auditoria: `created_at`, `updated_at`

### 7.3 Tabela `guest_price_simulations`
- `id`
- `ip_hash` (unico)
- `created_at`

## 8. Integracoes externas
- OpenAI Responses API: gera sugestao de preco e racional.
- Google OAuth2: autenticacao social.
- Resend API (opcional): envio de email de reset de senha.

## 9. Configuracao e ambiente

### 9.1 FrontEnd
- `NEXT_PUBLIC_API_URL`

### 9.2 BackEnd
- Aplicacao: `PORT`, `FRONTEND_URL`, `JWT_SECRET`
- Banco: `DB_HOST/DB_PORT/DB_USER/DB_PASSWORD/DB_NAME` (ou aliases MYSQL*)
- OpenAI: `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_BASE_URL`, `OPENAI_PROJECT_ID`, `OPENAI_ORG_ID`
- OAuth Google: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`
- Email/reset: `RESEND_API_KEY`, `MAIL_FROM`
- Limite visitante: `GUEST_SIMULATION_SALT`

## 10. Decisoes arquiteturais relevantes
- Monorepo simples com separacao clara FrontEnd/BackEnd.
- API stateless com JWT Bearer.
- Persistencia SQL com migrations versionadas (Knex).
- Regras de precificacao no BackEnd (nao no cliente).
- IA como dependencia externa encapsulada no `ItemsService`.
- Estrategia de degradacao para reset de senha sem provedor de email (console log).

## 11. Riscos e melhorias recomendadas
- JWT em `localStorage`: avaliar migracao para cookie `HttpOnly` para reduzir superficie XSS.
- Guard custom de JWT poderia migrar para estrategia Passport JWT padrao para uniformidade.
- `sales_channels` em texto JSON: avaliar coluna JSON nativa no MySQL.
- Limite de visitante por hash de IP e simples; pode evoluir para rate-limit por janela de tempo.
- Criar testes de integracao para fluxos criticos (simulacao, cadastro, reset de senha).
