# 🦎 Agente Camaleão 9.1 — Manual de Prevenção de Erros

> **⚠️ PROTOCOLO DE ENTRADA OBRIGATÓRIO:** Este é o PRIMEIRO documento que qualquer agente, desenvolvedor ou IA deve ler ao assumir este projeto. A violação das regras aqui documentadas causa falhas silenciosas de difícil diagnóstico.

---

## 🏗️ Visão Geral da Infraestrutura

```
[WhatsApp User]
      │
      ▼
[WAHA Vitrine — porta 3001] ← engine WEBJS (whatsapp-web.js)
      │  webhook POST
      ▼
[n8n — n8n.agbotia.com.br]
  ├── [Postgres] → agente_camaleon_sessoes
  └── [Anthropic API] → Claude Sonnet 4.5
      │
      ▼
[WAHA Vitrine — sendText]
      │
      ▼
[WhatsApp User] ← resposta do agente
```

| Componente | Valor |
|---|---|
| VPS IP | `212.85.2.130` |
| WAHA Vitrine | `http://212.85.2.130:3001` |
| n8n | `https://n8n.agbotia.com.br` |
| Workflow ID | `6nJBQ4J8SkysTTbx` |
| Sessão WAHA | `default` |
| Engine WAHA | `WEBJS` (whatsapp-web.js) |

---

## 🔴 REGRA #1 — O Caso do @lid (CRÍTICO)

### O Problema
O WAHA com engine `WEBJS` usa dois sistemas de identificação de contatos:

| Formato | Exemplo | Quando aparece |
|---|---|---|
| `@c.us` | `5511999887766@c.us` | Contatos salvos com número brasileiro |
| `@lid` | `118747406835835@lid` | Contatos via Linked Devices (IDs internos do WhatsApp) |

**O campo `payload.from` pode retornar qualquer um dos dois formatos.**

### O Bug Que Custou Horas de Debugging
```json
// payload.from recebido pelo webhook:
"from": "118747406835835@lid"

// Código ERRADO — remove @c.us mas não remove @lid:
const phone = from.replace('@c.us', ''); // resultado: "118747406835835@lid"

// chatId enviado ao WAHA (INVÁLIDO):
"chatId": "118747406835835@lid@c.us"  ← WAHA aceita silenciosamente, mas NÃO ENVIA
```

> **Sintoma traiçoeiro:** O WAHA retorna HTTP 200 com body `{}`. Nenhum erro aparece no n8n. A mensagem simplesmente não chega.

### A Solução Correta
```javascript
// No nó 📋 Extrair Dados WAHA:
const fromRaw = payload.from || '';
let phone;
if (fromRaw.includes('@lid')) {
  // Preservar @lid COMPLETO — o WAHA WEBJS aceita @lid para envio direto
  phone = fromRaw; // ex: "118747406835835@lid"
} else {
  phone = fromRaw.replace('@c.us', '').replace('@s.whatsapp.net', '');
}
```

```javascript
// No nó 📤 Enviar Resposta WAHA (chatId inteligente):
// Se phone já tem '@', usa como está. Senão, adiciona @c.us.
{{ $('📋 Extrair Dados WAHA').first().json.phone.includes('@')
  ? $('📋 Extrair Dados WAHA').first().json.phone
  : $('📋 Extrair Dados WAHA').first().json.phone + '@c.us' }}
```

### ⚠️ Regra de Ouro
> **NUNCA force `@c.us` em identificadores que já contenham `@lid`.** O WAHA retornará `{}` sem erro algum, mas a mensagem não será entregue.

---

## 🔴 REGRA #2 — Restrições de Sandbox (Code Nodes)

### O Problema
O n8n roda Code Nodes em um sandbox JavaScript restrito que **bloqueia `$helpers.httpRequest`** para requisições HTTP externas.

```javascript
// ❌ PROIBIDO — vai falhar silenciosamente no sandbox:
const response = await $helpers.httpRequest({
  method: 'POST',
  url: 'https://api.anthropic.com/v1/messages',
});
```

### A Solução Correta
Usar exclusivamente **nós nativos de HTTP Request** do n8n para qualquer chamada externa:

