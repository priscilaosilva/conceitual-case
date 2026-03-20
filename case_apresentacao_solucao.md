# Apresentação da Solução — Fluxo de Avaliação de Fraude (Resumo)

Este documento resume e apresenta a solução proposta para o fluxo de avaliação de fraudes em transações de cartão, considerando os tópicos e perguntas do case em [case-priscilaosilva.md](case-priscilaosilva.md) e o desenho de solução referenciado (`desenho_solucao.png`).

**Resumo curto:** arquitetura orientada a eventos com caminho síncrono de baixa latência (hot path) para decisão em <= 500ms e caminho assíncrono para investigação profunda (AI + workflows). Uso de CQRS (DynamoDB write, ElastiCache Redis read), SQS/EventBridge para desacoplamento, S3 Data Lake para análise e observabilidade via CloudWatch / X-Ray / CloudTrail / Datadog.

**Objetivo:** entregar uma avaliação rápida e confiável da transação (APPROVED / SUSPECT / DECLINED), persistir decisão e enriquecer com investigação assíncrona para melhorar modelos e regras.

**Arquitetura — visão de alto nível**
- **Entrada / Edge**: Route 53 → CloudFront (CDN) + WAF → API Gateway
- **Autenticação / Autorização**: AWS Cognito para identidade + `Lambda Authorizer` para validar JWT, permissões de rota e contexto do perfil
- **Caminho síncrono (Hot Path)**: `Lambda Orchestrator` (implementação .NET) — consulta camadas read (Redis) e invoca 3 lambdas paralelas para recuperar: perfil/saldo/limite, dispositivo/merchant/contexto, histórico/concorrência
- **Persistência rápida**: ElastiCache Redis (projections / read-model)
- **Persistência de verdade (write)**: DynamoDB (source of truth) — decisões e metadados
- **Desacoplamento / eventos**: DynamoDB Streams ou publicação direta (EventBridge / SQS) para acionar investigações
- **Fila de investigação**: SQS (buffer, retries, DLQ)
- **Investigação Profunda**: workers em Python (Lambda/containers) que orquestram agentes AI (LLMs) via camada controlada de ferramentas (`API - tool layer`)
- **Data Lake / Analytics**: S3 organizado em camadas (raw / curated / ml-features / agent-investigations)
- **Observabilidade & Segurança**: CloudWatch (logs, métricas, alarms), X‑Ray (tracing), CloudTrail (audit), Datadog (visão central)

**Fluxo detalhado (pontos-chave)**
- **1) Entrada da requisição**: cliente envia transação → CDN/WAF protege e encaminha para API Gateway
- **2) Autorização**: `Lambda Authorizer` valida JWT via Cognito e checa escopo/perfil
- **3) Hot Path (<= 500ms)**: `Lambda Orchestrator` paraleliza chamadas para obter: perfil/score, device+merchant, concorrência/velocity (cada uma em lambdas especializadas). Todos os dados preferencialmente lidos do Redis; faltantes usam fallback para DynamoDB com limites de timeout configurados.
- **4) Decisão imediata**: depois de agregar resultados (regras rápidas + ML score), retorna: APPROVED / SUSPECT / DECLINED. Se falha total (timeouts / indisponibilidade), retornar DECLINED por indisponibilidade.
- **5) Persistência da decisão**: `Decision Writer Lambda` grava transação, reason codes, score e metadados no `DynamoDB` e atualiza projeções no Redis.
- **6) Desacoplamento para investigação**: para SUSPECT / FRAUD_DECLINED (e optionally amostras de APPROVED) publica evento (via Streams / EventBridge → SQS) para investigação profunda.
- **7) Fila e worker de investigação**: SQS absorve picos; worker (Python) consome, monta payload investigativo e chama serviços internos (Case Enrichment, ML Scoring, Agent Orchestrator).
- **8) Agentes AI (LLMs)**: orquestrados por `Agent Orchestrator`, com acesso limitado por `API - tool layer` (somente dados estruturados e necessários). Agentes produzem resumo investigativo, sugestões de ação e justificativas (auditáveis).
- **9) Resultados da investigação**: ações possíveis — abrir ticket, bloquear conta, atualizar sinais de risco, enriquecer features para ML, armazenar evidências em S3 e atualizar dashboards.

**Componentes e responsabilidades (resumido)**
- **`API Gateway` + `Lambda Authorizer`**: proteção de rota, validação JWT, defesa inicial de autorização
- **`Lambda Orchestrator` (.NET)**: coordenação do hot path, orquestra chamadas concorrentes, aplica regras rápidas
- **`ElastiCache Redis`**: read-model com projeções atualizadas para latência < 500ms
- **`DynamoDB`**: single source of truth; grava eventos/decisões; alimenta streams
- **`Decision Writer Lambda`**: persistência consistente e gravação de reason codes/metadata
- **`SQS` / `EventBridge`**: desacoplamento e tolerância a picos
- **`Fraud Investigation Worker` (Python)**: lógica investigativa, enrichment, trigger de AI agents
- **`Agent Orchestrator` + `API - tool layer`**: executor de prompts e chamadas seguras para ferramentas, controla contexto e grava auditoria
- **`S3 Data Lake`**: armazenar raw e artefatos analíticos para reprocessamento e auditoria
- **Observabilidade**: CloudWatch, X‑Ray, CloudTrail, Datadog

**Padrões de arquitetura aplicados**
- **CQRS**: separação entre write model (DynamoDB) e read model (Redis projections) — reduz latência de leitura no hot path
- **Event-driven / Pub‑Sub**: DynamoDB Streams / EventBridge + SQS para desacoplar investigação e suportar reprocessamento
- **Saga / Fallbacks**: sequência de compensações e retries para operações que podem falhar (e.g., atualizar múltiplos sistemas), especialmente na camada assíncrona
- **Bulkhead / Circuit Breaker (operacional)**: isolar subsistemas para evitar cascatas de falha (aplicáveis em lambdas + dependências externas)

