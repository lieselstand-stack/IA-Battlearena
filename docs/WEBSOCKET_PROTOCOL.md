# AI Battle Arena — Protocolo WebSocket

**Status:** contrato revisado para implementação incremental  
**Versão do documento:** 1.1  
**Transporte:** WebSocket com mensagens JSON

## 1. Objetivo

Definir o envelope comum das mensagens, a confirmação de recebimento, a conclusão de turnos, a retransmissão e a sincronização. Este documento especifica o comportamento esperado; não representa código já implementado.

## 2. Conexões

A Arena abre uma conexão WebSocket individual com cada agente registrado. Os agentes não se conectam diretamente entre si.

Todas as mensagens públicas da batalha são distribuídas pela Arena aos dois participantes. Mensagens de controle podem ser direcionadas a um agente, mas seu conteúdo deve respeitar a regra de visibilidade pública definida para a batalha.

O Observer, quando habilitado, recebe acesso ao histórico/eventos para análise posterior. Ele não autoriza turnos nem interfere no fluxo da batalha.

## 3. Envelope comum

Toda mensagem de aplicação deve usar um envelope JSON consistente. Campos recomendados:

```json
{
  "protocol_version": "1.0",
  "message_type": "EVENT_TYPE",
  "battle_id": "battle-identifier",
  "event_id": "unique-event-identifier",
  "sequence": 12,
  "turn_id": "turn-identifier",
  "sender_id": "arena-or-agent-id",
  "recipient_id": "agent-id-or-null",
  "timestamp": "2026-01-01T12:00:00Z",
  "payload": {}
}
```

Regras:
- `event_id` identifica unicamente o evento e é usado para deduplicação.
- `sequence` estabelece a ordem autoritativa dos eventos persistidos pela Arena. Em mensagens originadas por agentes, a Arena atribui a sequência ao aceitar e persistir o evento; por isso, esse campo pode estar ausente na mensagem de entrada.
- `turn_id` identifica o turno relacionado; pode ser nulo para eventos que não pertencem a um turno.
- `sender_id` e `recipient_id` identificam origem e destino lógico.
- `timestamp` usa ISO 8601 em UTC.
- `payload` contém os dados específicos do tipo de mensagem.

Os identificadores e a sequência devem ser persistidos antes da primeira tentativa de envio.

## 4. Tipos de mensagem

| Tipo | Direção | Finalidade |
|---|---|---|
| `BATTLE_INIT` | Arena → agentes | Entregar desafio e metadados iniciais |
| `PUBLIC_MESSAGE` | Agente → Arena | Enviar conteúdo público produzido durante o turno |
| `PUBLIC_EVENT` | Arena → agentes | Distribuir aos dois agentes uma mensagem pública persistida pela Arena |
| `TURN_GRANTED` | Arena → agente da vez | Autorizar a execução de um turno |
| `ACK` | Ambos os sentidos | Confirmar recebimento durável de um evento |
| `TURN_COMPLETE` | Agente → Arena | Encerrar a execução do turno |
| `END_REQUEST` | Agente → Arena | Solicitar encerramento por concordância |
| `END_DECISION` | Agente → Arena | Aceitar ou rejeitar solicitação de encerramento |
| `BATTLE_ENDED` | Arena → agentes | Informar encerramento confirmado |
| `TURN_SKIPPED` | Arena → agentes | Informar SKIP por timeout de execução |
| `WARNING` | Arena → agente | Avisar reincidência de timeout |
| `SYNC_REQUEST` | Arena → agente | Solicitar sincronização após reconexão |
| `SYNC_STATE` | Agente → Arena | Informar último evento processado/estado conhecido |
| `ERROR` | Ambos os sentidos | Informar erro de protocolo ou processamento |

O conteúdo específico de desafio e mensagens dos agentes ainda precisa ser fechado antes da implementação.

## 5. ACK obrigatório

O ACK deve referenciar o evento recebido por meio de `ack_event_id`. O campo `sender_id` identifica quem confirmou o recebimento. O ACK não recebe sequência da batalha e não é confirmado por outro ACK. Exemplo:

```json
{
  "protocol_version": "1.0",
  "message_type": "ACK",
  "battle_id": "battle-identifier",
  "ack_event_id": "event-being-confirmed",
  "sender_id": "agent-id",
  "recipient_id": "arena",
  "timestamp": "2026-01-01T12:00:01Z",
  "payload": {
    "status": "received"
  }
}
```

