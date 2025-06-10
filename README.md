Claro! Aqui está o README.md com os links internos corrigidos para navegação perfeita no GitHub. Segui o padrão de links internos do GitHub, considerando acentuação e numeração dos títulos.

---

```markdown
# 3C-s

**Método 3 C’s**  
Arquitetura Determinística de Diálogo com n8n + Supabase e RAG como Garantia de Exatidão

---

## Sumário

1. [Resumo](#1-resumo)
2. [Visão Executiva](#2-visão-executiva)
3. [Modelo Conceitual](#3-modelo-conceitual)
4. [Dados: Supabase Relacional + Vetor Store](#4-dados-supabase-relacional--vetor-store)
5. [Orquestração n8n](#5-orquestração-n8n)
6. [Algoritmo do Compositor](#6-algoritmo-do-compositor)
7. [Pipeline Turno-a-Turno (Exemplo)](#7-pipeline-turno-a-turno-exemplo)
8. [Governança, Segurança & Conformidade](#8-governança-segurança--conformidade)
9. [Performance & Escalabilidade](#9-performance--escalabilidade)
10. [Validação Experimental](#10-validação-experimental)
11. [Limitações & Mitigações](#11-limitações--mitigações)
12. [Conclusão](#12-conclusão)
13. [Referências](#13-referências)

---

## 1. Resumo

> Apresenta-se uma estratégia onde toda criatividade linguística fica confinada às bordas do sistema. No núcleo, impera lógica determinística baseada em: Componentes (chave = valor), Context[...]

![Captura de tela 2025-06-09 232928](https://github.com/user-attachments/assets/ab21df32-7c16-4e46-a52a-989c2e446633)

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

![Captura de tela 2025-06-09 234350](https://github.com/user-attachments/assets/eaaa2ccc-848c-4c44-8521-6217c3001a23)

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

RAG deixa de ser mero complemento e ancora o LLM em fatos imutáveis, viabilizando núcleo 100% determinista. Com n8n e Supabase, equipes de dados podem construir, auditar e escalar chatbots conf[...]

---

## 13. Referências

1. [Tools Agent Node – n8n Docs](https://docs.n8n.io)
2. [Supabase Vector Store Node – n8n Docs](https://docs.n8n.io)
3. [RAG as Tools – n8n Community](https://community.n8n.io)
4. [AI Agent Node – n8n Docs](https://docs.n8n.io)
5. [Supabase Node – n8n Docs](https://docs.n8n.io)
6. [Understand AI Tools – n8n Docs](https://docs.n8n.io)
```

---
