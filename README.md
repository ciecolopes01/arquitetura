# Arquitetura Moderna de Dados: o que realmente é moderno (e o que é só diagrama bonito)

Se a sua "arquitetura moderna" cabe em um único slide bonito, provavelmente ela não é moderna.

Trocar MySQL por Snowflake não é modernização.  
Colocar Kafka no diagrama não é event-driven.  
Chamar Data Lake de Lakehouse não resolve governança.

Arquitetura moderna não é sobre ferramenta.  
É sobre **capacidade estrutural sob escala**.

Este artigo detalha o que realmente caracteriza uma arquitetura moderna quando você precisa sustentar:

* ML em produção
* 10M–100M+ usuários
* múltiplos domínios
* compliance (LGPD/GDPR)
* crescimento contínuo

---

# 1️⃣ O que define modernidade de verdade

Arquitetura moderna é definida por capacidades, não por stack.

## ✔ Replay total de dados

Você consegue reconstruir qualquer estado histórico a partir dos dados brutos?

## ✔ Consistência online/offline

A feature usada no treino é exatamente a mesma usada na inferência em produção?

## ✔ Governança distribuída operacional

Domínios são donos dos seus dados ou tudo depende de um time central de dados?

## ✔ Observabilidade end-to-end

Você mede latência, qualidade, disponibilidade **e custo**?

## ✔ Escalabilidade previsível

Quando o volume dobra, o sistema aguenta sem reescrever pipelines?

Se a resposta é "não" para **3 ou mais desses pontos**, a arquitetura ainda está evoluindo.

---

# 2️⃣ Anti-Patterns da Modernidade Cosmética

Alguns erros comuns que criam ilusão de modernidade:

❌ **Data Lake sem governança** → vira Data Swamp em 6 meses  
❌ **Streaming sem necessidade de negócio** → custo alto, valor baixo  
❌ **CDC capturando tudo indiscriminadamente** → sobrecarga sem estratégia  
❌ **Feature reimplementada em batch e online** → drift garantido  
❌ **Kafka com retention infinita** → virando pseudo–data lake

## Kafka não é data lake

Retention deve ser definida por caso de uso real:

* **Real-time alerting** → 7 dias
* **Reprocessamento** → 30 dias
* **Auditoria recente** → até 90 dias

Histórico oficial deve estar no **Bronze** (Delta/Iceberg/Hudi), não no log do Kafka.

Modernidade exige intencionalidade, não apenas ferramentas da moda.

---

# 3️⃣ Arquitetura Estrutural Moderna

## 🔹 3.1 Ingestão Event-Driven

Componentes típicos de uma ingestão moderna:

* **Debezium** (CDC)
* **Kafka / PubSub / Event Hub** (backbone de eventos)
* **Schema Registry** (versionamento de contratos)
* **Dead Letter Queue** (tratamento de falhas)

### Event Streaming ≠ Event Sourcing

Conceitos diferentes, frequentemente confundidos:

* **Event Streaming** → transporte e processamento em fluxo contínuo
* **Event Sourcing** → eventos como fonte primária de verdade do sistema

Nem todo sistema precisa de event sourcing. A maioria precisa apenas de streaming.

### Boas práticas obrigatórias

* Versionamento de schema (SemVer: v1.0.0, v2.0.0)
* Backward compatibility validada
* Snapshot incremental controlado
* Particionamento por chave de negócio
* Retention explícita por finalidade

### Exemplo de evolução de schema compatível

**v1.0:**
```json
{
  "type": "record",
  "name": "Pedido",
  "fields": [
    {"name": "pedido_id", "type": "string"},
    {"name": "valor", "type": "double"}
  ]
}
```

**v1.1 (backward compatible):**
```json
{
  "type": "record",
  "name": "Pedido",
  "fields": [
    {"name": "pedido_id", "type": "string"},
    {"name": "valor", "type": "double"},
    {"name": "cupom", "type": ["null", "string"], "default": null}
  ]
}
```

Campo opcional com default não quebra consumers antigos.

---

## 🔹 3.2 Lakehouse com ACID real

Estrutura mínima de camadas:

