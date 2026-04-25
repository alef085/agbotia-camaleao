# 🛡️ Backup de Segurança — Alex 7.5 (Oficial)

Este documento contém a "fonte da verdade" dos códigos críticos. Use estes trechos para restauração manual.

---

## 1. 🛑 Nó: Filtrar Encerrado (Censura de Interface)
**Função**: Remove tags técnicas e detecta o fim da conversa.

```javascript
// 🛑 VERIFICAR ENCERRAMENTO E LIMPAR INTERFACE
const aiResponse = ($input.first().json.content?.[0]?.text || '').trim();
const phone = $('📋 Extrair Dados WAHA').first().json.phone;

const isEncerrado = aiResponse.includes('[ENCERRADO]');

// Limpa qualquer vazamento técnico
let textoLimpo = aiResponse
  .replace(/\[ENCERRADO\]/g, '')
  .replace(/\[DIRETIVA DO SISTEMA\]/g, '')
  .trim();

// Remove rastros de fase se houver
textoLimpo = textoLimpo.replace(/VOCE ESTA NA FASE:.*?\\n/g, '').trim();

return [{ json: {
  ...$input.first().json,
  isEncerrado: isEncerrado,
  textoLimpo: textoLimpo,
  phone: phone
} }];
```

---

## 2. 📝 Nó: Formatar Histórico (Máquina de Estados 7.5)
**Função**: Define a fase baseada na contagem 0, 1-2, 3, 3+.

```javascript
const extracted = $('📋 Extrair Dados WAHA').first()?.json || {};
const sessionRow = $('🗃️ Buscar Sessão Usuário').first()?.json || null;
const phone = extracted.phone;
const message = extracted.message || '';

let historico = [];
if (sessionRow?.historico) {
  if (typeof sessionRow.historico === 'string') { try { historico = JSON.parse(sessionRow.historico); } catch(e) {} }
  else if (Array.isArray(sessionRow.historico)) { historico = sessionRow.historico; }
}

const historico_limpo = historico.filter(h => h.role === 'user' || h.role === 'assistant');
const assistantCount = historico_limpo.filter(h => h.role === 'assistant').length;

let stage = 'APRESENTACAO';

if (assistantCount === 0) {
    stage = 'APRESENTACAO';
} else if (assistantCount >= 1 && assistantCount <= 2) {
    stage = 'DOUTRINACAO';
} else if (assistantCount === 3) {
    stage = 'FECHAMENTO';
} else {
    stage = 'OBJECAO';
}

const stageDirective = '[DIRETIVA DO SISTEMA]\\nVOCE ESTA NA FASE: ' + stage;

return [{ json: { phone, message, mensagens_formatadas: historico_limpo, stageDirective, current_stage: stage } }];
```

---

## 3. 📤 Nó: Enviar Resposta WAHA (Configuração)
**URL**: `http://YOUR_VPS_IP:3001/api/sendText`
**Body (Text)**: 
`={{ $('🛑 Filtrar Encerrado').first().json.textoLimpo || 'Erro na resposta' }}`

---

## 📅 Status
- **Versão**: 7.5
- **Data**: 25/04/2026
- **Status**: Produção Estabilizada
