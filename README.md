
# 3C-s

**Método 3 C’s**  
Arquitetura Determinística de Diálogo com n8n + Supabase e RAG como Garantia de Exatidão

---

## Sumário

1. [Resumo](#resumo)
2. [Visão Executiva](#visão-executiva)
3. [Modelo Conceitual](#modelo-conceitual)
4. [Dados: Supabase Relacional + Vetor Store](#dados-supabase-relacional--vetor-store)
5. [Orquestração n8n](#orquestração-n8n)
6. [Algoritmo do Compositor](#algoritmo-do-compositor)
7. [Pipeline Turno-a-Turno (Exemplo)](#pipeline-turno-a-turno-exemplo)
8. [Governança, Segurança & Conformidade](#governança-segurança--conformidade)
9. [Performance & Escalabilidade](#performance--escalabilidade)
10. [Validação Experimental](#validação-experimental)
11. [Limitações & Mitigações](#limitações--mitigações)
12. [Conclusão](#conclusão)
13. [Referências](#referências)

---

## 1. Resumo

> Apresenta-se uma estratégia onde toda criatividade linguística fica confinada às bordas do sistema. No núcleo, impera lógica determinística baseada em: Componentes (chave = valor), Contexto-grafo e Compositor (AI Agent) com garantia de exatidão via RAG.
>
> 
┌──────────────────────────┐   (1) texto natural
│ Porta de Entrada         │ ◀────────── Cliente
│  n8n • NLU (LLM/Regex)   │
└────────────┬─────────────┘
             │ ΔComponentes
┌────────────▼─────────────┐
│ Núcleo 3 C’s (n8n)       │
│ • Componentes (SQL)      │
│ • Contexto-grafo (SQL)   │
│ • Compositor (AI Agent)  │
└────────────┬─────────────┘
             │ Resultado estruturado
┌────────────▼─────────────┐   (2) texto natural
│ Porta de Saída           │ ──────────▶ Cliente
│  n8n • NL-Gen (LLM/templ)│
└──────────────────────────┘

---

## 2. Visão Executiva

- **Portas humanizadas:**
    - **Entrada:** NLU (regex ou LLM) → pares exatos
    - **Saída:** NL Gen (LLM, temperature = 0) → resposta amigável
- **Núcleo 100% determinista:**
    - SQL armazena estados binários dos componentes
    - Grafo contém apenas combinações permitidas (incompatibilidades explícitas)
    - Compositor só age em grafos válidos
- **Por que RAG garante determinismo:**
    - Mesmo embedding → mesmos documentos → mesmas decisões
    - Elimina “alucinação”; decisões baseadas em fatos versionados

---

## 3. Modelo Conceitual

| Entidade         | Descrição                                                                 | Persistência             |
|------------------|---------------------------------------------------------------------------|-------------------------|
| Componente       | Par chave = valor; flag ligado ∈ {0, 1}                                   | Tabela `components`     |
| Contexto-grafo   | Subgrafo induzido pelos componentes ligados                               | Tabela `context`        |
| Compositor       | Algoritmo determinístico (AI Agent) que decide próxima ação               | Workflow n8n            |

---

## 4. Dados: Supabase Relacional + Vetor Store

```sql
create table components (
  id serial primary key,
  nome text unique,
  tipo text,
  incompat int[] -- IDs incompatíveis
);

create table context (
  id_conversa uuid,
  component_id int references components(id),
  valor text,
  ligado boolean default false,
  primary key (id_conversa, component_id)
);

create table casos_vector (
  id uuid primary key,
  embedding vector(1536),
  payload jsonb -- copy literal do grafo final
);
```
> Triggers impedem ligar componentes incompatíveis; cada alteração é logada em `context_history`.

---

## 5. Orquestração n8n


graph TD
A[Client] --> B[Webhook]
B --> C[Extractor (OpenAI/Regex)]
C --> D[Supabase (UPSERT)]
D --> E[AI Agent (Compositor)]
E -->|Supabase DB Tool| F[Supabase]
E -->|Vector Store Tool| G[Supabase Vector Store]
E --> H{ask_next}
E --> I{run_action → HTTP/Email}


- O AI Agent funciona como Tools Agent (n8n ≥ 1.82), integrado ao vetor store.

---

## 6. Algoritmo do Compositor

```python
def compositor(graph, obrigatorios, ordem):
    docs = retrieve_from_vector_store(graph)  # RAG -> sempre mesmos docs
    if falta_obrigatorio(graph, obrigatorios):
        cid = proximo_compatível(graph, obrigatorios, ordem)
        return ask_component(cid)
    return run_action(graph, docs)  # grafo válido, docs canon.
```
> As mesmas entradas (graph) + mesmo vetor store → mesmas docs → mesma decisão → reprodutibilidade garantida.

---

## 7. Pipeline Turno-a-Turno (Exemplo)

| Turno | Evento                   | Componentes Ligados | Ação do Compositor                    |
|-------|--------------------------|---------------------|----------------------------------------|
| 1     | "Quero reservar sala"    | —                   | Pergunta data                          |
| 2     | data=12/07               | data                | Pergunta quantidade                    |
| 3     | quantidade=15            | data, quantidade    | Consultar vetor store, acionar reserva |
| 4     | Confirmação              | data, quantidade    | Responde via NL Gen                    |

---

## 8. Governança, Segurança & Conformidade

- **Row-Level Security:** cada conversa lê só seu contexto
- **RAG Canônico:** apenas payloads verificados no vetor store (protege PII)
- **Logs e snapshots:** trilha de auditoria para SOX/GDPR

---

## 9. Performance & Escalabilidade

| Camada   | Estratégia                                       |
|----------|--------------------------------------------------|
| n8n      | Mode queue + Redis; workflows leves              |
| Supabase | pgbouncer para pool, índices HNSW em embedding   |
| AI       | temperature=0, top_p=1 — reduz variação e custo  |

---

## 10. Validação Experimental

1. **Dataset:** 2.000 diálogos sintéticos + 200 reais
2. **KPIs:** turnos médios, tempo p95, incidência de incompatibilidade
3. **Critério de sucesso:** erro ≤ 0, SLA < 2s, 25% menos turnos que baseline probabilístico

---

## 11. Limitações & Mitigações

| Limitação                        | Mitigação                                      |
|----------------------------------|------------------------------------------------|
| Parser pode coletar valor errado | Porta de Entrada devolve pergunta de confirmação|
| Ampliação exige migração SQL     | Gerar script de seeds versionado               |
| Vetor store cresce indefinido    | TTL + compressão de embeddings                 |

---

## 12. Conclusão

RAG deixa de ser mero complemento e ancora o LLM em fatos imutáveis, viabilizando núcleo 100% determinista. Com n8n e Supabase, equipes de dados podem construir, auditar e escalar chatbots confiáveis.

---

## 13. Referências

1. [Tools Agent Node – n8n Docs](https://docs.n8n.io)
2. [Supabase Vector Store Node – n8n Docs](https://docs.n8n.io)
3. [RAG as Tools – n8n Community](https://community.n8n.io)
4. [AI Agent Node – n8n Docs](https://docs.n8n.io)
5. [Supabase Node – n8n Docs](https://docs.n8n.io)
6. [Understand AI Tools – n8n Docs](https://docs.n8n.io)

