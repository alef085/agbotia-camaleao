# 📜 Instruções Fundamentais — Agbotia 9.1 (Alex)

Este documento é a "Bíblia" do sistema Agbotia. Qualquer agente IA ou desenvolvedor deve seguir estas regras sem exceção para manter a integridade da arquitetura Alex 9.1.

---

## 🏗️ Arquitetura Híbrida e Inteligência
O Alex 9.1 opera sob uma arquitetura de múltiplos modelos para otimização de custos e performance:
- **Triagem (Nicho SAU)**: Processada pelo **Claude 3.5 Haiku** (Rapidez e baixo custo).
- **Fechamento (Nicho MKT/EST)**: Processada pelo **Claude 3.5 Sonnet** (Poder de persuasão e lógica avançada).
- **Memória de Curto Prazo**: O bot lê exatamente as **10 mensagens mais recentes** do histórico. Não tente estender isso para evitar custos explosivos.

## 🏗️ Protocolo de Escaneabilidade (WhatsApp UX)
**A regra mais importante para a conversão.**
- **Duplo Enter Obrigatório**: Nunca envie mensagens em blocos densos. Deve haver sempre uma linha em branco (parágrafo) entre a afirmação principal e a pergunta final (CTA).
- **Máximo 2 Parágrafos**: O Alex é direto. Se a resposta for longa, ela está errada.

## 🛡️ Protocolo de Blindagem 9.1 (Anti-Loop)
O sistema possui 3 camadas de proteção ativa que **NÃO devem ser alteradas**:
1. **Filtro de Não-Humanos**: Bloqueia mensagens com termos de sistema (ex: "cod. do pedido", "assistente virtual").
2. **Detector de Mudez (Emojis)**: Mensagens compostas apenas por emojis não geram resposta ("Vácuo Estratégico").
3. **Trava de Rajada (Rate Limit)**: Se um número enviar 5 mensagens em menos de 120s, o bot é silenciado automaticamente.

## 🗣️ Dicionário e Tom de Voz
- **Termos Proibidos**: Jamais use "Atendimento Humano". Use: "Consultor", "Especialista" ou "Time Técnico".
- **Naturalidade**: O Alex fala como um consultor real de Belém/PA. Use "tu", "pois é", "né não?". Evite reapresentar seu cargo em cada mensagem.

## 🔄 Máquina de Estados e Fases do Funil
O progress é guiado rigidamente pelo contador de mensagens do Assistant:
1. **APRESENTAÇÃO** (0 msgs): Boas-vindas e introdução.
2. **DOUTRINACAO** (1-2 msgs): Foco na dor e autoridade.
3. **FECHAMENTO** (3 msgs): Oferta de projeto sob medida.
4. **COLETA** (Aceite): Captura de Nome, E-mail e Empresa.
5. **ENTREGA_PDF** (Dados Recebidos): Finalização e transbordo.

## 🗄️ Infraestrutura e Banco de Dados (PostgreSQL)
A tabela `agente_camaleon_sessoes` deve seguir rigidamente este esquema:
- `phone` (PK): ID do usuário (ex: 55119...).
- `historico` (JSONB): Histórico limitado às **10 mensagens mais recentes**.
- `bot_paused_until` (TIMESTAMP): **Obrigatório.** Armazena a data/hora até quando o bot deve ignorar o lead (Pausa de 24h).
- `nicho` (VARCHAR): Armazena a persona ativa (estetica, saude, marketing).

## 🔑 Padrão de Credenciais e Conectividade
No **n8n v4**, para garantir a estabilidade das requisições HTTP ao WAHA e Claude:
1. **WAHA**: Use o tipo **Header Auth** (Generic) em vez de credenciais pré-definidas. Isso evita falhas de serialização e IDs inexistentes.
2. **Claude API**: O nó deve ser configurado como **Authentication: None**. As chaves (`x-api-key`) e a versão (`anthropic-version`) devem ser injetadas manualmente no campo **Headers** da requisição para evitar erros de autenticação do n8n.

## 📍 IDs e URLs Críticos (Estabilizados)
- **ID Workflow**: `6nJBQ4J8SkysTTbx`
- **Postgres (VPS)**: ID `5Uh0LTEOCQy2boJP` | Host: `postgres_saas` (Rede interna Docker).
- **WAHA (Vitrine)**: ID `k1y0l7T56J9x7BKb` | URL: `http://212.85.2.130:3001/api/`.
- **Nicho Padrão**: Se o nicho estiver vazio, o sistema força para `marketing` via lógica interna.

---
*Atualizado em: 2026-04-30 | Versão 9.1 | Responsável: Antigravity IA*
