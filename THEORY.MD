# Theory: Normalização de Contratos Externos no Baileys

## Problem

A aplicação consumidora depende de contratos estáveis em dois pontos sensíveis do Evolution API: a resolução de contato (`onWhatsapp`) e os webhooks de mensagens. O risco central é o mesmo nos dois fluxos: o provider já possui a informação correta, mas o contrato externo é exposto com variações de shape e sem uma camada explícita de normalização. No caso dos webhooks, isso aparecia sobretudo em `messages.upsert`, `messages.edited` e `send.message.update`, onde `data.key` podia ou não trazer `remoteJidAlt` e `addressingMode` dependendo da origem do evento.

## Operating Theory

O lugar certo para estabilizar esse contrato não é o core do processamento da mensagem, mas o boundary de emissão. Internamente, o serviço já possui comportamentos dependentes do Baileys, inclusive trechos que convertem `remoteJid` de `@lid` para PN antes de seguir com chatbot, cache e persistência. Tentar “corrigir” isso no objeto principal teria alto risco de regressão. A hipótese mais segura é manter o fluxo interno intacto e introduzir uma normalização somente na cópia enviada para fora.

## Strategy

A estratégia passa a ser dupla. Para `onWhatsapp`, a saída continua normalizada para o LID base do usuário, sem `:device`, e o cache precisa privilegiar um `remoteJid` em PN sempre que o `lidMapping` conseguir resolver esse par; guardar apenas `@lid` como `remoteJid`, `jidOptions` e `lid` ao mesmo tempo gera um registro redundante e pouco reutilizável. Para webhooks, a camada de saída passa a construir contratos estáveis por tipo de evento. Em mensagens, `data.key` deve sempre conter `remoteJid`, `remoteJidAlt`, `participant`, `participantAlt` e `addressingMode`, permitindo `null` quando o Baileys realmente não fornece o valor. Em `contacts.update`, cada item passa a expor `remoteJid`, `remoteJidAlt` e `addressingMode`, usando os campos explícitos do `Contact` quando existirem e completando o restante via cache local e `lidMapping`. A regra crucial é que esse trio precisa ser derivado do par final normalizado: se o endereço alternativo colapsa para o mesmo valor do primário, ele deve ser removido e o `addressingMode` precisa refletir o endereço primário realmente exposto, não uma hipótese intermediária de resolução.

## Key Discoveries

O envelope HTTP continua uniforme e sempre inclui `event`, `instance`, `data`, `destination`, `date_time`, `sender`, `server_url` e `apikey`; a inconsistência estava concentrada no campo `data`. O checkout local do Baileys confirmou que `WAMessageKey` já suporta `remoteJidAlt`, `participantAlt` e `addressingMode`, mas o Evolution não propagava isso de forma uniforme em todos os eventos. Também ficou claro que `Contact` no branch local do Baileys já carrega `id`, `lid` e `phoneNumber`, mas os eventos `contacts.update` mais pobres frequentemente chegam apenas com `id` e metadados como `notify` ou `imgUrl`. A correção mais segura foi criar helpers de normalização no serviço Baileys e aplicá-los apenas nos pontos de emissão de `messages.upsert`, `messages.edited`, `send.message`, `send.message.update` e `contacts.update`, complementando faltas de addressing em contatos via `getOnWhatsappCache()` e `signalRepository.lidMapping`. A condição operacional mudou: com `source ~/.nvm/nvm.sh && nvm use 24`, o ambiente local passa a espelhar o runtime do projeto e fica apto para validar install/build com a pilha atual em vez de depender só de checagens estruturais.

## Open Questions

Ainda existe uma questão estrutural fora desse ajuste: o repositório permanece assimétrico entre provedores de banco, porque o fluxo de `lid` já está alinhado no ambiente ativo, mas o schema MySQL local não acompanha perfeitamente o mesmo desenho. Isso não bloqueia a normalização do contrato externo, porém segue sendo um ponto a revisar para evitar deriva entre providers.
