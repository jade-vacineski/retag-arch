# retag-arch

Repositório de arquitetura do sistema de precificação para brechó ("Brechó Pricing").

## Contexto
Este repositório documenta a visão arquitetural de uma solução com:
- `FrontEnd` em Next.js para experiência do usuário e consumo de API.
- `BackEnd` em NestJS para autenticação, regras de precificação com IA e persistência.
- `MySQL` para dados de usuários, itens e simulações de visitantes.
- Integrações externas com OpenAI (sugestão de preço), Google OAuth2 (login social) e Resend (reset de senha).

Objetivo funcional:
- Simular preço sugerido para itens de brechó.
- Permitir fluxo completo de gestão de itens (criar, listar, editar, excluir e marcar como vendido) para usuários autenticados.
- Oferecer simulação limitada para visitantes.

## O que existe neste repositório
- `DOCS/ARCHITECTURE.md`: descrição detalhada da arquitetura, módulos, regras de negócio, contratos de API e riscos.
- `DOCS/ARCHITECTURE-C4-CONTEXT.mmd`: diagrama C4 de contexto.
- `DOCS/ARCHITECTURE-C4-CONTAINERS.mmd`: diagrama C4 de containers.
- `DOCS/ARCHITECTURE-C4-DEPLOYMENT.mmd`: diagrama C4 de deployment (implantação).
- `DOCS/DIAGRAMA_ER.png`: diagrama entidade-relacionamento.

## Fluxo arquitetural resumido
1. Usuário acessa o FrontEnd via navegador.
2. FrontEnd chama a API BackEnd por HTTPS com JWT quando necessário.
3. BackEnd aplica regras de autenticação e precificação.
4. BackEnd consulta OpenAI para sugestão de preço e racional.
5. BackEnd persiste e consulta dados no MySQL.

## Principais decisões de arquitetura
- Separação clara entre camadas de interface e regras de negócio.
- API stateless com autenticação JWT Bearer.
- Regras de precificação centralizadas no BackEnd (evita lógica crítica no cliente).
- Integrações externas encapsuladas na API.

## Como usar este repositório
1. Comece por `DOCS/ARCHITECTURE.md` para entender o desenho completo.
2. Consulte os arquivos `.mmd` para visão C4 (Context, Containers e Deployment).
3. Use o `DIAGRAMA_ER.png` para entender o modelo de dados.

## Escopo
Este repositório foca em arquitetura e documentação. A implementação de código (FrontEnd/BackEnd) é tratada nos respectivos projetos de aplicação.
