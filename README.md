# 🦎 Agente Camaleão — Manual de Prevenção de Erros

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
| VPS IP | `YOUR_VPS_IP` |
| WAHA Vitrine | `http://YOUR_VPS_IP:3001` |
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

## 🔴 REGRA #4 — O "Escape das Aspas" no JSON do WAHA

### O Problema
Ao construir o body JSON para o WAHA manualmente em uma string, qualquer resposta da IA que contenha aspas, markdown ou caracteres especiais quebrará o JSON.

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
     📤 Enviar Resposta WAHA
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
| **3+** | `OBJECAO / DUVIDA` | Rotação de 3 CTAs para transbordo técnico. |

### 2. Rotação de CTA (Anti-Loop)
Para evitar que o bot se torne repetitivo na fase final, ele foi instruído a **nunca repetir o mesmo convite** do turno anterior. Ele alterna entre:
- Convite para falar com o **consultor**.
- Convite para falar com o **especialista**.
- Convite para alinhar com o **time técnico**.

### 3. Censura de Tags Internas
O marcador `[ENCERRADO]` e as diretivas de fase são filtrados no nó `🛑 Filtrar Encerrado` antes de chegarem ao WAHA. O lead recebe apenas o texto limpo.

---

## 🗣️ Protocolos de Resposta — Alex 7.5

> **Estas regras são parte do Core de Inteligência e devem estar sempre sincronizadas com o prompt no nó `📡 Chamar Claude MKT`.**

---

*Última atualização: 2026-04-25 | Alex 7.5 em produção | Histórico oficial consolidado*
