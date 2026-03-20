# Governança (obrigatório em banco)

Você PRECISA mencionar isso.

Controles

auditoria de todas as decisões

versionamento de prompt

versionamento de ferramentas

controle de acesso (IAM)

masking de dados sensíveis

trilha de execução do agente

Human-in-the-loop

Sempre que:

alta criticidade

baixa confiança

impacto financeiro alto

👉 decisão final humana


A arquitetura utiliza agentes de IA como camada de análise contextual, substituindo parcialmente modelos supervisionados na investigação profunda. Esses agentes operam de forma controlada por meio de um orquestrador e um conjunto de ferramentas definidas, garantindo segurança e auditabilidade. O processo de evolução dos agentes não depende apenas de treinamento tradicional, mas de um ciclo contínuo baseado em dados históricos, feedback humano, versionamento de prompts e avaliação sistemática. Isso permite melhorar progressivamente a qualidade das análises mantendo governança e previsibilidade.


# Camada de analytics
Em vez de depender exclusivamente de modelos supervisionados, a arquitetura incorpora agentes de IA como camada de análise contextual e investigação assistida, mantendo governança, auditabilidade e controle operacional. Os agentes não substituem decisões críticas do fluxo transacional em tempo real, mas ampliam a capacidade de análise, explicação e resposta a fraudes complexas.

# Decisão parcial com política conservadora

Quando um sinal antifraude falha, você não precisa obrigatoriamente recusar tudo.
Você pode decidir com os dados disponíveis, mas aumentando cautela.

Exemplos

ausência de velocity aumenta score base

ausência de profile impede aprovação automática em transações altas

ausência de merchant risk manda para “suspeita”

# Reaquecimento de cache

Quando houver fallback ao banco, você pode aproveitar para atualizar o Redis.

Exemplo

Redis miss em profile

consulta DynamoDB

resposta volta

salva novamente no Redis com TTL

Isso evita repetir misses em rajadas.

# Fallback por componente

## Auth / token

Se falhar:

- normalmente fail-closed
- não autoriza

## Limite / saldo / status do cartão

Fallback:

- Redis → DynamoDB / sistema fonte
- se indisponível: recusa ou revisão
- não pode aprovar no escuro

## Velocity

Fallback:

- Redis miss → retorna unknown
- não consultar histórico pesado no hot path
- aumenta cautela
- pode gerar investigação assíncrona

## Profile

Fallback:

- Redis → read model materializado
- se indisponível, score parcial
- para ticket alto, pode mandar para suspeita

## Merchant risk / device risk

Fallback:

- usar último valor conhecido
- usar default conservador
- usar lista estática de alto risco
- se indisponível, não bloquear tudo por isso

## ML score rápido

Fallback:

- usar só regras + sinais essenciais
- não travar a autorização esperando inferência pesada

Exemplo prático

Imagine uma compra de R$ 7.500.

A orchestrator chama em paralelo:

- saldo/limite
- velocity
- profile
- merchant risk

Resultado:

- saldo: ok
- velocity: timeout
- profile: ok
- merchant risk: indisponível

Em vez de falhar tudo, ela faz:

- usa limite e profile
- marca velocity_unknown=true
- aplica score conservador
- se ultrapassar limiar, classifica como SUSPECT
- envia para análise profunda

# Comentário

A camada de decisão rápida adota fallback diferenciado por criticidade da dependência. Informações mandatórias para autorização, como status do cartão e limite disponível, utilizam fallback controlado ao sistema fonte e operam em fail-closed quando indisponíveis. Já sinais antifraude complementares, como velocity, profile enriquecido e reputação de merchant/device, operam com timeouts curtos, defaults conservadores e decisões parciais, preservando o SLA transacional sem comprometer a segurança. Em cenários degradados, transações podem ser encaminhadas para investigação assíncrona adicional.

# Tabela de fallback da camada de decisão rápida

| Dependência / Sinal            | Fonte principal                | Fallback                                      | Timeout sugerido | Comportamento em falha                                                                 |
|--------------------------------|--------------------------------|-----------------------------------------------|------------------|----------------------------------------------------------------------------------------|
| Autenticação / JWT             | API Gateway Authorizer / Lambda Auth | Nenhum relevante                              | 5–20 ms          | **Fail-closed**. Requisição negada por segurança.                                      |
| Status do cartão               | Redis / read model             | DynamoDB / sistema transacional                | 10–30 ms         | Se indisponível, **não autoriza**. Dado crítico.                                       |
| Limite / saldo disponível      | Redis / read model             | DynamoDB / core transacional                   | 10–30 ms         | Se indisponível, **fail-closed** ou revisão, conforme política.                        |
| Status da conta / cliente      | Redis                          | DynamoDB                                      | 10–30 ms         | Se indisponível, decisão conservadora; em casos críticos, **nega**.                    |
| Velocity (1 min / 5 min / 1h)  | Redis                          | Sem fallback pesado no hot path                | 5–15 ms          | Retorna `unknown`; aumenta cautela do score; pode mandar para investigação assíncrona. |
| Profile enriquecido            | Redis / materialized view      | DynamoDB ou read model leve                    | 10–25 ms         | Se faltar, usa score parcial; em ticket alto pode classificar como `SUSPECT`.          |
| Merchant risk                  | Redis / tabela materializada   | Último valor conhecido / configuração estática | 5–15 ms          | Se indisponível, usa default conservador.                                              |
| Device trust / fingerprint     | Redis / serviço de risco       | Último valor conhecido / default conservador   | 5–15 ms          | Se indisponível, sinal `unknown`; aumenta cautela.                                     |
| Geolocalização / geo mismatch  | Redis / serviço auxiliar       | Default `unknown`                              | 5–15 ms          | Não bloqueia sozinho; influencia score.                                                |
| Blacklist / whitelist          | Cache local / Redis            | Configuração local em memória                  | 1–5 ms           | Se Redis falhar, usa lista local carregada previamente.                                |
| Regras rápidas de fraude       | Engine local / Lambda Rules    | Configuração de contingência                   | 5–15 ms          | Mantém regras essenciais mesmo em falha parcial.                                       |
| ML score rápido                | Serviço de inferência leve / Redis | Regras determinísticas + score simplificado    | 10–20 ms         | Segue com decisão simplificada; não bloqueia esperando inferência.                     |
| Histórico recente              | Redis                          | DynamoDB (apenas se leve)                      | 10–25 ms         | Evita consulta pesada; usa decisão parcial.                                            |
| Dados acessórios               | Redis / APIs auxiliares        | Nenhum                                         | 5–10 ms          | Pode retornar vazio; não impacta decisão.                                              |