```
✅ Correto: nó "n8n-nodes-base.httpRequest" com configuração visual
❌ Proibido: Code Node com $helpers.httpRequest para APIs externas
```

### Quando Code Nodes são Permitidos
Code Nodes são **apenas** para:
- Filtrar/transformar dados (sem HTTP)
- Lógica condicional complexa
- Formatação de strings/arrays
- Extração de campos do payload

---

## 🔴 REGRA #3 — Filtros de Webhook Triplo

### O Problema
O WAHA WEBJS envia **múltiplos tipos de eventos** para o webhook, não apenas mensagens de texto. Sem filtragem adequada, o fluxo processa eventos vazios e para silenciosamente.

### Tipos de eventos problemáticos observados:

| Tipo | Sintoma | Causa |
|---|---|---|
| `e2e_notification` | `body.payload.body = ""` | Notificação interna de criptografia end-to-end |
| `status` | `body.event != 'message'` | Atualização de status do contato |
| `fromMe = true` | Mensagem do próprio bot | Loop infinito de auto-resposta |

### A Lógica Tripla Obrigatória
```javascript
// No nó 🚡 Filtrar Evento:
const body = $input.first().json.body;

const isRealMessage = (
  body.event === 'message' &&              // ✅ É um evento de mensagem
  !body.payload.fromMe &&                   // ✅ Não é mensagem do próprio bot
  body.payload.body &&                      // ✅ Tem conteúdo de texto
  body.payload.body.trim() !== '' &&        // ✅ Texto não é vazio
  body.payload._data?.type !== 'e2e_notification' // ✅ Não é notificação interna
);

if (isRealMessage) {
  return $input.all();
}
return []; // Para o fluxo silenciosamente
```

---

## 🔴 REGRA #4 — Configuração de Envio (Key-Value Pairs)

### O Problema
O campo `specifyBody: "json"` com `jsonBody` como string causa erros de escape de caracteres quando a resposta da IA contém aspas, markdown ou caracteres especiais.

```javascript
// ❌ PROIBIDO — causa erro "Session name is required" ou 500 no WAHA:
"jsonBody": "={\n  \"session\": \"{{ $json.session }}\",\n  \"text\": \"{{ $json.aiResponse }}\"\n}"
// Problema: aspas da IA quebram o JSON
```

### A Solução Correta
Usar `specifyBody: "keypair"` com campos separados:

```json
// ✅ Correto — configuração do nó HTTP Request:
{
  "specifyBody": "keypair",
  "bodyParameters": {
    "parameters": [
      { "name": "session", "value": "default" },
      { "name": "chatId", "value": "={{ ... }}" },
      { "name": "text", "value": "={{ ... }}" }
    ]
  }
}
```

> **Por quê funciona:** Cada campo é serializado independentemente pelo n8n, eliminando problemas de escape de caracteres especiais da resposta da IA.

---

## 🟡 REGRA #5 — Captura Universal da Resposta da IA

Como apenas UMA das 3 IAs executa por vez (Switch de nicho), a expressão de captura deve usar operador `||` em cascata:

```javascript
// ✅ Expressão correta para o campo 'text':
{{ 
  $('📡 Chamar Claude MKT').first()?.json?.content?.[0]?.text || 
  $('📡 Chamar Claude EST').first()?.json?.content?.[0]?.text || 
  $('📡 Chamar Claude SAU').first()?.json?.content?.[0]?.text || 
  'Erro na resposta' 
}}
```

> Usar `?.` (optional chaining) é obrigatório para evitar erros quando os nós não executaram.

---

## 📊 Fluxo do Workflow (Mapa Mental)

```
Webhook → 🚡 Filtrar Evento → 📋 Extrair Dados WAHA
                                        │
                             🗃️ Buscar Sessão Usuário (Postgres)
                                        │
                             🔀 Merge Dados + Sessão
                                        │
                             📝 Formatar Histórico
                                        │
                        ┌── ❓ Tem Nicho? ──┐
                        │ SIM              │ NÃO
                        ▼                  ▼
             🦎 Switch Nicho      💾 Auto-Definir Marketing
            /     |      \                │
     EST     SAU    MKT    ───────────────►│
      │       │      │
      └───────┴──────┘
             │
      📡 Chamar Claude [EST/SAU/MKT]
             │
      💾 Salvar Histórico (Postgres)
             │
      🛑 Filtrar Encerrado (Code)
             │
      ┌── ❓ Lead Encerrado? ──┐
      │ SIM (Tag [ENCERRADO]) │ NÃO
      ▼                       ▼
⏸️ Pausar Bot 24h (Postgres)  📤 Enviar Resposta WAHA
      │
🔔 Alertar Dono (WAHA)
```

