# AI Battle Arena

Projeto de uma arena para batalhas contínuas entre dois agentes de IA, coordenadas por uma Arena central.

## Estado atual

A documentação inicial da arquitetura e do protocolo foi preparada. **A implementação ainda não começou.**

## Documentação

- [Especificação de arquitetura](docs/ARCHITECTURE.md)
- [Protocolo WebSocket](docs/WEBSOCKET_PROTOCOL.md)

## Diretrizes principais

- Dois agentes por batalha, com turnos alternados.
- A Arena mantém uma conexão WebSocket individual com cada agente.
- ACK confirma recebimento; `TURN_COMPLETE` confirma a conclusão do turno.
- O relógio de execução só começa após ACK da autorização do turno.
- Falha de entrega sem ACK não consome o turno.
- Estado e eventos são persistidos em SQLite para permitir recuperação.
- O Observer é opcional e avalia a batalha somente após o encerramento.

## Segurança

Este é um repositório público. Não publique tokens, senhas, chaves privadas, credenciais, endpoints privados ou dados pessoais. Use valores fictícios nos exemplos e mantenha segredos fora do controle de versão.

## Próximos passos

Contrato WebSocket revisado (v1.1): ACK bidirecional, persistência antes da confirmação, deduplicação e estado de entrega por destinatário. Modelo de estados da batalha e do turno documentado (arquitetura v1.1). Próximo passo: validar a regra de encerramento pendente e iniciar o núcleo mínimo com testes incrementais.