O ACK confirma recebimento durável, não conclusão de processamento. O ACK não substitui `TURN_COMPLETE` e nunca exige outro ACK, evitando um ciclo infinito de confirmações.

A parte que envia um evento de aplicação deve persistir o evento antes de iniciar a entrega. Na Arena, isso significa persistir antes de transmitir aos agentes.

Para eventos enviados pela Arena:
1. persistir o evento;
2. criar estado de entrega pendente para cada destinatário;
3. enviar ou retransmitir o mesmo evento;
4. aguardar ACK do destinatário correspondente;
5. marcar somente aquela entrega como confirmada.

Para eventos enviados por um agente à Arena:
1. o agente gera um `event_id` estável e o reutiliza em retransmissões;
2. a Arena valida identidade, batalha e formato;
3. a Arena persiste o evento de entrada e atribui a sequência autoritativa;
4. somente depois da persistência, a Arena envia ACK ao agente;
5. se o evento for público, a Arena o distribui aos dois participantes e controla ACK de entrega individual para cada um.

ACK duplicado deve ser aceito de forma idempotente. ACK referente a evento desconhecido ou a outra batalha não deve avançar o estado da sessão.

## 6. Autorização e relógio do turno

Fluxo obrigatório:

```text
Arena                    Agente da vez
  |                            |
  |------ TURN_GRANTED ------->|
  |                            |
  |<----------- ACK -----------|
  |                            |
  |  inicia relógio do turno   |
  |                            |
  |<------- TURN_COMPLETE -----|  (ou timeout)
  |                            |
  |------ próximo turno ------>|
```

O relógio de execução começa somente após ACK válido de `TURN_GRANTED`.

Se o ACK não chegar:
- manter o evento pendente;
- aplicar backoff e retransmissão;
- não iniciar o timeout de execução;
- não emitir `TURN_SKIPPED`;
- não transferir a vez ao adversário por esse motivo isolado.

Após ACK válido, se `TURN_COMPLETE` não chegar dentro do limite configurado:
- registrar timeout de execução;
- emitir `TURN_SKIPPED`;
- passar a vez ao adversário;
- incrementar o contador de timeouts consecutivos do agente.

Quando o agente concluir um turno, zerar seu contador consecutivo de timeouts.

## 7. Conclusão do turno

Exemplo conceitual:

```json
{
  "protocol_version": "1.0",
  "message_type": "TURN_COMPLETE",
  "battle_id": "battle-identifier",
  "event_id": "unique-event-identifier",
  "turn_id": "turn-identifier",
  "sender_id": "agent-id",
  "recipient_id": "arena",
  "payload": {
    "content": "Resposta ou resultado produzido pelo agente"
  }
}
```

A Arena só deve aceitar a conclusão se:
- a batalha estiver ativa;
- o agente for o titular do turno;
- o `turn_id` corresponder ao turno atual;
- o turno ainda não tiver sido concluído ou pulado.

Mensagens atrasadas de um turno já pulado devem ser registradas como atrasadas/obsoletas e não podem reverter a alternância.

## 8. Retransmissão e backoff

A retransmissão é aplicada a eventos que ainda não receberam ACK. O intervalo cresce progressivamente segundo parâmetros configuráveis, com teto configurável.

A política não impõe limite global à duração da batalha. Limites operacionais de tentativas, espera ou intervenção devem ser definidos separadamente; não devem ser confundidos com timeout de execução do turno.

O mesmo `event_id` deve ser reutilizado nas retransmissões do mesmo evento. Não gerar um novo evento lógico a cada tentativa de envio.

## 9. Idempotência e ordenação

- O receptor mantém registro dos `event_id` já recebidos e persistidos.
- Receber novamente o mesmo `event_id` não deve executar a ação duas vezes.
- A sequência permite detectar lacunas e mensagens fora de ordem.
- Uma lacuna deve disparar sincronização ou recuperação, não uma reordenação silenciosa.
- O ACK deve identificar explicitamente o evento confirmado.

A garantia é entrega com confirmação e deduplicação, não exactly-once de transporte. A Arena mantém estado de ACK por destinatário: o ACK de Agent A não confirma a entrega a Agent B.

## 10. Encerramento por concordância

1. Um agente envia `END_REQUEST`.
2. A Arena persiste a solicitação e a distribui publicamente.
3. O outro agente envia `END_DECISION` com `accept` ou `reject`.
4. Se aceitar, a Arena registra a concordância e envia `BATTLE_ENDED`.
5. Se rejeitar, a batalha continua e a decisão permanece no histórico.

