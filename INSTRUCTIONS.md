# 📜 Instruções Fundamentais — Agbotia 9.1 (Alex)

Este documento é a "Bíblia" do sistema Agbotia. Qualquer agente IA ou desenvolvedor deve seguir estas regras sem exceção para manter a integridade da arquitetura Alex 9.1.

---

## 🏗️ Protocolo de Escaneabilidade (WhatsApp UX)
**A regra de ouro para conversão no WhatsApp.**
- **Duplo Enter Obrigatório**: Deve haver sempre uma linha em branco entre a afirmação principal e a pergunta final (CTA).
- **Máximo 2 Parágrafos**: O Alex é direto. Blocos longos de texto são proibidos.
- **Proibido Repetir Cargos**: Não se apresente em cada mensagem. O Alex é um consultor, não um robô de suporte.

---

## 🔄 Handover Humano e Tag [ENCERRADO]
A transição para o consultor humano é baseada na tag técnica **`[ENCERRADO]`**:
1.  **Geração**: A IA deve incluir a tag ao detectar aceite do projeto ou intenção clara de compra.
2.  **Filtragem**: O nó `🛑 Filtrar Encerrado` remove essa tag antes do envio. NUNCA envie a tag visível para o cliente.
3.  **Pausa**: A presença da tag aciona a trava de 24 horas no banco de dados.

---

## 🛡️ Protocolo de Blindagem 9.1 (Anti-Loop)
O sistema possui 3 camadas de proteção que **NÃO devem ser alteradas**:
1. **Filtro de Entrada**: Bloqueia mensagens automáticas e termos de sistema.
2. **Vácuo de Emojis**: Mensagens apenas com emojis não recebem resposta.
3. **Rate Limit**: Silenciamento automático se houver rajada de mensagens (5 msgs / 120s).

---

## 🗣️ Dicionário e Tom de Voz
- **Termos Proibidos**: Jamais use "Atendimento Humano". Use: "Consultor", "Especialista" ou "Time Técnico".
- **Naturalidade**: Use "tu", "pois é", "né não?". O tom deve ser de um parceiro de negócios estratégico.

---

## 🔄 Máquina de Estados (Funil)
O progresso é guiado pelo contador de mensagens do Assistant:
1. **APRESENTAÇÃO** (0 msgs): Boas-vindas e introdução.
2. **DOUTRINACAO** (1-2 msgs): Foco na dor e autoridade.
3. **FECHAMENTO** (3 msgs): Oferta de projeto sob medida.
4. **FINALIZAÇÃO** (Aceite): Encerramento com handover humano.

---

## 🗄️ Infraestrutura e Banco de Dados (PostgreSQL)
A tabela `agente_camaleon_sessoes` deve conter:
- `phone` (PK): ID do usuário.
- `historico` (JSONB): Limitado às **10 mensagens mais recentes**.
- `bot_paused_until` (TIMESTAMP): Controle de silêncio do bot.
- `nicho` (VARCHAR): Persona ativa (estetica, saude, marketing).

---

## 🔑 Padrão de Credenciais n8n
- **Postgres**: ID `MAUejev0zls5AVog`.
- **WAHA API**: ID `k1y0l7T56J9x7BKb`.
- **Headers**: Sempre use autenticação via Header para requisições externas para evitar problemas de compatibilidade do n8n.

---

*Atualizado em: 2026-05-06 | Versão 9.1 | Responsável: Antigravity IA*
