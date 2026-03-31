# Modelo ER Conceitual - Sistema Impulso IARA

## Visão Geral

Este documento descreve o modelo entidade-relacionamento conceitual do sistema de atendimento automatizado IARA, baseado na análise dos workflows n8n em produção.

---

## Entidades

### 1. LEAD
**Tabela**: `leads`  
**Descrição**: Prospectos e contatos que interagem com o sistema via WhatsApp.

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Data de criação |
| `Telefone` | VARCHAR(40) | NOT NULL, UNIQUE | Número de WhatsApp (formato E.164) |
| `Nome` | TEXT | | Nome do lead (opcional) |

**Relacionamentos**:
- 1 LEAD pode ter N CONVERSATIONs
- 1 LEAD pode ter N MESSAGE_HISTORYs

---

### 2. CONVERSATION
**Tabela**: `mensagens`  
**Descrição**: Sessão de conversa entre um lead e o agente em um canal específico.

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `telefone` | VARCHAR(40) | NOT NULL, FK → LEAD(Telefone) | Lead responsável |
| `id_conversa` | VARCHAR(255) | NOT NULL | ID da conversa no Chatwoot |
| `nome_app` | VARCHAR(100) | NOT NULL | Nome da aplicação/canal (ex: "SDR-IMOB") |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Início da conversa |
| `update_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Última atualização |

**Índices**:
- `UNIQUE(telefone, nome_app)` - Uma conversa ativa por canal
- `INDEX(id_conversa)`

**Relacionamentos**:
- N CONVERSATION pertencem a 1 LEAD
- 1 CONVERSATION pode ter N MESSAGE_HISTORYs
- 1 CONVERSATION é configurado por 1 CHANNEL_CONFIG

---

### 3. MESSAGE_HISTORY
**Tabela**: `historico_mensagens`  
**Descrição**: Registro completo de cada troca de mensagens entre lead e agente.

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Timestamp da troca |
| `telefone` | VARCHAR(40) | NOT NULL, FK → LEAD(Telefone) | Lead participante |
| `id_conversa` | VARCHAR(255) | NOT NULL, FK → CONVERSATION(id_conversa) | Conversa |
| `nome_usuario` | TEXT | | Nome do usuário no Chatwoot |
| `nome_app` | VARCHAR(100) | NOT NULL | Aplicação/Canal |
| `mensagem_usuario` | TEXT | | Conteúdo da mensagem do lead |
| `mensagem_agente` | TEXT | | Conteúdo da resposta do agente |
| `message_type` | VARCHAR(50) | | Tipo: text, audio, image, document |
| `ativo` | BOOLEAN | NOT NULL, DEFAULT TRUE | Status do bot na conversa |

**Índices**:
- `INDEX(telefone)`
- `INDEX(id_conversa)`
- `INDEX(created_at)`

**Relacionamentos**:
- N MESSAGE_HISTORY pertencem a 1 LEAD
- N MESSAGE_HISTORY pertencem a 1 CONVERSATION

---

### 4. CHANNEL_CONFIG
**Tabela**: `aplicacao_canal`  
**Descrição**: Configuração de integração com o Chatwoot (inbox, conta, token, departamentos).

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `chatwoot_inbox_id` | VARCHAR(100) | NOT NULL, UNIQUE | ID do inbox no Chatwoot |
| `chatwoot_account_id` | VARCHAR(100) | NOT NULL | ID da conta Chatwoot |
| `chatwoot_token` | TEXT | NOT NULL | Token de API do Chatwoot |
| `chatwoot_teams_department_id` | VARCHAR(100) | | ID do departamento para escalação |

**Relacionamentos**:
- 1 CHANNEL_CONFIG configura N CONVERSATIONs

---

### 5. DOCUMENT
**Tabela**: `documents`  
**Descrição**: Base de conhecimento para RAG (Retrieval Augmented Generation).

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `titulo` | TEXT | | Título do documento |
| `content` | TEXT | NOT NULL | Conteúdo extraído (texto ou markdown) |
| `metadata` | JSONB | | Metadados (file_id, version, creator, etc.) |
| `embedding` | VECTOR(1536) | | Vetor de embedding OpenAI |

**Índices**:
- `INDEX USING ivfflat (embedding vector_cosine_ops)` - Para similarity search

---

### 6. TOKEN_COST
**Tabela**: `custo_tokens`  
**Descrição**: Rastreamento de custos de tokens LLM para billing.

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | BIGSERIAL | PK | Identificador único |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | Timestamp |
| `Workflow` | VARCHAR(255) | NOT NULL | Nome do workflow que gerou o custo |
| `Input` | TEXT | | Prompt enviado ao LLM |
| `Output` | TEXT | | Resposta do LLM |
| `PromptTokens` | INTEGER | | Quantidade de tokens de input |
| `CompletationTokens` | INTEGER | | Quantidade de tokens de output |
| `CachedTokens` | INTEGER | | Tokens em cache (se disponível) |
| `Cost` | DECIMAL(10,6) | | Custo em USD |

**Índices**:
- `INDEX(created_at)`
- `INDEX(Workflow)`

---

## Relacionamentos

```mermaid
erDiagram
    LEAD ||--o{ CONVERSATION : "has"
    LEAD ||--o{ MESSAGE_HISTORY : "communicates_with"
    CONVERSATION ||--o{ MESSAGE_HISTORY : "contains"
    CHANNEL_CONFIG ||--o{ CONVERSATION : "configures"
    DOCUMENT }|..|| AGENT : "provides_knowledge_to"
    TOKEN_COST ||--|| MESSAGE_HISTORY : "tracks_cost_of"

    LEAD {
        bigint id PK
        timestamptz created_at
        varchar(40) Telefone UK NOT NULL
        text Nome
    }

    CONVERSATION {
        bigint id PK
        varchar(40) telefone FK NOT NULL
        varchar(255) id_conversa UK NOT NULL
        varchar(100) nome_app NOT NULL
        timestamptz created_at
        timestamptz update_at
    }

    MESSAGE_HISTORY {
        bigint id PK
        timestamptz created_at
        varchar(40) telefone FK NOT NULL
        varchar(255) id_conversa FK NOT NULL
        text nome_usuario
        varchar(100) nome_app NOT NULL
        text mensagem_usuario
        text mensagem_agente
        varchar(50) message_type
        boolean ativo DEFAULT TRUE
    }

    CHANNEL_CONFIG {
        bigint id PK
        varchar(100) chatwoot_inbox_id UK NOT NULL
        varchar(100) chatwoot_account_id NOT NULL
        text chatwoot_token NOT NULL
        varchar(100) chatwoot_teams_department_id
    }

    DOCUMENT {
        bigserial id PK
        text titulo
        text content NOT NULL
        jsonb metadata
        vector embedding
    }

    TOKEN_COST {
        bigserial id PK
        timestamptz created_at
        varchar(255) Workflow NOT NULL
        text Input
        text Output
        integer PromptTokens
        integer CompletationTokens
        integer CachedTokens
        decimal(10,6) Cost
    }
```

---

## Regras de Negócio

### 1. Ciclo de Vida do Lead
1. Lead envia primeira mensagem via WhatsApp
2. Sistema verifica se lead existe por `Telefone`
3. Se não existe → INSERT em LEAD
4. Cria ou atualiza CONVERSATION (`telefone + nome_app`)

### 2. Fluxo de Mensagens
1. Mensagem recebido → INSERT em MESSAGE_HISTORY (`mensagem_usuario`)
2. Agente processa e responde
3. UPDATE em MESSAGE_HISTORY com `mensagem_agente`
4. UPDATE em CONVERSATION com `update_at`

### 3. Escalação Humana
1. Sistema identifica necessidade de escalação
2. Consulta CHANNEL_CONFIG para obter `chatwoot_teams_department_id`
3. Conversation permanece ativa mas `ativo = false`
4. Lead é atendido por agente humano

### 4. RAG (Documents)
1. Documentos são carregados e processados
2. Embeddings são gerados via OpenAI
3. Busca usa similaridade vetorial para encontrar contexto relevante
4. Contexto é injetado no prompt do agente

---

## Tabelas Removidas do Modelo Original

| Tabela Original | Motivo da Remoção |
|-----------------|-------------------|
| `historico_mensagens_simples` | Substituído por MESSAGE_HISTORY completo |
| `empresa` | Configuração via variáveis do workflow |
| `usuario` | Não utilizado nos workflows ativos |
| `agente` | Não utilizado nos workflows ativos |
| `agente_config` | Não utilizado nos workflows ativos |
| `canal_whatsapp` | Funcionalidade coberta por CHANNEL_CONFIG |
| `prompt` | Configuração inline no workflow |
| `campanha` | Sistema separado de campanhas |
| `campanha_logs` | Sistema separado de campanhas |
| `leads` original | Substituído por LEAD com campos corretos |
| `n8n_*` | Tabelas internas do n8n (não são entidades de negócio) |

---

## Notas de Implementação

### Redis (Não persistido em tabela)
- **Conversa Memory**: `{chat_id}_mem_agent` - Histórico da conversa para contexto do LLM
- **Fila de Mensagens**: `{chat_id}_mem` - Buffer temporário de mensagens
- **Status**: Gerenciado via `ativo` em MESSAGE_HISTORY

### Chatwoot Integration
- Conversation ID é gerenciado externamente pelo Chatwoot
- Sistema sincroniza via webhooks
- Departamento de escalação configurado em CHANNEL_CONFIG

### Supabase Vector Store
- DOCUMENTS usa Supabase pgvector para embeddings
- Busca por similaridade de cosseno
- Metadata contém file_id para rastreamento

---

*Documento gerado em 2026-03-31*
*Versão: 1.0*
