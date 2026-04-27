# Agente Camaleão 🦎 — Automação de Vendas WhatsApp

Sistema de automação inteligente multicanal (Multitenant) para WhatsApp, utilizando **WAHA**, **n8n** e **Claude IA (Anthropic)**.

## 🚀 Arquitetura

O projeto é baseado em um fluxo dinâmico que identifica o nicho do lead e adapta a persona de atendimento (Sofia, Ana ou Alex).

- **WAHA**: Interface de API para WhatsApp (Webhooks e Mensagens).
- **n8n**: Orquestrador do fluxo e lógica de negócio.
- **Claude 4.6 (Anthropic)**: Cérebro da automação, responsável pela geração de respostas naturais e conversão.
- **PostgreSQL**: Persistência de sessões, histórico de conversas e estados do bot.

## 🛠️ Componentes Principais

- `workflow_backup.json`: Exportação completa do workflow n8n configurado.
- `BUSINESS_LOGIC.md`: Documentação detalhada das regras de ouro, personas e fases do funil.
- `README.md`: Este guia de referência.

## 📋 Configuração e Credenciais

Para o funcionamento correto, as seguintes credenciais devem ser configuradas no n8n:

| Tipo | Nome Sugerido | Descrição |
| :--- | :--- | :--- |
| **HTTP Header Auth** | `WAHA Vitrine API Key` | Autenticação para a instância do WAHA. |
| **PostgreSQL** | `Postgres account` | Conexão com o banco de dados de sessões. |
| **Anthropic API** | `Claude API` | Chave de API para acesso aos modelos Claude. |

## 🧠 Personas Disponíveis

1.  **Sofia (Estética)**: Focada em clínicas e agendamento de avaliações.
2.  **Ana (Saúde)**: Secretaria médica para triagem e agendamentos.
3.  **Alex (Marketing)**: Especialista em automação e geração de projetos.

---
*Desenvolvido por Alexandre Fernandes / Agbotia*
