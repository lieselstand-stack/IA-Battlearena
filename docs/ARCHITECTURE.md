# AI Battle Arena — Especificação de Arquitetura

**Status:** especificação inicial aprovada para orientar a implementação  
**Versão do documento:** 1.0  
**Branch de documentação:** `docs/architecture-v1`

## 1. Objetivo

A AI Battle Arena coordena uma batalha contínua entre exatamente dois agentes de IA. A Arena fornece o desafio inicial, controla a alternância dos turnos, distribui as mensagens públicas, registra os eventos e permite avaliação posterior.

O sistema deve privilegiar previsibilidade, recuperação após falhas, rastreabilidade e implementação incremental.

## 2. Escopo da primeira versão

- Uma batalha ativa por vez.
- Exatamente dois agentes participantes, previamente cadastrados.
- Duração global ilimitada.
- Turnos alternados.
- Comunicação via WebSocket.
- A Arena inicia e mantém uma conexão individual com cada agente.
- Todas as mensagens da batalha são públicas para os dois participantes.
- O encerramento exige concordância dos dois agentes.
- Persistência em SQLite.
- Recuperação automática da sessão após reinicialização.
- ACK obrigatório para eventos entregues.
- Observer opcional, com avaliação posterior à batalha.

Fora do escopo inicial: múltiplas batalhas simultâneas, comunicação direta agente-agente, avaliação do Observer durante a batalha e execução distribuída em múltiplos servidores.

## 3. Componentes lógicos

### 3.1 Arena

Responsável por:
- receber a configuração da batalha e o desafio inicial;
- selecionar os dois agentes cadastrados;
- estabelecer conexões WebSocket individuais;
- encaminhar eventos públicos;
- aguardar ACK antes de iniciar o relógio de execução de um turno;
- controlar a alternância e os timeouts de execução;
- registrar eventos e estado da sessão;
- recuperar e sincronizar uma sessão interrompida.

### 3.2 Agent Registry

Mantém o cadastro dos agentes e seus metadados de conexão. O cadastro também pode identificar um agente elegível para atuar como Observer.

O Observer cadastrado não é contado como um dos dois participantes da batalha.

### 3.3 Agentes

Cada agente:
- recebe o desafio inicial e os eventos públicos;
- recebe uma autorização explícita quando chega sua vez;
- confirma o recebimento por ACK;
- executa sua estratégia de forma independente;
- envia a conclusão do turno por uma mensagem distinta do ACK;
- pode solicitar o encerramento da batalha.

Os agentes não precisam conhecer nem acessar o endpoint um do outro. A Arena intermedeia a comunicação.

### 3.4 Observer

É opcional e não participa do controle da batalha. Depois do encerramento, um Observer cadastrado analisa o histórico. A avaliação é híbrida: a IA produz a análise e um humano pode acrescentar comentários ou corrigir a avaliação.

A avaliação original da IA deve ser preservada. A revisão humana deve ser registrada separadamente, sem sobrescrever silenciosamente o resultado original.

### 3.5 Persistência

SQLite armazena:
- configuração e estado da batalha;
- identidade dos participantes;
- sequência de eventos;
- estado de entrega e ACK;
- turnos, timeouts e avisos;
- solicitações e decisões de encerramento;
- avaliação original do Observer e revisão humana.

Os eventos da batalha devem ser tratados como histórico append-only. Alterações de estado não devem apagar o histórico que permite reconstruir a sessão.

## 4. Topologia de comunicação

A topologia aprovada é centralizada:

```
             +----------------+
             |      Arena     |
             +----------------+
                /          \\
       WebSocket            WebSocket
             /                \\
     +-----------+        +-----------+
     |  Agent A  |        |  Agent B  |
     +-----------+        +-----------+

       Observer (opcional; leitura do histórico)
```

Há uma conexão Arena–Agent A e outra Arena–Agent B. Não há conexão direta obrigatória entre os agentes.

A opção de conexão registrada anteriormente como `both` permanece anotada, mas seu significado exato não está suficientemente definido para alterar esta topologia. O modelo operacional selecionado posteriormente foi `arena`: a Arena inicia as conexões.

## 5. Controle da batalha

### 5.1 Autoridade

Os agentes controlam suas estratégias e podem propor o encerramento. A Arena/Controller controla a infraestrutura, a autorização de turnos, a entrega de eventos e a aplicação das regras de tempo.

### 5.2 Início

1. O operador fornece o desafio inicial.
2. O operador escolhe qual agente começa: A ou B.
3. A Arena registra a sessão e o desafio.
4. A Arena distribui o desafio aos dois agentes e aguarda as confirmações de recebimento.
5. A Arena autoriza o primeiro agente.
6. O relógio de execução só começa após o ACK da autorização de turno.

### 5.3 Alternância

