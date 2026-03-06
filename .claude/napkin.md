# Napkin

## Corrections
| Date | Source | What Went Wrong | What To Do Instead |
|------|--------|----------------|-------------------|
| 2026-03-05 | self | Assumi inicialmente que o campo `lid` já representava o LID real no fluxo `whatsappNumber`, mas ele era usado só como flag literal `'lid'`. | Verificar o contrato completo até o provider e confirmar o significado real de campos opcionais antes de propagar comportamento novo. |
| 2026-03-05 | self | Tentei validar com `npx tsc` e `npx eslint`, mas o ambiente sem `node_modules` puxou binários globais/incorretos. | Quando o repo estiver sem dependências instaladas, declarar a limitação explicitamente e evitar confiar em `npx` sem toolchain local. |

## User Preferences
- Responder em PT-BR ao explicar mudanças no código.
- Quando o `lid` for exposto em API, preferir o LID base do usuário, sem o sufixo `:device`.

## Patterns That Work
- Inspecionar primeiro o fluxo completo `route -> controller -> service -> cache` antes de alterar contratos de resposta.
- Quando o comportamento depende do Baileys, validar a API no checkout local do pacote antes de implementar.
- Para documentar payload de webhook, seguir `sendDataWebhook -> webhook.controller -> ponto de emissão` e não assumir que o contrato é uniforme entre eventos do mesmo nome.
- Quando um contrato externo precisa ser estabilizado sem arriscar regressão interna, normalizar o payload apenas no boundary do webhook e preservar os objetos internos usados por cache/chatbot/DB.
- Para `contacts.update`, combinar `contact.id/lid/phoneNumber` com `getOnWhatsappCache()` e `signalRepository.lidMapping` cobre tanto eventos ricos (`contacts.upsert`) quanto updates pobres (`notify`, `picture`).
- Para validar localmente o Evolution alinhado ao runtime do projeto, usar `source ~/.nvm/nvm.sh && nvm use 24` antes de `npm install`/`npm run build`.

## Patterns That Don't Work
- Inferir APIs do Baileys sem abrir o código-fonte local quando há integrações específicas como `lidMapping`.
- Usar `npx` para validar TypeScript/ESLint em repo sem dependências instaladas; isso pode executar versões erradas das ferramentas.

## Domain Notes
- O endpoint público para checagem de números é `POST /chat/whatsappNumbers`.
- Existe um endpoint interno `POST /baileys/onWhatsapp` que hoje retorna o resultado cru de `client.onWhatsApp()`.
- O cache `IsOnWhatsapp` já possui coluna `lid` em PostgreSQL e `psql_bouncer`; o contrato desejado para API é o LID base do usuário, sem `:device`.
- O envelope HTTP do webhook inclui `event`, `instance`, `data`, `destination`, `date_time`, `sender`, `server_url` e `apikey`.
- Alguns eventos não têm shape único no Baileys atual: `contacts.update` pode sair como objeto único ou array; `messages.delete` pode sair em forma compacta (`key + status`) ou enriquecida com campos da mensagem.
- O contrato desejado para webhooks de mensagem agora é `data.key` sempre contendo `remoteJid`, `remoteJidAlt`, `participant`, `participantAlt` e `addressingMode`, ainda que alguns desses campos precisem sair como `null`.
- O contrato desejado para `contacts.update` agora é cada item sempre conter `remoteJid`, `remoteJidAlt` e `addressingMode`; para grupos, `remoteJidAlt` e `addressingMode` podem continuar `null`.