---

## 🦎 Arquitetura Alex 7.5 — Máquina de Estados Rígida

O fluxo do Alex não é mais baseado em "decisão" da IA, mas sim em um contador de mensagens do `assistant` que injeta diretivas imutáveis no System Prompt.

### 1. Contagem de Mensagens (Assistant Count)
| Mensagens enviadas | Estágio | Objetivo |
|---|---|---|
| **0** | `APRESENTACAO` | Valor imediato e permissão. |
| **1 - 2** | `DOUTRINACAO` | Conscientização sobre perda de vendas. |
| **3** | `FECHAMENTO` | Oferta de projeto sob medida. |
| **3+** | `OBJECAO / DUVIDA` | Rotação de CTAs ou Transição Humana. |
| **Aceite** | `FINALIZACAO` | Encerramento com tag **[ENCERRADO]**. |

### 2. Rotação de CTA (Anti-Loop)
Para evitar que o bot se torne repetitivo na fase final, ele alterna entre:
- Convite para falar com o **consultor**.
- Convite para falar com o **especialista**.
- Convite para alinhar com o **time técnico**.

### 3. Handover Humano Imediato (Alex 9.1)
Ao detectar intenção de compra ou o lead dizer "SIM", o bot deve:
1.  Gerar a tag **`[ENCERRADO]`** ao final da mensagem.
2.  A tag é filtrada pelo nó `🛑 Filtrar Encerrado`, garantindo que o lead não veja o comando técnico.
3.  O workflow identifica a flag `isEncerrado` e silencia o robô por **24 horas** via banco de dados.
4.  O proprietário recebe um alerta imediato com o número do lead qualificado.

---

## 🗣️ Protocolos de Resposta — Alex 7.5

### 🧠 Identidade Base: O Consultor de Automação

O Alex é um **parceiro de negócios** que fala de forma direta e profissional.

| Correto (✅) | Proibido (❌) |
|---|---|
| `tu`, `pois é`, `né não?` | `você`, `nossa empresa`, `prezado cliente` |
| Máximo 2 parágrafos | Blocos densos de texto |
| Espaço duplo antes da pergunta | Perguntas coladas no texto |
| 1 emoji por mensagem | Excesso de emojis |

---

## ⚡ Trava de Pausa Pós-Fechamento (24h)

O sistema conta com um bloqueio de execução para evitar loops pós-qualificação:

1.  **Gatilho:** Presença da tag `[ENCERRADO]` na resposta da IA.
2.  **Ação:** Atualização da coluna `bot_paused_until` no banco com `NOW() + INTERVAL '24 hours'`.
3.  **Resultado:** O nó `📝 Formatar Histórico` aborta a execução se detectar que o tempo de pausa ainda não expirou.
4.  **Notificação:** O nó `🔔 Alertar Dono` envia os dados do lead para atendimento manual.

---

## 🗃️ Schema do Banco de Dados (Postgres)

```sql
CREATE TABLE IF NOT EXISTS agente_camaleon_sessoes (
    phone VARCHAR(20) PRIMARY KEY,
    nicho VARCHAR(50),
    historico JSONB,
    updated_at TIMESTAMP,
    bot_paused_until TIMESTAMP -- Controle de pausa por 24h (Alex 9.1)
);
```

---

## ⚡ Arquitetura Híbrida de Modelos

- **Triagem (Nicho SAU):** Claude 3.5 Haiku (Rápido e Barato).
- **Venda (Nicho MKT/EST):** Claude 3.5 Sonnet (Inteligente e Persuasivo).

---

*Última atualização: 2026-05-06 | Alex 9.1 Estabilizado | Lógica de Handover Ativa*
