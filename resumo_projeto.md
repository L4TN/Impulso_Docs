# IARA - Resumo do Projeto

## Meta Atual

Criar um modelo ER conceitual limpo usando notação Peter Chen para um sistema de chatbot WhatsApp chamado "IARA". O usuário forneceu um schema PostgreSQL e múltiplos JSONs de workflows n8n, e pediu para:
1. Identificar quais entidades são realmente relevantes
2. Remover as não utilizadas
3. Criar um diagrama ER adequado com relacionamentos

## Instruções

- **Nomeação de entidades**: Modelo conceitual usa nomes em UPPERCASE em inglês (LEAD, CONVERSATION, MESSAGE_HISTORY)
- **Formato ER**: Diagrama Mermaid.js em arquivo markdown
- **Escopo**: Apenas o que está em PRODUÇÃO (PRD), não planejado/futuro
- **Tabelas a REMOVER**: CAMPANHA, CAMPANHA_LOGS (movidas para Chatwoot), LEAD (órfã), APPLICATION_CHANNEL, historico_mensagens_simples
- **Tabelas a MANTER**: LEAD, CONVERSATION, MESSAGE_HISTORY, CHANNEL_CONFIG, DOCUMENT, TOKEN_COST
- **Tabelas a IGNORAR**: empresa, usuario, agente, agente_config, canal_whatsapp, prompt (não usados em workflows)
- **Decisões do usuário**:
  - Manter TOKEN_COST (importante para billing)
  - Remover SIMPLE_MESSAGE (historico_mensagens_simples não necessário)
  - Remover AGENT_STATUS (inativação acontece via departamento no Chatwoot, não Redis)

## Descobertas

**Achado crítico da análise de workflows n8n**:

Os workflows n8n usam tabelas **completamente diferentes** do schema original do usuário:

| Workflows REALMENTE Usam | Schema Original do Usuário (NÃO usado) |
|------------------------|----------------------------------|
| `documents` (Supabase Vector) | `empresa` |
| `leads` | `usuario` |
| `mensagens` (como CONVERSATION) | `agente`, `agente_config` |
| `historico_mensagens` | `canal_whatsapp` |
| `aplicacao_canal` (como CHANNEL_CONFIG) | `prompt` |
| `custo_tokens` | `leads` (versão original) |
| Redis para memória | `campanha`, `campanha_logs` |

**Workflows ativos (active:true, isArchived:false)**:
- IARA_01_Rabbit_PRD - Ingestão de mensagens RabbitMQ
- IARA_02_Flow_PRD - Fluxo principal, usa `aplicacao_canal` para config Chatwoot
- IARA_03_Message_PRD - Apenas Redis
- IARA_05_Mensagem_Agente - Registro de leads (upsert por telefone)
- IARA_06_Agente_e_Ferramentas - RAG com tabela `documents`
- IARA_07 - Histórico de mensagens (INSERT em `historico_mensagens`)
- Job Disparador - Campanhas (a ser removido)

**Tabelas usadas por workflow ativo**:
- `historico_mensagens_simples` - IARA_01, IARA_02 (buffer de mensagens recentes)
- `aplicacao_canal` - IARA_02 (chatwoot_inbox_id, chatwoot_token, department_id)
- `leads` - IARA_05 (upsert por telefone)
- `documents` - IARA_06 (busca相似idade RAG)
- `mensagens` - IARA_07 (upsert rastreamento de conversa)
- `historico_mensagens` - IARA_07 (histórico completo)

**Workflows arquivados**: IARA_PRD, IARA_V2 (também usam custo_tokens, mas arquivados)

## Accomplished

1. ✅ Analisado schema PostgreSQL original do usuário (DDL)
2. ✅ Identificados relacionamentos FK explícitos no schema original
3. ✅ Lidos e analisados 15+ arquivos JSON de workflow n8n usando sub-agents paralelos
4. ✅ Descoberta incompatibilidade entre schema original e uso real nos n8n
5. ✅ Confirmadas tabelas NÃO usadas: empresa, usuario, agente, agente_config, canal_whatsapp, prompt
6. ✅ Usuário confirmou que essas tabelas não são usadas
7. ✅ Criada análise consolidada de tabelas por status de workflow (ativo vs arquivado)
8. ✅ Usuário respondeu perguntas sobre escopo ER, inclusão de tabelas, tratamento de conversas
9. ✅ Criado `modelo_er_conceitual.md` com modelo final de 6 entidades

## Arquivos Relevantes

**Criados**:
- `C:\Users\l4tni\source\l4tn_pkm\pages\10_Projetos\10.1_Impulso\Impulso_Docs\modelo_er_conceitual.md` - Documento do modelo ER final

**Analisados (workflows n8n)**:
- `C:\Users\l4tni\source\l4tn_pkm\pages\10_Projetos\10.1_Impulso\Impulso_Docs\backups\` - Diretório contendo todos os JSONs de workflow

Arquivos-chave analisados:
- `IARA_01_Rabbit_PRD.json` - Ingestão RabbitMQ
- `IARA_02_Flow_PRD.json` - Fluxo principal de mensagens (usa aplicacao_canal)
- `IARA_05_Mensagem_Agente.json` - Registro de leads
- `IARA_06_Agente_e_Ferramentas.json` - Agente RAG
- `IARA_07.json` - Histórico de mensagens
- `00. Configurações.json` - Cria tabelas n8n_* (arquivado)

**Schema original (mencionado na conversa)**:
- DDL PostgreSQL com tabelas: empresa, usuario, agente, agente_config, canal_whatsapp, prompt, documents, leads, mensagens, historico_mensagens, aplicacao_canal, campanha, campanha_logs

## Próximos Passos

O modelo ER está completo. Próximos passos potenciais:
- Revisar o `modelo_er_conceitual.md` para precisão
- Gerar modelo físico (DDL) a partir do modelo conceitual
- Discutir decisões de normalização ou desnormalização
- Abordar a questão anterior do usuário sobre se o modelo faz sentido de negócio