* **Bronze** → imutável, histórico definitivo, dados brutos
* **Silver** → validado, limpo, enriquecido
* **Gold** → data products por domínio de negócio

Mas isso só é moderno se houver:

* **Schema enforcement** → validação na entrada
* **Time travel** → consulta de estados históricos
* **Data quality automatizada** → validação contínua
* **Storage/compute desacoplados** → escala independente

### Z-Ordering: Quando Vale a Pena

**Fazer se:**

* Tabela > 1TB
* Filtros seletivos recorrentes (`WHERE user_id = X AND data = Y`)
* Padrão de consulta estável

**Exemplo:**

```sql
OPTIMIZE tabela
ZORDER BY (user_id, data)
```

**Não fazer se:**

* Tabela < 100GB (overhead não compensa)
* Consultas imprevisíveis (padrão muda constantemente)
* Alta taxa de escrita (reotimização cara)

**Trade-off:**

* ✔ Reduz data skipping (melhora leitura em 10-50x)
* ✖ Aumenta custo de compaction (escrita mais lenta)

Otimização sem contexto é desperdício de recursos.

---

## 🔹 3.3 Streaming Stateful de Verdade

Streaming moderno não é apenas consumir tópico e salvar no banco.

Ele envolve processamento complexo:

* **Agregações por janela** (tumbling, sliding, session)
* **Exactly-once semantics** (sem duplicatas, sem perda)
* **Checkpoint persistido** (recuperação de falhas)
* **State backend escalável** (RocksDB, S3)

### Comparativo de ferramentas

| Ferramenta                 | Quando Usar                          | Quando Evitar              |
| -------------------------- | ------------------------------------ | -------------------------- |
| **Flink**                  | State > 10GB, janelas complexas      | Caso simples, time pequeno |
| **Spark Structured Streaming** | Unificar batch + stream          | Latência < 1s necessária   |
| **Kafka Streams**          | Transformações leves, baixa latência | State distribuído pesado   |

Escolha é engenharia de requisitos, não moda de conferência.

---

## 🔹 3.4 Feature Store Consistente

Aqui é onde **80% das arquiteturas de ML falham**.

### Separação clara de ambientes

**OFFLINE (Treino):**

* Storage: Delta/Iceberg
* Compute: Spark batch
* Versionamento de feature

**ONLINE (Inferência):**

* Storage: Redis / DynamoDB / Cassandra
* Latência: P99 < 10ms
* Atualização: CDC em tempo real

### Princípio fundamental

A feature deve ter **definição única e determinística**.

Nunca reimplementar lógica diferente no ambiente online.

### Exemplo prático

**Feature:** `total_compras_ultimos_30_dias`

**Definição única (batch):**

```sql
SELECT SUM(valor) AS total_compras_30d
FROM pedidos
WHERE user_id = :user_id
  AND data >= CURRENT_DATE - INTERVAL 30 DAY
```

**Aplicação online:**

Atualização incremental via CDC aplicando **exatamente a mesma regra**.

```python
# CDC event handler
def on_pedido_criado(event):
    user_id = event['user_id']
    valor = event['valor']
    
    # Incrementa feature online
    redis.zincrby(f"compras_30d:{user_id}", valor, event['timestamp'])
    
    # Remove valores antigos (> 30 dias)
    redis.zremrangebyscore(
        f"compras_30d:{user_id}", 
        0, 
        time.time() - 30*24*3600
    )
```

Sem definição única, você cria **drift artificial** entre treino e produção.

---

## 🔹 3.5 Orquestração Moderna

Orquestração moderna não é cron job no Linux.

### Requisitos obrigatórios

* **DAG versionado em Git** (infraestrutura como código)
* **Retry com backoff exponencial** (resiliência)
* **Idempotência garantida** (reexecução segura)
* **Observabilidade integrada** (logs, métricas, traces)
* **Lineage automático** (rastreabilidade)

### Exemplo de pipeline idempotente

**Correto:**