Os turnos são alternados: A, B, A, B e assim por diante. O WebSocket é bidirecional, mas isso não significa execução simultânea. O Controller concede autorização a apenas um agente por vez.

### 5.4 ACK e conclusão são eventos diferentes

- **ACK:** confirma que o agente recebeu determinado evento.
- **TURN_COMPLETE:** informa que o agente concluiu seu turno.

O ACK não significa que o agente terminou de pensar ou executar. O timeout de execução começa somente depois do ACK da autorização de turno.

### 5.5 Timeout e SKIP

- Se a autorização ainda não recebeu ACK, a entrega permanece pendente e segue a política de retransmissão. O relógio de execução não começa e não há SKIP.
- Depois do ACK, se o agente não enviar `TURN_COMPLETE` dentro do limite configurado, a Arena registra timeout de execução e pula o turno.
- O turno passa ao adversário.
- A duração global da batalha continua ilimitada; o limite aplica-se individualmente à execução de cada turno.

### 5.6 Avisos por reincidência

Após N timeouts de execução consecutivos, a Arena emite um aviso ao agente. N é configurável. O contador é reiniciado quando o agente conclui um turno. A política aprovada não elimina nem suspende automaticamente o agente.

## 6. Encerramento

Qualquer agente pode enviar uma solicitação de encerramento. A batalha só termina quando ambos concordam.

A solicitação deve ser registrada como pendente até que o outro agente responda explicitamente:
- aceitar; ou
- rejeitar.

Uma rejeição mantém a batalha ativa. A solicitação, a resposta e o resultado devem permanecer no histórico. A semântica de uma nova solicitação após rejeição deve ser definida pelo fluxo de eventos, sem apagar a tentativa anterior.

## 7. Recuperação

Após reinicialização da Arena:
1. carregar do SQLite a sessão que estava ativa;
2. identificar os agentes e o último estado persistido;
3. restabelecer as conexões;
4. sincronizar os agentes com o último evento confirmado;
5. reenviar eventos pendentes conforme a política de entrega;
6. deduplicar eventos já processados;
7. retomar a batalha somente após a sincronização necessária.

A recuperação não deve criar um novo turno nem perder mensagens já persistidas.

## 8. Garantias e consistência

- ACK obrigatório.
- Retransmissão com backoff progressivo e parâmetros configuráveis.
- Cada evento possui identificador único e sequência por batalha.
- O receptor deduplica eventos pelo identificador.
- A persistência precede a distribuição.
- A entrega pode ocorrer mais de uma vez; o processamento deve ser idempotente.
- Não prometer exactly-once. O desenho usa entrega com ACK e deduplicação para obter comportamento efetivamente idempotente.

O backoff pode ter intervalo máximo configurável. A batalha não possui duração global máxima.

## 9. Segurança para repositório público

Este repositório é público. Portanto:
- nunca versionar tokens, senhas, chaves privadas, cookies ou credenciais;
- nunca incluir endpoints privados, IPs residenciais ou dados pessoais reais;
- usar valores fictícios em exemplos;
- manter segredos somente na configuração local/ambiente de execução;
- usar TLS (`wss://`) em conexões fora de ambiente local;
- não registrar segredos em logs;
- documentar variáveis de ambiente apenas por nome e finalidade, sem valores reais.

O mecanismo concreto de autenticação dos agentes ainda precisa ser decidido antes da implementação de produção. O cadastro prévio não deve ser confundido com autenticação segura.

## 10. Decisões ainda abertas

As decisões abaixo não devem ser inventadas durante a implementação:

1. Definição exata do valor de conexão `both`, registrado no início da concepção.
2. Mecanismo de autenticação e autorização dos agentes.
3. Limite de tempo padrão por turno e limites mínimos/máximos.
4. Valores padrão e teto do backoff.
5. Formato final do payload de desafio e das mensagens de conteúdo dos agentes.
6. Política de reconexão quando um agente permanece indisponível por longo período.
7. Regras detalhadas para uma nova solicitação de encerramento após rejeição.

Esses pontos podem ser definidos individualmente antes de cada etapa de código.

## 11. Ordem de implementação proposta

1. Definir e validar o contrato de mensagens WebSocket.
2. Criar o núcleo mínimo da sessão e alternância de turnos.
3. Criar dois agentes de teste simples.
4. Adicionar persistência SQLite e deduplicação.
5. Testar ACK ausente, retransmissão, timeout após ACK e recuperação.
6. Adicionar o fluxo de encerramento por concordância.
7. Adicionar Observer e revisão humana.

Cada etapa deve ter testes próprios, commit identificável e atualização breve do README. Não introduzir camadas ou módulos sem necessidade demonstrável.

## 12. Princípio de implementação

Preservar simplicidade e compatibilidade. Reutilizar estruturas existentes sempre que possível. Modularizar quando reduzir a complexidade real, não por convenção. Nenhuma etapa deve avançar sem validar o comportamento da etapa anterior.
