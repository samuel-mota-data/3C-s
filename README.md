# 3C-s

Método 3 C’s
Arquitetura Determinística de Diálogo com n8n + Supabase e RAG como Garantia de Exatidão
________________________________________
1 · Abstract
Apresenta-se uma estratégia na qual toda criatividade linguística fica confinada às bordas do sistema. Ao centro, impera lógica determinística baseada em: Componentes (chave = valor), Contexto-grafo (rede de nós já validados) e Compositor (algoritmo que decide perguntas e ações). A exatidão do núcleo é obtida com RAG (Retrieval-Augmented Generation): antes de cada decisão, o agente consulta um vetor store que devolve somente documentos/fatos canônicos, eliminando variação estocástica. A solução usa n8n como orquestrador (AI Agent node + Supabase Vector Store) e Supabase como banco relacional + vetorial.
________________________________________
2 · Visão Executiva
1.	Portas humanizadas:
o	Entrada—NLU (regex ou LLM) → pares exatos.
o	Saída—NL Gen (LLM temperature = 0) → resposta amigável.
2.	Núcleo 100 % determinista:
o	Tabelas SQL mantêm o estado binário dos Componentes.
o	Grafo só contém combinações permitidas (incompatibilidades explícitas).
o	Compositor age apenas quando o grafo satisfaz a regra de produção.
3.	Por que RAG garante determinismo:
o	Cada consulta ao vetor store retorna o mesmo conjunto de documentos para o mesmo embedding → mesmas instruções → mesmas decisões.
o	Elimina “alucinação” do LLM; o texto decisório deriva de fatos versionados.
________________________________________
3 · Modelo Conceitual
Entidade	Descrição	Persistência
Componente	Par chave = valor; flag ligado ∈ {0, 1}.	Tabela components, 1 linha por chave.
Contexto-grafo	Subgrafo GtG_t induzido pelos componentes ligados; arestas vêm de E_base.	Tabela context.
Compositor	Algoritmo determinístico (AI Agent) que, com ajuda de RAG, escolhe a próxima pergunta ou executa ação final.	Workflow n8n.
________________________________________
4 · Dados: Supabase Relacional + Vetor Store
create table components (
  id serial primary key,
  nome text unique,
  tipo text,
  incompat int[]           -- IDs incompatíveis
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
  payload jsonb            -- copy literal do grafo final
);
Triggers impedem ligar componentes incompatíveis; cada alteração é logada em context_history.
________________________________________
5 · Orquestração n8n
(Client) ➜ Webhook
           │
           ▼
   Extractor (OpenAI/Regex)
           │  ΔComponentes
           ▼
   Supabase (UPSERT)
           │
           ▼
 AI Agent (Compositor)
   ├─ Supabase DB Tool
   └─ Supabase Vector Store Tool  ← RAG determinístico
           │
    ask_next ↺            run_action ➜ HTTP/Email
•	O AI Agent node funciona como Tools Agent em todas as versões ≥ 1.82; basta conectar a ferramenta vetor store (docs.n8n.io, docs.n8n.io).
•	Consulta vetorial é feita pelo Supabase Vector Store node, ligado diretamente ao conector tools do agente (docs.n8n.io).
________________________________________
6 · Algoritmo do Compositor (simplificado)
def compositor(graph, obrigatorios, ordem):
    docs = retrieve_from_vector_store(graph)  # RAG -> sempre mesmos docs
    if falta_obrigatorio(graph, obrigatorios):
        cid = proximo_compatível(graph, obrigatorios, ordem)
        return ask_component(cid)
    return run_action(graph, docs)            # grafo válido, docs canon.
•	As mesmas entradas (graph) + mesmo vetor store → mesmas docs → mesma decisão → reprodutibilidade garantida.
________________________________________
7 · Pipeline Turno-a-Turno (Exemplo Genérico)
Turno	Evento	Componentes Ligados	Ação do Compositor
1	“Quero reservar sala”	—	Pergunta data.
2	data=12/07	data	Pergunta quantidade.
3	quantidade=15	data, quantidade	Consultar vetor store → verificar regra → acionar /reserva.
4	Confirmação	data, quantidade	Responde friendly via NL Gen.
________________________________________
8 · Governança, Segurança & Conformidade
•	Row-Level Security garante que cada conversa só leia seus próprios contextos.
•	RAG Canônico — apenas payloads verificados entram no vetor store, impedindo vazamento de PII.
•	Logs SQL + snapshots possibilitam trilha completa de auditoria para SOX/GDPR.
________________________________________
9 · Performance & Escalabilidade
Camada	Estratégia
n8n	Mode queue + Redis; workflows leves (parser & agent).
Supabase	pgbouncer para pool; índices HNSW em embedding.
AI	temperature=0, top_p=1 — reduz variação e custo de tokens.
________________________________________
10 · Validação Experimental
1.	Dataset – 2 000 diálogos sintéticos + 200 reais.
2.	KPIs – turnos médios até ação, tempo p95, incidência de incompatibilidade.
3.	Critério de sucesso – erro ≤ 0, SLA < 2 s, 25 % menos turnos que baseline probabilístico.
________________________________________
11 · Limitações & Mitigações
Limitação	Mitigação
Parser pode coletar valor errado	Porta de Entrada devolve pergunta de confirmação.
Ampliação do catálogo exige migração SQL	Gerar script de seeds versionado.
Vetor store cresce indefinidamente	Política de TTL + compressão de embeddings.
________________________________________
12 · Conclusão
RAG deixa de ser mero complemento: ele ancora o LLM em fatos imutáveis e torna viável um núcleo 100 % determinista. Com n8n e Supabase, equipes de dados podem construir, auditar e escalar chatbots corporativos sem temer variações estocásticas — e ainda preservar uma experiência de linguagem natural nas bordas.
________________________________________
Referências
1.	Tools Agent Node – n8n Docs (docs.n8n.io)
2.	Supabase Vector Store Node – n8n Docs (docs.n8n.io)
3.	RAG as Tools – n8n Community (community.n8n.io)
4.	AI Agent Node – n8n Docs (docs.n8n.io)
5.	Supabase Node – n8n Docs (docs.n8n.io)
6.	Understand AI Tools – n8n Docs (docs.n8n.io)