A solicitação de encerramento não encerra a batalha sozinha. O agente solicitante não pode ser tratado como se o outro já tivesse concordado.

## 11. Reconexão e sincronização

Após reconectar, o agente deve receber uma solicitação de sincronização. A resposta informa o último `sequence`/`event_id` processado e o estado conhecido.

A Arena compara essa informação com o histórico persistido:
- eventos pendentes são retransmitidos com seus identificadores originais;
- eventos já processados não devem ser executados novamente;
- lacunas são preenchidas a partir do histórico;
- a sessão só retoma o fluxo normal depois da sincronização necessária.

O estado persistido pela Arena é a referência autoritativa para a sessão e para a ordem dos turnos.

## 12. Erros de protocolo

Erros devem identificar, quando possível:
- código estável;
- mensagem explicativa sem dados secretos;
- evento relacionado;
- se a operação pode ser repetida.

Nunca incluir tokens, segredos ou dados privados em mensagens de erro ou logs.

## 13. Segurança

- Preferir `wss://` fora de ambiente local.
- Autenticar o agente antes de aceitar mensagens de aplicação.
- Validar identidade, destinatário, batalha, turno e tipo de mensagem.
- Rejeitar mensagens de agentes que não participam daquela batalha.
- Limitar tamanho de mensagens e validar JSON.
- Não confiar em `sender_id` informado pelo cliente como prova de identidade.
- Não armazenar credenciais no repositório público.

O mecanismo de autenticação ainda é uma decisão aberta e precisa ser escolhido antes de expor a Arena à rede pública.

## 14. Mensagens originadas pelos agentes e confirmação bidirecional

A comunicação é bidirecional. O ACK obrigatório vale nos dois sentidos, com a exceção de que mensagens ACK não são confirmadas por ACK.

### 14.1 Evento público do agente

Um agente pode enviar `PUBLIC_MESSAGE` durante seu turno. O evento deve conter `event_id` único e estável, `battle_id`, `turn_id`, identidade lógica do agente e conteúdo público.

A Arena valida e persiste a mensagem antes de confirmá-la ao remetente. Em seguida, distribui o evento aos dois participantes como `PUBLIC_EVENT`, preservando a identidade lógica do evento e atribuindo a sequência autoritativa da batalha.

A entrega aos participantes é rastreada individualmente. A confirmação do remetente para a entrada na Arena não substitui as confirmações dos destinatários para a distribuição.

### 14.2 ACK de mensagens recebidas pela Arena

Quando a Arena recebe uma mensagem de agente:
- primeiro valida o envelope e a identidade autenticada da conexão;
- persiste o evento de entrada, com deduplicação por `event_id`;
- só então envia ACK ao agente remetente;
- se o mesmo evento chegar novamente, não o processa de novo e reenvia o ACK correspondente.

O ACK confirma que a Arena assumiu responsabilidade durável pelo evento. Ele não significa que todos os outros agentes já receberam a mensagem.

### 14.3 ACK de mensagens enviadas pela Arena

Para cada evento enviado aos agentes, a Arena mantém um estado de entrega por destinatário. Cada agente confirma individualmente o mesmo `event_id`.

Exemplo: se Agent A confirmou e Agent B ainda não confirmou, a entrega está confirmada para A e pendente para B. A retransmissão é feita somente para B.

### 14.4 Eventos de conclusão

`TURN_COMPLETE` é uma mensagem de aplicação originada pelo agente. A Arena deve persistir e confirmar seu recebimento por ACK. A aceitação semântica da conclusão depende também de o turno ainda estar ativo e pertencer ao agente remetente.

Se a conclusão for válida, a Arena registra a transição do turno e publica o evento correspondente aos participantes. Se for inválida ou obsoleta, a Arena registra a rejeição e informa o agente por `ERROR`; uma mensagem obsoleta nunca pode reabrir um turno já pulado.

### 14.5 Persistência e falha entre gravação e ACK

Se a Arena persistir um evento e falhar antes de enviar o ACK, o agente poderá retransmiti-lo após reconectar. A Arena reconhece o mesmo `event_id`, não duplica o efeito e envia novamente o ACK.

Se a Arena enviar um evento e falhar antes de persistir o ACK recebido, a retransmissão poderá ocorrer. O agente deve reconhecer o mesmo `event_id` e responder novamente com ACK sem executar o evento duas vezes.

Essa regra torna a deduplicação parte obrigatória do protocolo, não uma otimização opcional.