```python
@task(retries=3, retry_delay=timedelta(minutes=5))
def processar_pedidos(data_ref: str):
    """Pipeline idempotente - reexecução segura"""
    spark.sql(f"""
        INSERT OVERWRITE TABLE pedidos_processados
        PARTITION (data = '{data_ref}')
        SELECT 
            pedido_id,
            cliente_id,
            valor_total,
            status,
            '{data_ref}' AS data
        FROM pedidos_raw
        WHERE data = '{data_ref}'
    """)
```

**Anti-pattern (NÃO fazer):**

```sql
-- Perigoso: rerun duplica dados
INSERT INTO pedidos_processados 
SELECT * FROM pedidos_raw
```

Sem idempotência, cada retry gera inconsistência.

---

## 🔹 3.6 Catálogo de Dados Vivo

Catálogo de dados moderno não é planilha Excel compartilhada.

### Componentes essenciais

* **Lineage bidirecional** (upstream e downstream automáticos)
* **Classificação de dados** (público, interno, confidencial, restrito)
* **Ownership claro** (quem é responsável por cada dataset)
* **Metadata técnico + contexto de negócio**
* **Busca self-service** (discovery independente)

### Exemplo de metadata rico

```json
{
  "dataset": "pedidos_validados",
  "domain": "vendas",
  "owner": "squad-checkout",
  "owner_slack": "#squad-checkout",
  "descricao_negocio": "Pedidos processados e validados para analytics e ML",
  "classificacao": "interno",
  "pii_fields": ["cliente_email", "cliente_cpf"],
  "atualizacao": "tempo real via CDC",
  "freshness_slo": "< 5 minutos",
  "lineage_upstream": [
    "pedidos_raw",
    "clientes_bronze"
  ],
  "lineage_downstream": [
    "pedidos_gold",
    "feature_store.compras_recentes",
    "dashboard_vendas"
  ],
  "consumers": [
    {
      "squad": "financeiro",
      "use_case": "faturamento_diario",
      "criticidade": "alta"
    },
    {
      "squad": "ml",
      "use_case": "churn_prediction",
      "criticidade": "media"
    }
  ],
  "tags": ["vendas", "ml-ready", "pii"],
  "ultima_atualizacao": "2026-02-02T08:30:00Z"
}
```

**Sem catálogo:** "Onde está o dado X? Posso deletar a tabela Y?"  
**Com catálogo:** Governança operacional real.

---

# 4️⃣ Governança Operacional (Data Mesh Real)

Data Mesh não é reorganizar times no organograma.  
É implementar **contratos executáveis** entre domínios.

### Exemplo de Data Contract

```yaml
# contracts/vendas/pedidos_validados/v2.1.0.yaml

domain: vendas
product: pedidos_validados
version: 2.1.0
owner: squad-checkout
owner_contact: checkout-team@empresa.com

schema:
  - name: pedido_id
    type: string
    description: "ID único do pedido"
    constraints:
      - not_null
      - unique
      
  - name: cliente_id
    type: string
    description: "FK para clientes.cliente_id"
    constraints:
      - not_null
      
  - name: valor_total
    type: decimal(10,2)
    description: "Valor total do pedido em BRL"
    constraints:
      - not_null
      - greater_than: 0
      
  - name: status
    type: enum
    values: [pendente, pago, cancelado]
    description: "Status atual do pedido"

sla:
  freshness: "< 5 minutos"
  completeness: "> 99.9%"
  availability: "> 99.95%"
  
quality_checks:
  - tipo: "valores_nulos"
    colunas: [pedido_id, cliente_id, valor_total]
    threshold: 0
    
  - tipo: "valores_duplicados"
    colunas: [pedido_id]
    threshold: 0
    
  - tipo: "valores_fora_range"
    coluna: valor_total
    min: 0.01
    max: 1000000

consumers:
  - squad: financeiro
    version_range: ">=2.0.0 <3.0.0"
  - squad: ml
    version_range: ">=2.1.0"
```

### Evolução de schema: breaking vs non-breaking

**Não-breaking (compatível):**

```yaml
# v2.1.0 → v2.2.0
schema:
  # ... campos existentes ...
  + notificacao_enviada: boolean
    default: false
    description: "Indica se cliente foi notificado"
```

