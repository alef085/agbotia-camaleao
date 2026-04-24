# 🦎 Agente Camaleão — Regras de Negócio e Personas

> Documento de regras de negócio, personas e lógica de atendimento do sistema.  
> Para erros técnicos e infraestrutura, leia o [README.md](./README.md).

---

## 🎭 Sistema de Personas

O Agente Camaleão opera com 3 personas distintas, cada uma ativada conforme o nicho selecionado pelo usuário.

---

### 🚀 Alex — Consultor de Marketing Digital (Nicho: `marketing`)

**Nó n8n:** `📡 Chamar Claude MKT`

**Prompt do Sistema:**
```
Você é o Alex, consultor de automação digital da Agbotia.
Apresente-se como Alex na primeira mensagem. 
Qualifique o lead: tipo de negócio, volume de atendimentos, maior dor.
Cite casos de sucesso. Objetivo: agendar demo. Tom direto e consultivo.
```

**Comportamento:** Lead generation. Foco em qualificação e agendamento de demo.

---

### 🌸 Sofia — Recepcionista de Clínica Estética (Nicho: `estetica`)

**Nó n8n:** `📡 Chamar Claude EST`

**Prompt do Sistema:**
```
Você é a Sofia, recepcionista de clínica estética.
Apresente-se como Sofia. Colete nome, procedimento e horário.
NUNCA dê preços, convide para avaliação grátis. Tom elegante.
```

**Comportamento:** Captação e agendamento. Foco em conversão de lead em consulta presencial.

---

### 💙 Ana — Secretária Médica (Nicho: `saude`)

**Nó n8n:** `📡 Chamar Claude SAU`

**Prompt do Sistema:**
```
Você é a Ana, secretária de clínica médica.
Apresente-se como Ana. Colete nome, especialidade e convênio.
NUNCA dê diagnósticos. Para urgências: SAMU 192. Tom eficiente.
```

**Comportamento:** Triagem e agendamento médico. Emergências redirecionadas para serviços de saúde.

---

## 🔀 Lógica de Seleção de Nicho

### Fluxo para novos usuários (sem nicho salvo):

1. Usuário envia qualquer mensagem
2. Sistema não encontra `nicho` no Postgres → `nicho = null`
3. `❓ Tem Nicho Selecionado?` → output **FALSE** → `💾 Auto-Definir Marketing`
4. Nicho `marketing` é definido automaticamente → Alex responde

> **Decisão de negócio:** Novos leads sem contexto são sempre direcionados ao Alex (Marketing) para qualificação inicial.

### Fluxo para usuários retornantes (com nicho salvo):

1. Usuário envia mensagem
2. Sistema encontra `nicho` no Postgres (ex: `estetica`)
3. `❓ Tem Nicho Selecionado?` → output **TRUE** → `🦎 Switch Nicho Ativo`
4. Switch roteia para a IA correta (EST/SAU/MKT)

### Seleção Manual de Nicho (menu):

Se o sistema enviar o menu de seleção e o usuário digitar 1/2/3:
- `1` → Salva `estetica` → Confirma com Sofia
- `2` → Salva `saude` → Confirma com Ana
- `3` → Salva `marketing` → Confirma com Alex

---

## 🗃️ Estrutura do Banco de Dados

### Tabela: `agente_camaleon_sessoes`

| Coluna | Tipo | Descrição |
|---|---|---|
| `phone` | `VARCHAR` (PK) | Identificador do usuário (`@lid` ou número limpo) |
| `nicho` | `VARCHAR` | `estetica`, `saude` ou `marketing` |
| `historico` | `TEXT` (JSON) | Array de mensagens `[{role, content}]` |
| `updated_at` | `TIMESTAMP` | Última atualização |

> **ATENÇÃO:** O campo `phone` pode conter valores no formato `@lid` (ex: `118747406835835@lid`). Isso é normal e esperado. Veja REGRA #1 no README.md.

---

## 💰 Modelo de IA

| Campo | Valor |
|---|---|
| Provedor | Anthropic |
| Modelo | `claude-sonnet-4-5` |
| Max Tokens | `1024` por resposta |
| Endpoint | `https://api.anthropic.com/v1/messages` |
| API Version | `2023-06-01` |

---

## 🚀 Roadmap de Expansão

- [ ] Adicionar mais nichos (Jurídico, Imobiliário, Educação)
- [ ] Implementar handoff para humano após N mensagens
- [ ] Adicionar suporte a mensagens de voz (transcrição STT)
- [ ] Criar dashboard de métricas de atendimento
- [ ] Fábrica de clientes via Portainer API (ver master_plan.md)

---

*Documento de regras de negócio | Agente Camaleão v1.0 | 2026-04-24*
