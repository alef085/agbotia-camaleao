# 🧠 Lógica de Negócio e Estratégia — Alex 9.1

O Alex 9.1 é um ecossistema de atendimento híbrido projetado para máxima eficiência de custo e conversão, utilizando uma arquitetura de múltiplos modelos e blindagem anti-loop.

---

## 🏗️ Arquitetura Híbrida (Custo-Eficiência)
- **Fase de Triagem (SAU)**: Processada pelo **Claude 3.5 Haiku**. Foco em velocidade e baixo custo para saudações e identificação inicial.
- **Fase Estratégica (MKT/EST)**: Processada pelo **Claude 3.5 Sonnet**. Foco em persuasão, quebra de objeções e fechamento de alto nível.

---

## 🏗️ Máquina de Estados (Roteiro do Agente)

| Fase | Gatilho (Msgs Assistant) | Objetivo |
|---|---|---|
| **Apresentação** | 0 | Contextualizar o foco e pedir permissão. |
| **Doutrinação** | 1 - 2 | Conscientização sobre a dor/solução. |
| **Fechamento** | 3 | Oferta direta de projeto ou agendamento. |
| **Objeção / Dúvida**| 3+ | Responder dúvidas e oferecer transbordo técnico. |
| **Finalização** | Intenção de Compra / SIM | Encerramento com tag **[ENCERRADO]**. |

---

## 🛡️ Protocolo de Blindagem 9.1 (Anti-Loop)
Para evitar "conversas infinitas" entre robôs ou spam, o sistema possui 3 travas automáticas:

1. **Filtro de Não-Humanos**: O nó de entrada ignora mensagens que contenham termos de sistema (ex: "cod. do pedido", "assistente virtual", "comprovante").
2. **Detector de Mudez (Emojis)**: Se a mensagem contiver apenas emojis, o fluxo é encerrado imediatamente ("Vácuo Estratégico"), sem resposta do bot.
3. **Trava de Rajada (Rate Limit)**: Se um número enviar 5 ou mais mensagens em menos de 120 segundos, o bot se silencia automaticamente para aquele contato.

---

## ⏸️ Lógica de Handover e Pausa (24 Horas)
Para garantir que o bot não interrompa um atendimento humano após a qualificação do lead:
- **Gatilho**: Ao detectar a tag **`[ENCERRADO]`** na resposta da IA.
- **Ação Técnica**: 
    1. O nó `🛑 Filtrar Encerrado` remove a tag do texto visível ao lead.
    2. O nó `⏸️ Pausar Bot 24h` grava no banco (`bot_paused_until`) a data/hora atual + 24 horas.
    3. O nó `🔔 Alertar Dono` envia uma notificação ao consultor humano.
- **Objetivo**: Garantir silêncio absoluto do robô enquanto o humano assume a negociação final.

---

## 🦎 Triagem e Personas (Efeito Camaleão)
O bot identifica e assume personas differentes baseando-se na escolha do usuário ou histórico:
1. **Triagem Inicial**: O menu de nichos permite que o usuário selecione entre Estética, Saúde ou Marketing.
2. **Auto-Nicho**: Se um lead entra no fluxo sem nicho definido, o sistema o direciona para a persona de **Marketing (Alex)**.
3. **Persistência**: Uma vez definido o nicho, ele é travado no banco de dados para que o atendimento mantenha a mesma voz até o encerramento.

---

## 📝 Filtro e Formatação de Histórico
O nó `📝 Formatar Histórico` é o guardião da sessão:
- **Trava de Pausa**: Verifica se `bot_paused_until > NOW()`. Se positivo, aborta a execução.
- **Limpeza de Dados**: Remove mensagens vazias ou erros de sistema do contexto enviado à IA.
- **Injeção de Fase**: Calcula o estágio do funil e injeta a diretiva de comportamento correspondente.

---

## 🗄️ Estrutura do Banco de Dados (Postgres)
Tabela: `agente_camaleon_sessoes`
- `phone`: ID do usuário (PK).
- `historico`: Array JSONB de mensagens (Limitado às **10 mais recentes**).
- `bot_paused_until`: Timestamp de silenciamento.
- `nicho`: Identificador da persona (estetica, saude, marketing).
- `updated_at`: Última interação.

---

*Última atualização: 2026-05-06 | Lógica de Handover 9.1*