**Potencialmente breaking:**

```yaml
# v2.1.0 → v3.0.0
schema:
  - status: enum[pendente, pago, cancelado]
  + status: enum[pendente, pago, cancelado, reembolsado]
```

**Por que pode quebrar:**

Consumer com validação estrita (`if status NOT IN ['pendente', 'pago', 'cancelado']`) vai falhar.

**Estratégia segura de migração:**

1. **Comunicar mudança** com 90 dias de antecedência
2. **Publicar v3.0.0** em paralelo com v2.1.0
3. **Migrar consumers** progressivamente (tracking no catálogo)
4. **Deprecar v2.1.0** após confirmação de 100% migração
5. **Desativar v2.1.0** após período de grace (30 dias)

Governança exige **disciplina operacional**, não apenas discurso bonito.

---

# 5️⃣ Observabilidade End-to-End

Arquitetura moderna mede 4 dimensões críticas:

## 1. Latência

* P50 / P95 / P99 de CDC lag
* Tempo total evento → dashboard
* Tempo de processamento por pipeline

## 2. Volume

* Throughput (eventos/segundo)
* Taxa de erro por pipeline
* Backlog de processamento

## 3. Qualidade

* % de registros válidos
* Drift de schema detectado
* Freshness SLA (quanto tempo desde último update)

## 4. Custo

* Custo por pipeline
* Custo por Data Product
* Recursos ociosos (clusters sem uso)

### Exemplo de SLO com alertas

| Métrica       | SLO      | Warning | Critical | Ação               |
| ------------- | -------- | ------- | -------- | ------------------ |
| P95 latência  | < 5min   | > 7min  | > 10min  | Page oncall        |
| Taxa de erro  | < 0.1%   | > 0.5%  | > 1%     | Page oncall        |
| Completeness  | > 99.9%  | < 99.5% | < 99%    | Alerta + investigação |
| Uptime mensal | > 99.95% | < 99.9% | < 99.5%  | Postmortem obrigatório |

### Consequências reais

* **Warning** → Notificação Slack, monitorar
* **Critical** → Page oncall, incident aberto
* **Reincidência** → Postmortem obrigatório com action items

### Medição na prática

```promql
# Taxa de sucesso do pipeline
rate(eventos_processados_total{pipeline="pedidos", status="success"}[5m])
/ 
rate(eventos_recebidos_total{pipeline="pedidos"}[5m])

# P95 de latência
histogram_quantile(0.95, 
  rate(latencia_processamento_segundos_bucket{pipeline="pedidos"}[5m])
)

# Custo por pipeline (aproximado)
sum(compute_cost_usd{pipeline="pedidos"}) 
/ 
sum(eventos_processados_total{pipeline="pedidos"})
```

**Princípio fundamental:**

Sem consequência real (oncall, postmortem), SLO é apenas decoração no dashboard.

---

# 6️⃣ Custos, ROI e Trade-offs

Arquitetura moderna é mais poderosa.  
Mas também **significativamente mais complexa e cara**.

### Você paga com:

* Mais infraestrutura (Kafka cluster, Flink, Redis)
* Mais observabilidade (Prometheus, Grafana, Datadog)
* Mais disciplina de engenharia (testes, versionamento, postmortems)
* Mais maturidade organizacional (ownership distribuído)

### Quando investir em cada capacidade

| Capacidade           | Investir se:                  | Evitar se:               |
| -------------------- | ----------------------------- | ------------------------ |
| **Feature Store**    | 5+ modelos em produção        | Apenas POCs de ML        |
| **Streaming Stateful** | Latência < 1min crítica     | Batch diário atende      |
| **Data Mesh**        | 3+ squads produzindo dados    | Time central funciona    |
| **Multi-region DR**  | Downtime = perda financeira   | RTO de 24h aceitável     |

### Exemplo de ROI: Feature Store

**Situação sem Feature Store:**

* Cientista de dados reescreve feature para produção: **40h por modelo**
* Drift online/offline não detectado: **15% de falsos positivos**
* 3 modelos em produção
* Custo: `3 modelos × 40h × $100/h = $12k/ano` + perda de precision

