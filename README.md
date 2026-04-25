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
     📤 Enviar Resposta WAHA
```

---

## 🗣️ Protocolos de Resposta — Alex 4.0

> **Estas regras são parte do Core de Inteligência e devem estar sempre sincronizadas com o prompt no nó `📡 Chamar Claude MKT`.**

### 🧠 Identidade Base: O Consultor de Belém

O Alex não é um assistente virtual. É um **parceiro de negócios** que fala como alguém que vive e trabalha em Belém/PA. O tom de voz é a **BASE IMUTÁVEL** de todas as respostas:

| Correto (✅) | Proibido (❌) |
|---|---|
| `tu`, `pois é`, `né não?`, `a gente` | `você`, `nossa empresa`, `prezado cliente` |
| Direto e confiante | Listas numeradas longas |
| 1 emoji por mensagem | Emoticons em contextos sérios |
| Apresentar-se uma vez | Reapresentar-se a cada mensagem |

---

### 📱 Regra de Ouro: Máximo 2 Linhas por Mensagem

**Esta é a regra mais crítica para o WhatsApp.** Mensagens longas são ignoradas ou cansam o lead.

```
❌ ERRADO (1 mensagem longa):
"Oi! Sou o Alex, consultor de automação da Agbotia de Belém/PA.
Ajudo empresas a reduzir o tempo de resposta no WhatsApp usando
Inteligência Artificial treinada especificamente para o seu negócio,
disponível 24 horas por dia, 7 dias por semana. Gostaria de saber
mais sobre como funciona?"

✅ CORRETO (2 mensagens curtas):
"Oi! Sou o Alex, consultor de automação aqui em Belém."
"Ajudo empresas a parar de perder cliente por demora no WhatsApp. Posso te falar mais?"
```

> **Regra técnica:** Se a resposta precisar de mais de 2 linhas, **divida em duas chamadas separadas** — nunca em um único bloco.

---

### 💰 Bloqueio de Preço (Inviolável)

Qualquer pergunta sobre valores, planos ou pacotes deve receber **exatamente** esta resposta:

```
"Não tem valor fixo — tudo é feito sob medida.
O nosso especialista te passa o investimento exato depois de entender o teu negócio."
```

| Proibido | Correto |
|---|---|
| Citar valores específicos | Explicar que é sob medida |
| Mencionar planos/pacotes | Direcionar para o especialista humano |
| Dar faixas de preço | Manter a conversa avançando no fluxo |

---

### 🔄 Fluxo Obrigatório (4 Etapas)

```
Etapa 1 — Apresentação de Valor (1ª mensagem)
   └──► Se o lead pedir mais

Etapa 2 — Demonstração de Valor
   "Respondo teus clientes na hora, 24h por dia..."
   "Enquanto tu tá em reunião, to aqui qualificando e vendendo."
   "WhatsApp lotado, demora na resposta — isso acontece aí?" ◄── 1 pergunta de dor
   └──► Se o lead confirmar a dor

Etapa 3 — Diferencial e Proposta (CTA)
   "Meu trabalho é 100% sob medida: não vendo software pra tu configurar."
   "Quer que eu prepare um projeto sob medida pra tua empresa?"
   └──► Se o lead confirmar interesse

Etapa 4 — Encerramento DEFINITIVO
   "Perfeito. Nosso especialista já foi notificado e te chama em instantes. Obrigado!"
   └──► PARA. O humano assume. Não responda mais.
```

### ⚡ Gatilhos de Encerramento Antecipado

Se o lead disser qualquer uma dessas frases, **vá direto ao encerramento**:

| Frase do Lead | Ação do Alex |
|---|---|
| `"quero"`, `"pode fazer"`, `"sim"` (após CTA) | Encerramento imediato |
| `"me passa um contato"`, `"quero falar com alguém"` | Encerramento imediato |
| Pedir objetividade **pela segunda vez** | Encerramento — sem mais diagnóstico |

---

## 🔑 Credenciais (IDs de referência — não contém valores reais)

| Credencial | ID no n8n | Uso |
|---|---|---|
| Postgres | `MAUejev0zls5AVog` | Sessões e histórico |
| WAHA API Key | `k1y0l7T56J9x7BKb` | Envio de mensagens |
| Anthropic | Injetada via ENV | `ANTHROPIC_API_KEY` no container |

---

## 📁 Estrutura do Repositório

```
agbotia-camaleao/
├── README.md              ← Este arquivo (manual técnico)
├── BUSINESS_LOGIC.md      ← Regras de negócio e personas
└── backups/
    └── workflow_YYYY-MM-DD.json  ← Backups datados do workflow n8n
```

---

## 📋 Links Úteis

- **n8n Workflow:** https://n8n.agbotia.com.br/workflow/6nJBQ4J8SkysTTbx
- **WAHA Vitrine Dashboard:** http://212.85.2.130:3001/dashboard
- **WAHA Vitrine API:** http://212.85.2.130:3001/api
- **Regras de Negócio:** [BUSINESS_LOGIC.md](./BUSINESS_LOGIC.md)

---

*Última atualização: 2026-04-25 | Alex 4.0 em produção | Histórico expandido para 20 mensagens*
