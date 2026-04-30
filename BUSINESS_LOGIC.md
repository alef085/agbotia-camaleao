# 🧠 Lógica de Negócio e Estratégia — Alex 9.1

O Alex 9.1 é um ecossistema de atendimento híbrido projetado para máxima eficiência de custo e conversão, utilizando uma arquitetura de múltiplos modelos e blindagem anti-loop.

## 🏗️ Arquitetura Híbrida (Custo-Eficiência)
- **Fase de Triagem (SAU)**: Processada pelo **Claude 3.5 Haiku**. Foco em velocidade e baixo custo para saudações e identificação inicial.
- **Fase Estratégica (MKT/EST)**: Processada pelo **Claude 3.5 Sonnet**. Foco em persuasão, quebra de objeções e fechamento de alto nível.

---

## 🏗️ Máquina de Estados (A Régua de Consciência)

| Fase | Gatilho (Msgs Assistant) | Objetivo |
|---|---|---|
| **Apresentação** | 0 | Contextualizar o foco e pedir permissão. |
| **Doutrinação** | 1 - 2 | Conscientização sobre a dor/solução. |
| **Fechamento** | 3 | Oferta direta de projeto ou agendamento. |
| **Coleta** | Após Aceite | Coleta de dados (Nome, E-mail, Empresa). |
| **Entrega / PDF** | Dados Recebidos | Finalização e entrega de valor. |
| **Objeção / Dúvida** | 3+ | Responder dúvidas e oferecer transbordo técnico. |

---

## 🛡️ Protocolo de Blindagem 9.1 (Anti-Loop)
Para evitar "conversas infinitas" entre robôs ou spam, o sistema possui 3 travas automáticas:

1. **Filtro de Não-Humanos**: O nó de entrada ignora mensagens que contenham termos de sistema (ex: "cod. do pedido", "assistente virtual", "comprovante").
2. **Detector de Mudez (Emojis)**: Se a mensagem contiver apenas emojis, o fluxo é encerrado imediatamente ("Vácuo Estratégico"), sem resposta do bot.
3. **Trava de Rajada (Rate Limit)**: Se um número enviar 5 ou mais mensagens em menos de 120 segundos, o bot se silencia automaticamente para aquele contato.

---

---

## ⏸️ Lógica de Pausa Automática (24 Horas)
Para garantir que o bot não interrompa um atendimento humano após a qualificação do lead:
- **Gatilho**: Ao final da fase `FINALIZACAO` ou quando o lead é marcado como `[ENCERRADO]`.
- **Ação**: O nó `⏸️ Pausar Bot 24h` grava no banco (`bot_paused_until`) a data/hora atual + 24 horas.
- **Bypass de Teste**: Durante fases de estabilização, a lógica de pausa pode estar comentada no nó de código ou o nó de escrita desativado, permitindo testes repetidos com o mesmo número.

## 🦎 Triagem e Personas (Efeito Camaleão)
O bot identifica e assume personas diferentes baseando-se na escolha do usuário ou histórico:
1. **Triagem Inicial**: O menu de nichos permite que o usuário selecione entre Estética, Saúde ou Marketing.
2. **Auto-Nicho**: Se um lead entra no fluxo sem nicho definido, o sistema possui uma regra de fallback que o direciona para a persona de **Marketing (Alex)** para garantir que nenhum lead fique sem resposta.
3. **Persistência**: Uma vez definido o nicho, ele é travado no banco de dados para que o atendimento mantenha a mesma voz até o encerramento da sessão.

---

## 📝 Filtro e Formatação de Histórico
O nó `📝 Formatar Histórico` é o cérebro da sessão e possui proteções críticas:
- **Trava de Pausa**: Antes de processar, ele verifica se `bot_paused_until` é maior que a hora atual. Se sim, o fluxo é abortado para respeitar o silêncio do bot.
- **Filtro de Ruído**: Mensagens que contenham `[object Object]`, `undefined`, `null` ou erros de sistema são removidas do histórico enviado à IA para evitar alucinações.
- **Diretiva de Fase**: O nó injeta dinamicamente a `stageDirective`, forçando a IA a seguir o roteiro da fase atual do funil.

---

---

## 🗄️ Estrutura do Banco de Dados
Tabela: `agente_camaleon_sessoes`
- `phone`: ID do usuário (PK).
- `historico`: Array JSONB de mensagens (Limitado às **10 mais recentes** para economia de tokens).
- `bot_paused_until`: Timestamp que define até quando o bot deve ignorar novas mensagens (Silenciamento Pós-Qualificação).
- `nicho`: Identificador da persona ativa (estetica, saude, marketing).
- `updated_at`: Controle de última interação.