**Situação com Feature Store:**

* Investimento inicial: **$50k** (infra + implementação)
* Feature reutilizável: **4h por modelo** (apenas configuração)
* Consistency garantida
* Economia: `3 modelos × 36h × $100/h × 3 releases/ano = $32k/ano`
* Payback: **~18 meses**

**Decisão:** Investir se você tem **3+ modelos em produção** ou planeja ter em 12 meses.

### Regra de ouro

Modernidade sem ROI mensurável é **hobby técnico caro**.

---

# 7️⃣ Compliance (LGPD/GDPR)

Compliance não é checkbox em formulário.  
É implementação técnica desde a fundação.

## 7.1 Right to be Forgotten

**Desafio:** Bronze imutável vs direito de deleção.

**Solução: Tombstone Events**

```json
{
  "user_id": "123",
  "event_type": "GDPR_DELETE",
  "timestamp": "2026-02-02T10:00:00Z",
  "reason": "user_request",
  "requestor_id": "compliance_team"
}
```

**Implementação:**

1. Evento de deleção publicado no Kafka
2. Silver/Gold filtram registros tombstone:

```sql
SELECT *
FROM pedidos_silver
WHERE user_id NOT IN (
  SELECT DISTINCT user_id 
  FROM gdpr_deletions
  WHERE active = true
)
```

3. **Purge físico anual** do Bronze (compaction com rewrite)

---

## 7.2 Pseudonimização

**Problema:** PII em Bronze imutável dificulta compliance.

**Solução: Hash determinístico one-way**

```python
import hashlib
import os

# Salt armazenado de forma segura (AWS Secrets Manager, Vault)
SALT = os.environ['PSEUDONYMIZATION_SALT']

def pseudonymize(valor: str, salt: str = SALT) -> str:
    """
    Pseudonimiza valor de forma determinística.
    Mesmo input sempre gera mesmo output.
    """
    return hashlib.sha256(
        f"{valor}{salt}".encode('utf-8')
    ).hexdigest()

# Uso
cpf_original = "123.456.789-00"
cpf_pseudo = pseudonymize(cpf_original)
# → "a7f3c5e9d2b4f8a1c3e6d9b2f5a8c1e4d7b0a3f6c9e2d5b8a1f4c7e0d3b6a9f2"
```

**Arquitetura:**

* **Bronze:** Apenas dados pseudonimizados
* **Mapping table:** `cpf_hash → cpf_real` (acesso restrito via IAM)
* **Right to forget:** Deletar registro do mapping (torna Bronze inutilizável)

**Policy IAM de exemplo:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/compliance-team"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::data-lake/pii_mapping/*"
    }
  ]
}
```

Apenas time de compliance acessa mapeamento real.

---

## 7.3 Classificação de Dados

**Taxonomia padrão:**

```yaml
classificacao:
  publico:
    descricao: "Sem restrição de acesso"
    exemplos: ["produto_id", "categoria", "preco"]
    
  interno:
    descricao: "Acesso apenas funcionários"
    exemplos: ["metricas_internas", "custos"]
    
  confidencial:
    descricao: "Acesso restrito a squad específico"
    exemplos: ["salarios", "estrategia_comercial"]
    
  restrito:
    descricao: "PII, dados financeiros sensíveis"
    exemplos: ["cpf", "cartao_credito", "conta_bancaria"]
    compliance: ["LGPD", "PCI-DSS"]
    retencao_minima: "5 anos"