**Segurança e governança**
- **Autenticação**: Cognito para identidade, com MFA/rotacionamento de tokens conforme política
- **Autorização**: `Lambda Authorizer` com verificação de claims, escopos e perfil
- **Auditoria**: CloudTrail para todas as chamadas AWS; gravação das decisões e justificativas no S3/DynamoDB
- **Proteção de dados**: criptografia at‑rest (S3/EBS/DynamoDB) e in‑transit (TLS), access policies restritivas (IAM) para `API - tool layer` e agentes

**Observabilidade e operação**
- **Logs & métricas**: CloudWatch para logs e métricas; criar métricas customizadas para latência do hot path, taxa de SUSPECT, tempos de investigação
- **Tracing**: X‑Ray para end‑to‑end tracing do hot path e investigação assincrona
- **Centralização e alertas**: Datadog para dashboards, correlação e alertas operacionais (ex.: spike em SUSPECT ou DLQ)

**Decisões de tecnologia e justificativas**
- **.NET no hot path**: forte tipagem, performance determinística, produtividade em ambientes backend corporativos; adequado para latência crítica e contratos rígidos
- **Python na investigação**: ecossistema rico para ML/feature engineering e integração com LLMs; agilidade na construção de pipelines investigativos
- **DynamoDB + Redis**: DynamoDB para escala, baixa manutenção e streams; Redis para leituras ultra-rápidas via projeções
- **SQS/EventBridge**: SQS para buffering/ordenamento de investigação; EventBridge para roteamento de eventos e integração cross-account

**Restrições, riscos e mitigação**
- **Risco**: Dependência forte em Redis para latência; se Redis fora, hot path pode usar DynamoDB causando aumento de latência.
  - **Mitigação**: limites de timeout, circuit-breaker, degrade gracioso (ex.: respostas conservadoras) e gravação de telemetria para análise
- **Risco**: LLMs podem gerar falsas inferências / vieses
  - **Mitigação**: agentes não tomam ações finais; retornam recomendações que passam por revisão humana; registrar prompts, respostas e evidências para auditoria
- **Risco**: custo de armazenamento e queries no Data Lake
  - **Mitigação**: lifecycle rules no S3 e políticas de retenção; compactação e catalogação adequada

**APIs e contratos (sugestão)**
- **Auth**: `/auth/introspect` — valida token
- **Decision**: `POST /transactions/assess` — payload transacional, retorna `decision`, `score`, `reason_codes` (síncrono)
- **Investigação**: eventos publicados em bus; API interna `GET /cases/{id}` e `POST /cases/{id}/actions` para ferramentas e agentes

**Métricas principais a monitorar**
- P95 / P99 latência do hot path (target <=500ms)
- Taxa de APPROVED / SUSPECT / DECLINED
- Número de mensagens em SQS / mensagens em DLQ
- Taxa de fallback para DynamoDB (indicando cache misses)
- Tempo médio de investigação e taxa de confirmação de fraude

**Próximos passos sugeridos**
1. Prototipar o hot path com contratos de API e SLAs de timeout (simular latências e falhas)
2. Implementar testes de carga e chaos engineering para validar fallbacks e SLOs
3. Definir esquema de eventos do DynamoDB Streams / EventBridge e políticas de DLQ/retries
4. Especificar contrato seguro para `API - tool layer` usado pelos agentes (audit trail)
5. Construir POC do Agent Orchestrator com dataset limitados e revisar governança de uso de LLMs

**CI/CD e Deploy**
- **Pipeline**: usar GitHub Actions para CI (build, lint, testes unitários) e CD com conexão segura à conta AWS (OIDC/GitHub Actions roles) para deploy estruturado
- **Estratégia de deploy**: adotar deploys canary para lambdas/serviços críticos, usando tráfego gradual + métricas automáticas para rollback (ex.: Lambda aliases + CloudWatch alarms / CodeDeploy Canary)
- **Automação**: pipelines separados para hot path (.NET) e coleta/worker (Python), com etapas de infraestrutura como código (Terraform / CloudFormation) e validações automáticas de segurança (IaC scanning)
- **Rollbacks e contingência**: gatilhos automáticos para rollback em caso de degradação de latência, erros 5xx ou aumento de taxa de SUSPECT; stages de pré-produção e testes canary em dados sintetizados

**Uso de IA no desenvolvimento e testes**
- **Assistência ao desenvolvimento**: uso de IA para geração de trechos de código, sugestão de refatoração e templates de testes unitários, com revisão humana obrigatória antes do merge
- **Testes automatizados assistidos por IA**: gerar casos de teste, inputs edge-case e mocks para melhorar cobertura e detectar regressões de performance
- **Qualidade e governança**: integrar verificações automáticas (security linters, dependency checks) com suporte de IA para priorizar findings; manter logs de prompts e versão das ferramentas de IA para auditoria
- **Treinamento contínuo**: usar outputs das investigações e labels confirmadas para treinar modelos ML e melhorar prompts/agents; manter pipeline de MLOps controlado e versionado

---
Arquivo fonte e referências: [case-priscilaosilva.md](case-priscilaosilva.md), [case_concentual-priscilaoliveira.md](case_concentual-priscilaoliveira.md), [contextosimportantes.md](contextosimportantes.md), desenho de referência: `desenho_solucao.png`.

Se desejar, eu adapto o conteúdo para um formato de apresentação (slides), incluo diagrama em SVG/mermaid gerável automaticamente, ou gero um README simplificado para o repositório.
