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

### 🧠 Identidade Base: O Consultor de Belém

O Alex não é um assistente virtual. É um **parceiro de negócios** que fala como alguém que vive e trabalha em Belém/PA. O tom de voz é a BASE IMUТÁVEL de todas as respostas:

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
| Dar faixas de preço | Manter a conversa avancando no fluxo |

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

## 🔒 Segurança e Anti-Loop (Alex 9.1)

O sistema agora conta com um triplo bloqueio de execução para evitar loops infinitos com outros bots e spam:

1.  **Filtro de Sistema (Entrada):** O nó de entrada (`🚡 Filtrar Evento`) descarta mensagens com termos como `cod. do pedido`, `comprovante`, `assistente virtual`, etc.
2.  **Detector de Mudez (Emojis):** Mensagens compostas exclusivamente por emojis não geram resposta da IA, aplicando o "Vácuo Estratégico".
3.  **Trava de Rajada (Rate Limit):** O sistema monitora a frequência de mensagens. Se o mesmo contato enviar 5 mensagens em menos de 120 segundos, o bot é silenciado para aquele número automaticamente.
4.  **Pausa Automática Pós-Fechamento (24h):** Após enviar a tag `[ENCERRADO]`, o bot atualiza a coluna `bot_paused_until` no banco com a data/hora atual + 24h, bloqueando respostas da IA para permitir intervenção humana segura.

---

## 🗃️ Schema do Banco de Dados (Postgres)

A tabela principal do sistema (`agente_camaleon_sessoes`) requer a seguinte estrutura mínima para suportar as travas do Alex 9.1:

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

Para otimizar custos em até 60% sem perder a qualidade do fechamento:
- **Triagem (Nicho SAU):** Claude 3.5 Haiku (Rápido e Barato).
- **Venda (Nicho MKT/EST):** Claude 3.5 Sonnet (Inteligente e Persuasivo).

---

*Última atualização: 2026-04-30 | Alex 9.1 em produção | Histórico limitado a 10 mensagens*