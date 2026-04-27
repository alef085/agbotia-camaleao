# Regras de Negócio — Agente Camaleão 🦎

## 1. Regras de Ouro (Core Logic)

Estas regras são injetadas em todas as personas via System Prompt para garantir a qualidade do atendimento:

*   **Escuta Ativa Prioritária**: Se o usuário aceitar a oferta (ex: "quero agendar", "pode fazer"), a IA deve parar de "vender" e iniciar imediatamente a coleta de dados.
*   **Anatomia Rígida da Mensagem**:
    *   Máximo de 2 parágrafos por resposta.
    *   Espaço duplo (Enter duplo) obrigatório antes de qualquer pergunta final (CTA).
    *   Proibido blocos densos de texto.
*   **Censura de Tags**: A IA nunca deve mostrar colchetes ou diretivas técnicas (ex: `[FASE: COLETA]`) para o cliente.
*   **Gestão de Preços**: Proibido passar preços fixos. O script padrão é: *"O valor depende da avaliação/projeto — que é o nosso próximo passo."*

## 2. Funil de Conversão (Fases)

O bot segue um ciclo de estados baseado no histórico:

1.  **APRESENTAÇÃO**: Boas-vindas e introdução do benefício.
2.  **DOUTRINAÇÃO**: Geração de autoridade e consciência do problema.
3.  **FECHAMENTO**: Convite para o agendamento ou diagnóstico.
4.  **COLETA**: Captura de Nome e E-mail (leads qualificados).
5.  **ENTREGA**: Confirmação final e encaminhamento para humano.

## 3. Hand-off Humano (Takeover)

O sistema possui um detector de intervenção humana. Quando um operador responde manualmente via WhatsApp Web/Celular, a IA silencia automaticamente por 1 hora para não interromper a conversa real.