```

**Aplicação no catálogo:**

```json
{
  "tabela": "clientes",
  "colunas": [
    {"nome": "cliente_id", "classificacao": "interno"},
    {"nome": "nome", "classificacao": "confidencial"},
    {"nome": "cpf", "classificacao": "restrito", "pii": true},
    {"nome": "email", "classificacao": "restrito", "pii": true}
  ]
}
```

---

## 7.4 Audit Logs Obrigatórios

**O que registrar:**

* Quem acessou dados PII (user_id + timestamp)
* Queries executadas em tabelas restritas
* Exportações de dados (CSV, Parquet)
* Tentativas de acesso negadas

**Implementação:**

```python
# Interceptor de queries
@log_audit
def execute_query(query: str, user: str, tables: List[str]):
    """Registra toda query antes de executar"""
    
    # Identifica tabelas com PII
    pii_tables = [t for t in tables if has_pii(t)]
    
    if pii_tables:
        audit_log.write({
            "timestamp": datetime.utcnow(),
            "user": user,
            "action": "QUERY",
            "tables": pii_tables,
            "query_hash": hashlib.sha256(query.encode()).hexdigest(),
            "ip_address": get_client_ip(),
            "approved": True
        })
    
    # Executa query
    return spark.sql(query)
```

**Retenção:** Mínimo **5 anos** (LGPD Art. 16).

**Compliance não é opcional.** É requisito operacional desde o design inicial.

---

# 8️⃣ Disaster Recovery

## 8.1 Requisitos Mínimos

* **RTO (Recovery Time Objective):** < 4 horas
* **RPO (Recovery Point Objective):** < 15 minutos

## 8.2 Implementação

* Replicação **multi-region** do Bronze (S3 Cross-Region Replication)
* Backup incremental de **state backend** (Flink/Spark checkpoints)
* Runbooks versionados no Git
* Automação de failover

## 8.3 Teste Trimestral - Checklist Executável

### Preparação (T-7 dias):

- [ ] Comunicar teste para todos stakeholders
- [ ] Validar que backups estão atualizados
- [ ] Preparar rollback plan documentado
- [ ] Reservar janela de manutenção (4h, baixa carga)

### Execução (durante janela):

- [ ] **T0:** Simular falha total da região primária (desligar compute)
- [ ] **T+15min:** Promover região secundária a primária
- [ ] **T+30min:** Validar checksums de dados (amostragem 1%)
- [ ] **T+45min:** Redirecionar tráfego (DNS/Load Balancer)
- [ ] **T+60min:** Testar pipeline crítico end-to-end
- [ ] **T+90min:** Verificar lag de replicação < 5min
- [ ] **T+120min:** Validar dashboards operacionais

### Pós-teste:

- [ ] Documentar tempo **real** de recovery (vs objetivo de 4h)
- [ ] Identificar gaps e pontos de melhoria
- [ ] Atualizar runbook com aprendizados
- [ ] Comunicar resultados para liderança

### Métricas de sucesso:

* Recovery completo em < 4h? ✅ / ❌
* Perda de dados < 15min? ✅ / ❌
* Todos stakeholders notificados? ✅ / ❌
* Zero downtime em pipelines críticos? ✅ / ❌

**Regra de ouro:**

Se você **nunca testou recovery em produção**, você não tem disaster recovery.  
Você tem plano no papel.

---

# 9️⃣ Evolução por Maturidade

Nem toda empresa precisa estar no nível 3 imediatamente.  
Evolução deve ser **incremental e intencional**.

## Nível 1 — Fundação

**Perfil:** Startups e empresas iniciais (50–200 pessoas)

**Capacidades mínimas:**

* Bronze imutável versionado
* Schema enforcement básico
* Batch funcional com orquestração
* Observabilidade básica (logs + métricas)
* Data quality manual

**Ferramentas típicas:**

* S3 + Delta Lake
* Airflow
* dbt
* CloudWatch / Datadog

**Foco:** Estrutura confiável antes de escala.

---

## Nível 2 — Escalável

**Perfil:** Scale-ups e empresas em crescimento (200–1000 pessoas)

**Capacidades adicionais:**

* CDC estruturado (Debezium + Kafka)
* Streaming stateful (Flink ou Spark Streaming)
* Feature Store offline
* Data contracts formais
* Lineage automático
* Observabilidade com SLOs

**Ferramentas típicas:**

* Kafka + Schema Registry
* Flink / Spark Structured Streaming
* Feature Store (Feast, Tecton)
* DataHub / OpenMetadata

**Foco:** ML em produção + múltiplos domínios.

---

## Nível 3 — Avançado

**Perfil:** Enterprises e empresas consolidadas (1000+ pessoas)

**Capacidades adicionais:**

* Consistência online/offline **validada** em produção
* Data Mesh operacional (3+ domínios autônomos)
* DR testado trimestralmente
* Compliance automatizado (LGPD/SOC2/ISO27001)
* Custos otimizados por produto
* Self-service analytics completo

**Ferramentas típicas:**

* Multi-region deployment
* Feature Store enterprise (custom ou Tecton)
* Observabilidade APM (Datadog, New Relic)
* Governance platform (Collibra, Alation)

**Foco:** Escala global + compliance rigoroso.

---

### Regra crítica:

**Não pule níveis.**

Tentar implementar Data Mesh (nível 3) sem ter Feature Store funcionando (nível 2) é receita para falha.

Cada nível depende das fundações do anterior.

---

# 🔟 Checklist de Modernidade Real

Use este checklist para avaliar sua arquitetura atual:

- [ ] **Replay total:** Consigo reconstruir qualquer estado histórico?
- [ ] **Bronze versionado:** Dados brutos imutáveis com time travel?
- [ ] **Feature consistente:** Definição única entre treino e inferência?
- [ ] **Contratos formais:** Data contracts versionados e validados?
- [ ] **Observabilidade com consequência:** SLOs com alertas + postmortems?
- [ ] **Custos mensuráveis:** Sei o custo de cada pipeline/produto?
- [ ] **DR testado:** Último teste de failover foi há < 3 meses?
- [ ] **Compliance by design:** LGPD implementado desde Bronze?

### Interpretação:

* **8/8 checks:** Arquitetura moderna consolidada ✅
* **5-7 checks:** Arquitetura em evolução, foco nos gaps 🟡
* **< 5 checks:** Arquitetura ainda tradicional, priorizar fundações 🔴

---

## Conclusão Final

Se sua arquitetura só funciona no PowerPoint, ela não é moderna.

**Ela é frágil.**

---

# 📚 Glossário Técnico

| Termo            | Significado                                    | Exemplo                                    |
| ---------------- | ---------------------------------------------- | ------------------------------------------ |
| **CDC**          | Change Data Capture - captura de mudanças     | Debezium capturando INSERTs do PostgreSQL  |
| **Tombstone**    | Marcador de deleção lógica                     | `{"user_id": "123", "deleted": true}`      |
| **Drift**        | Divergência entre treino e produção            | Feature calculada diferente em cada ambiente |
| **Lineage**      | Rastreamento de origem e transformações        | `pedidos_raw` → `pedidos_silver` → `dashboard` |
| **Time Travel**  | Consulta de estado histórico                   | `SELECT * FROM tabela VERSION AS OF '2026-01-01'` |
| **Exactly-once** | Processamento sem duplicatas                   | Flink checkpoints garantem                 |
| **State Backend**| Armazenamento de estado em streaming           | RocksDB local + S3 para recovery           |
| **Idempotência** | Reexecução segura sem efeitos colaterais       | `INSERT OVERWRITE` vs `INSERT INTO`        |
| **SLI**          | Service Level Indicator (métrica)              | P95 latência = 4.2min                      |
| **SLO**          | Service Level Objective (objetivo)             | P95 latência < 5min                        |
| **SLA**          | Service Level Agreement (acordo com penalidade)| Uptime > 99.9% ou crédito                  |
| **Breaking Change**| Mudança incompatível com versão anterior     | Remover campo obrigatório                  |
| **Backfill**     | Reprocessamento histórico de dados             | Recalcular features dos últimos 2 anos     |
| **Compaction**   | Consolidação de arquivos pequenos              | Merge de 1000 arquivos em 10 otimizados    |

---

**Arquitetura moderna não é sobre usar as ferramentas mais novas.**

É sobre construir sistemas que **resistem ao crescimento, à auditoria, ao erro humano e ao tempo**.

**Ferramentas mudam.**  
**Capacidades permanecem.**

---

**Sobre o autor:**  
Cristiano Lopes

**Feedback e discussões:**  
cristianolopes.ti@gmail.com, https://www.linkedin.com/in/cristianolopesia/

**Última atualização:** Fevereiro 2026
