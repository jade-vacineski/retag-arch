# Retag - Architecture Documentation

Este repositório contém a documentação arquitetural
do sistema **Retag - Simulador de Preços para Brechós**.

A arquitetura está documentada em níveis:
- **Context Diagram** – visão de alto nível do sistema
- **Container Diagram** – fronteiras e contêineres
- **Decisões arquiteturais** (ADRs)
- **Modelos de dados / ER**

### 📂 Estrutura

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
