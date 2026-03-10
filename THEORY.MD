# Theory: Contrato Estável Para Fora, Hot Path Leve Para Dentro

## Problem

Os sintomas recentes se dividiram em dois grupos, mas com a mesma raiz operacional: o Evolution está servindo como boundary entre um provider complexo e um consumidor que precisa de contratos previsíveis e latência baixa. De um lado, havia inconsistências no contrato externo de `onWhatsapp` e dos webhooks, especialmente em torno de `lid`, `remoteJidAlt` e `addressingMode`. Do outro, apareceram atrasos intermitentes em `messages.upsert` e `messages.update`, sugerindo que o problema não era apenas o socket do Baileys, mas também trabalho evitável dentro da fila serial do próprio Evolution.

## Operating Theory

O ajuste seguro precisa continuar separando duas responsabilidades. O contrato externo deve ser estabilizado no boundary de emissão, porque alterar o objeto interno principal aumentaria o risco de regressão em cache, persistência e chatbot. Já o ganho de desempenho mais seguro não vem de paralelizar o mundo inteiro de uma vez, mas de cortar I/O redundante e serialização desnecessária do hot path. Como a fila de `ev.process()` é única, qualquer trabalho evitável dentro de `messages.upsert`, `messages.update` ou `contacts.*` se propaga para todo o sistema.

## Strategy

A linha adotada agora é incremental e conservadora. No lado de contrato, `lid` continua exposto como LID base do usuário, sem `:device`, e os webhooks seguem normalizados para sempre expor `remoteJid`, `remoteJidAlt`, `participant`, `participantAlt` e `addressingMode` quando o evento é de mensagem, além do trio `remoteJid`, `remoteJidAlt` e `addressingMode` em `contacts.update`. No lado de performance, os ganhos seguros vieram de três cortes: cache curto de `findSettings()` para evitar leitura de banco a cada batch, remoção de `console.log/error` do caminho quente e retirada do fetch remoto de avatar do fluxo síncrono de `messages.upsert`, reaproveitando DB e cache local sempre que possível.

## Key Discoveries

Os logs e a leitura do código confirmaram que o gargalo principal continua no hot path do Evolution, não no webhook em si. `findSettings()` fazia `findUnique` em todo batch do `ev.process()`, o que é desnecessário quando as settings mudam raramente e já são refletidas em `localSettings`. Em paralelo, `profilePicture()` era chamado em `messages.upsert` para praticamente toda mensagem recebida e também em `contacts.upsert`/`contacts.update`, o que transforma um metadata opcional em dependência síncrona de rede. A correção aplicada foi deixar `messages.upsert` usar apenas `profilePicUrl` já persistido ou em cache, e fazer `contacts.*` buscar foto remota só quando não houver `imgUrl` explícito nem valor reaproveitável do banco/cache. Isso reduz round-trips sem mudar o contrato externo. Também ficou claro que `console.log` no meio do handler custa mais do que parece num fluxo serial, porque força serialização extra mesmo quando não há valor operacional real naquele log.

## Event Backpressure

A hipótese de starvation por `status@broadcast` continua válida e foi tratada no fork do Baileys via `ignoreStatusBroadcast`, mas isso sozinho não resolve o atraso quando o próprio handler do Evolution carrega I/O desnecessário. Por isso a leitura correta agora é combinada: há otimizações no Baileys que aliviam a entrada, e há otimizações no Evolution que reduzem o tempo de serviço por batch. A telemetria opcional adicionada em `ev.process()` continua sendo a melhor ferramenta para diferenciar atraso de ingresso, fila saturada e trabalho excessivo dentro do handler.

## Open Questions

Ainda resta trabalho estrutural maior, como separar melhor histórico de tempo real ou revisar a fila única por tipo de evento, mas esses passos têm risco mais alto. O corte implementado nesta rodada foi deliberadamente menor e mais seguro: reduzir I/O redundante no hot path sem alterar a semântica de persistência, sem mexer em `saveCreds()` e sem introduzir concorrência nova em pontos críticos.
