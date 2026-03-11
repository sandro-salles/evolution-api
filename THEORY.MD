# Theory: Preserve LID in Evolution Outbound Send Path

## Problem

O contrato externo entre Contact-Z e Evolution passou a carregar corretamente
`remoteJid`, `remoteJidAlt` e `addressingMode` para contatos WhatsApp com LID.
Mesmo assim, respostas humanas continuaram saindo em `pn` em casos concretos de
produção. O risco aqui não é apenas cosmético no webhook de `send.message`; é a
possibilidade de o endereço efetivo do envio divergir da preferência persistida
da sessão.

## Operating Theory

O `service` já estava enviando o JID correto. Em produção, a sessão analisada
estava com `preferred_external_id = ...@lid`, e os logs do Evolution mostraram:

- `Number ...@lid found in cache`
- `Sending message to ...@s.whatsapp.net`

Isso desloca a causa para dentro do Evolution. O send path usa
`sendMessageWithTyping(number, ...)`, que primeiro chama `whatsappNumber()` e
depois usa `isWA.jid` como destino real. Quando há cache hit para um contato com
LID resolvido, `whatsappNumber()` devolve `cached.remoteJid` como `jid`, e esse
campo hoje guarda o PN (`@s.whatsapp.net`). O LID permanece em `lid`, mas deixa
de ser o endereço efetivo do envio.

## Strategy

A correção deve ser cirúrgica. `whatsappNumber()` continua útil para verificar
existência do contato e recuperar metadados, então a mudança mais segura é no
consumo do resultado, não no contrato global do helper.

O princípio é:

- se o caller pediu explicitamente um `@lid`, o send path deve preservar esse
  LID como destino real;
- se o caller pediu `@s.whatsapp.net`, o comportamento atual permanece;
- presença (`sendPresence`) deve seguir a mesma regra para não reintroduzir
  inconsistência entre typing/presence e envio.

## Key Discoveries

O cache de on-whatsapp foi desenhado para manter mapeamento entre PN e LID, mas
o send path passou a tratá-lo como autoridade para o endereço final. Esse é o
desvio. O helper ainda serve para validar existência, porém não deve rebaixar o
addressing mode escolhido pelo caller. A correção precisa preservar o LID só no
outbound explícito, evitando alterar outros consumidores de `whatsappNumber()`
como bloqueio, labels, profile e catálogos, que podem continuar dependendo do
PN resolvido.

O patch aplicado segue exatamente essa linha: um helper local resolve o JID
outbound preservando `@lid` quando ele já veio explícito no argumento original,
e `sendMessageWithTyping`/`sendPresence` passaram a usar esse helper no lugar de
`isWA.jid` diretamente. A validação com `npm run build` passou, o que aumenta a
confiança de que o ajuste está isolado ao hot path de envio sem quebrar
tipagem nem bundle.

## Open Questions

- Pode haver outros fluxos outbound além de `sendMessageWithTyping` e
  `sendPresence` que também precisem preservar LID explicitamente.
- Depois do patch, vale validar se o webhook de `send.message` passa a refletir
  `remoteJid = ...@lid` e `addressingMode = lid` como esperado, sem regressão em
  números puramente PN.
