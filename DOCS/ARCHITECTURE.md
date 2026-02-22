# Documentação Arquitetural

## 1. Visão geral
Este repositório segue uma arquitetura em 2 aplicações principais:
- `FrontEnd`: aplicação web em Next.js (App Router), responsável por UX, autenticação no navegador e consumo da API.
- `BackEnd`: API REST em NestJS, responsável por autenticação, regras de precificação com IA e persistência em MySQL via Knex.

Objetivo do sistema:
- Simular preço sugerido para itens de brechó com apoio de IA.
- Permitir salvar, listar, editar, excluir e marcar itens como vendidos.
- Suportar usuários autenticados e uma simulação limitada para visitantes.

## 2. Arquitetura lógica (containers)

```text
[Browser / Next.js FrontEnd]
        |
        | HTTPS (JSON + JWT Bearer)
        v
[NestJS API BackEnd]
   |            |
   |            +--> [OpenAI Responses API] (sugestão de preço)
   |
   +--> [MySQL] (users, items, guest_price_simulations)

Fluxos auxiliares:
BackEnd -> Resend API (opcional) para reset de senha
BackEnd -> Google OAuth2 para login social
```

## 3. FrontEnd (Next.js)

### 3.1 Estrutura
- Base: `FrontEnd/app`
- Páginas principais:
- `FrontEnd/app/page.tsx`: landing, cadastro/login, Google login, forgot password e simulação rápida.
- `FrontEnd/app/admin/page.tsx`: painel autenticado com listagem, filtros, paginação, edição e exclusão.
- `FrontEnd/app/perfil/page.tsx`: perfil do usuário.
- `FrontEnd/app/reset-password/page.tsx`: redefinição por token.
- Componente central: `FrontEnd/app/components/PriceCalculator.tsx`.

### 3.2 Comunicação com API
- URL base: `NEXT_PUBLIC_API_URL` (fallback `http://localhost:3000`).
- JWT armazenado em `localStorage` (`auth_token`).
- Header usado em rotas protegidas: `Authorization: Bearer <token>`.

### 3.3 Responsabilidades de UI
- Captura de dados do item e contexto do brechó (faixa de preço/média de canais).
- Simulação de preço sem persistência (`POST /items/price`).
- Persistência do item (`POST /items`) somente para usuário autenticado.
- Painel administrativo com operações CRUD e marca de venda.

## 4. BackEnd (NestJS)

### 4.1 Módulos
- `AuthModule`: registro/login, OAuth Google, perfil/conta, forgot/reset password.
- `ItemsModule`: simulação de preço, cadastro de item, listagem e manutenção de itens.

`AppModule` importa ambos os módulos.

### 4.2 Camadas
- `Controller`: contrato HTTP e validação de entrada via DTO.
- `Service`: regras de negócio (auth, precificação, limites de visitante, persistência).
- `database/knex.ts`: instância de conexão MySQL.

### 4.3 Middleware/infra global
- CORS habilitado com `origin: true` e `credentials: true`.
- `ValidationPipe` global com:
- `whitelist: true`
- `forbidNonWhitelisted: true`
- `transform: true`

## 5. Domínio e regras de negócio

### 5.1 Precificação
No `ItemsService`:
1. Valida campos obrigatórios (`name`, `description`, `basePrice`).
2. Para visitantes, aplica limite de 1 simulação por IP hash na tabela `guest_price_simulations`.
3. Consulta OpenAI Responses API para sugerir preço e racional.
4. Ajusta preço:
- `-15%` quando `curated = false`.
- Acréscimo para cobrir taxa de marketplace (máximo entre canais selecionados).
5. `POST /items/price` retorna simulação sem salvar.
6. `POST /items` salva no banco e vincula ao `user_id` autenticado.

### 5.2 Autenticação e conta
No `AuthService`:
- Senha com `scrypt` + salt aleatório.
- JWT assinado com `JWT_SECRET`, expiração de 7 dias.
- Login social com Google (OAuth2).
- Recuperação de senha por token hash (`sha256`) com expiração de 1 hora.
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
- `POST /price` (público, com limite para visitante)
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
- autenticação: `password_hash`, `google_id`
- recuperação: `password_reset_token_hash`, `password_reset_expires_at`
- contexto de negócio: `average_price_ceiling`, `sales_channels` (JSON em texto)
- auditoria: `created_at`, `updated_at`

### 7.2 Tabela `items`
Campos principais:
- ownership: `user_id`
- item: `name`, `description`, `brand`, `size`, `category`, `condition`, `gender`
- detalhamento por categoria: `clothing_type`, `footwear_type`, `bag_type`
- precificação: `base_price`, `suggested_price`, `ai_rationale`
- operação: `curated`, `curation_notes`, `is_sold`
- auditoria: `created_at`, `updated_at`

### 7.3 Tabela `guest_price_simulations`
- `id`
- `ip_hash` (único)
- `created_at`

## 8. Integrações externas
- OpenAI Responses API: gera sugestão de preço e racional.
- Google OAuth2: autenticação social.
- Resend API (opcional): envio de email de reset de senha.

## 9. Configuração e ambiente

### 9.1 FrontEnd
- `NEXT_PUBLIC_API_URL`

### 9.2 BackEnd
- Aplicação: `PORT`, `FRONTEND_URL`, `JWT_SECRET`
- Banco: `DB_HOST/DB_PORT/DB_USER/DB_PASSWORD/DB_NAME` (ou aliases MYSQL*)
- OpenAI: `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_BASE_URL`, `OPENAI_PROJECT_ID`, `OPENAI_ORG_ID`
- OAuth Google: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`
- Email/reset: `RESEND_API_KEY`, `MAIL_FROM`
- Limite visitante: `GUEST_SIMULATION_SALT`

## 10. Decisões arquiteturais relevantes
- Monorepo simples com separação clara FrontEnd/BackEnd.
- API stateless com JWT Bearer.
- Persistência SQL com migrations versionadas (Knex).
- Regras de precificação no BackEnd (não no cliente).
- IA como dependência externa encapsulada no `ItemsService`.
- Estratégia de degradação para reset de senha sem provedor de email (console log).

## 11. Riscos e melhorias recomendadas
- JWT em `localStorage`: avaliar migração para cookie `HttpOnly` para reduzir superfície XSS.
- Guard custom de JWT poderia migrar para estratégia Passport JWT padrão para uniformidade.
- `sales_channels` em texto JSON: avaliar coluna JSON nativa no MySQL.
- Limite de visitante por hash de IP é simples; pode evoluir para rate-limit por janela de tempo.
- Criar testes de integração para fluxos críticos (simulação, cadastro, reset de senha).
