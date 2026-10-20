# Lições aprendidas

Este arquivo reúne os incidentes e as surpresas que apareceram desde o início da integração
(setembro de 2026). Quase todos têm a mesma raiz: a documentação oficial das APIs descreve um
comportamento e a API real faz outro. Leia antes de mexer no workflow ou em
`preencher_fila_sync()`.

## Regras que saíram dessas histórias

1. **Teste contra o payload real.** Quando uma chamada documentada como simples não funciona,
   o atalho é fazer a ação à mão na tela, capturar a requisição pela aba Network do navegador
   (F12) e comparar campo a campo. É mais rápido do que testar parâmetro por parâmetro.
2. **Escopo na fonte, não num toggle.** Uma proteção que depende de alguém lembrar de manter um
   node desligado vai falhar um dia. Restrinja o escopo dentro da view ou da query.
3. **Quem ganha poder de escrita ganha também um novo risco.** Quando um caminho que só falhava
   com segurança (404) passa a criar dados, reavalie o escopo de tudo que alimenta esse caminho.
4. **Eco se compara por valor, depois de normalizar.** Texto que vai e volta de uma API externa
   raramente volta idêntico (HTML escapado, quebras de linha achatadas). Normalize antes de comparar.
5. **Antes de construir uma busca nova, veja o que já vem no payload.** Boa parte dos dados do
   lead chega de carona no `GET` do lead que o workflow já faz em todo evento.

## Setembro de 2026

### Webhook sem escopo criou cerca de 90 contatos indevidos (22/set)

Na manhã seguinte ao cadastro dos webhooks de produção no CVCRM, apareceram uns 90 contatos
novos no GHL, sem nome, de leads de outros empreendimentos. Os quatro webhooks estavam com o
campo "Empreendimentos" em branco, que significa todos. Isso era inofensivo enquanto o despacho
só sabia atualizar contatos. No mesmo dia ele passou a criar contato novo quando não existia, e
a combinação não foi reavaliada.

Correção: os webhooks foram restritos ao empreendimento do piloto e os contatos foram removidos.

Poucas horas depois, um segundo lote de uns 91 contatos apareceu, vindo do node `Reconciliar CVCRM`,
que estava ativo quando devia estar desligado. Ele varre `bronze.leads` inteiro sem filtro de
empreendimento. Desta vez a restrição foi para dentro da view `integracao.v_reconciliacao_cvcrm`
(regra 2 acima).

### O payload real do webhook de Oportunidade do GHL não bate com a documentação (23/set)

A documentação descreve `pipelineStageId` e `contactId`. A ação Webhook de um workflow do GHL
manda outra coisa: o corpo achatado de sempre (telefone, e-mail, tags, campos personalizados) mais
os dados da oportunidade como texto solto, com dois erros de digitação do próprio GHL:
`pipleline_stage` (nome da etapa em texto) e `pipleline_id`. Como o nome da etapa já vem pronto,
o de-para `integracao.depara_situacao_ghl` funciona por nome e não precisou do id do estágio.

### O gatilho "Nova interação" do CVCRM não dispara (23/set)

Dezenas de anotações de teste, nenhuma chamou o webhook. Outros gatilhos do mesmo webhook
funcionam normalmente. O texto do chamado aberto está em
[`chamado-cvcrm-nova-interacao.md`](chamado-cvcrm-nova-interacao.md).

A solução não depende do gatilho. O node `Buscar lead CVCRM` traz o lead inteiro em qualquer
evento, então a última interação já vem junto. `Normalizar evento CVCRM` envia também
`interacao_cvcrm_id`, e `preencher_fila_sync()` compara com
`depara_contato.ultima_interacao_id_cvcrm` para não reenviar uma anotação antiga toda vez que
outro campo mudar.

Do lado do GHL a direção contrária usa um gatilho de verdade: o workflow "Nota adicionada". O
texto chega em `note.body`, um objeto aninhado.

### Loop infinito de eco entre nota do GHL e interação do CVCRM (23/set)

Uma nota criada de um lado virava interação do outro, que virava nota de novo, e assim por
diante. Uma única mensagem de teste virou quatro duplicatas em 16 minutos, até o gatilho de
despacho ser desligado à mão.

Causa: a detecção de eco genérica (`eh_eco`) compara o hash de todos os campos do evento, e o
formato muda de um lado para o outro. `nota_ghl` chega sozinho, `interacao_cvcrm` chega junto
de corretor, empreendimento e origem. Hashes de formas diferentes nunca são iguais.

Correção: duas checagens novas em `preencher_fila_sync()`, por texto. Compara o valor que
chegou com o último registro de `log_sync` e, se for igual, tira o campo do evento antes de
calcular o hash.

### Tarefa: `lead.tarefa` já vem no `GET` do lead (23/set)

A primeira versão buscava a tarefa num endpoint à parte, pelo `idtarefa` que o gatilho "Nova
tarefa" manda. Funcionava, mas quando outro gatilho disparava primeiro (sem `idtarefa`), a
tarefa nunca chegava. Descobriu-se então que `lead.tarefa` já vem embutido no `GET` do lead,
igual à interação. A tarefa passou a usar o mesmo esquema de carona e de controle por id
(`ultima_tarefa_id_cvcrm`).

Detalhe: reaplicar o SQL no meio de uma sequência de testes pode deixar passar um evento de
transição, porque a coluna de controle começa vazia. O efeito some no evento seguinte.

### Situação voltou a ser nos dois sentidos (24/set)

Escrever no GHL não é um campo personalizado. É um `PUT /opportunities/:id` que muda o estágio
do pipeline, e isso exige antes resolver o id do estágio (`depara_situacao_ghl`) e o id da
Oportunidade do contato (`GET /opportunities/search?contactId=...&pipelineId=...`). O eco é
detectado por valor (o `idsituacao` numérico).

Pegadinha do n8n: a condição de "Tem situação pra mudar?" comparava um número usando tipo
`string` com validação estrita e dava "Wrong type". Resolvido com `String(...)` na expressão.

### Visita do CVCRM vira Compromisso no GHL (24/set)

Visita é uma tarefa do CVCRM com `tipo_interacao = 'V'`. O array `lead.tarefa` embutido não traz
esse campo, então para separar visita de tarefa o workflow voltou a usar o endpoint
`/cvdw/leads/tarefas`, dessa vez com motivo real.

No GHL, compromisso é a API de calendário (`POST /calendars/events/appointments`). A chamada
mínima da documentação não funcionou e precisou de duas correções:

1. `400 "The slot you have selected is no longer available"`: faltava
   `ignoreFreeSlotValidation: true`. É diferente do `ignoreDateRange`, que só pula a antecedência
   mínima. A documentação não deixa isso claro.
2. `422 "A team member needs to be selected"`: `assignedUserId` é opcional na documentação, mas
   obrigatório para esse tipo de calendário quando a chamada vem da API, sem sessão.

O GHL tem um calendário por corretor. Hoje `calendarId` e `assignedUserId` estão fixos para o
corretor de teste. Falta o de-para corretor do CVCRM para calendário e usuário do GHL.

### Criar tarefa e visita no CVCRM exige a API v3 (24/set)

Na API v1 nenhuma tentativa funcionou (`"Lead não existe na base de dados"`). A tela usa uma
rota interna protegida por token de sessão, o que levou à conclusão errada de que a API pública
não tinha esse endpoint. Ele existe, na **v3**, com autenticação diferente:

- Login: `POST /api/v3/auth/token`, corpo `{email, senha, painel: "gestor"}`. A resposta vem
  envelopada em `data.access_token` (a documentação mostra na raiz). `expires_in` veio como um
  valor parecido com timestamp, não como segundos.
- Criar tarefa: `POST /api/v3/comercial/leads/{idLead}/tarefas` (com `/api/` no caminho, a
  documentação omite). Corpo mínimo: `{tipoInteracao: "T" ou "V", nome, data, idresponsavel, tipoResponsavel}`.
  Um 403 "Wrong number of segments" significa que o token está `undefined`, quase sempre por
  ler `access_token` da raiz em vez de `data.access_token`.
- O workflow faz login a cada despacho, porque o token expira em horas e é a opção mais simples.
- A senha nunca vai para o git. Fica preenchida só no node `Login CVCRM v3`, dentro do n8n.
  Hoje é um login pessoal, e o ideal é uma conta de serviço.

### Eco de tarefa e visita escapava por causa de HTML escapado (24/set)

O endpoint CVDW devolve a descrição da tarefa com HTML escapado (`>` vira `&gt;`). A comparação
de eco, por texto exato, nunca batia, e uma tarefa criada a partir do GHL duplicou no CVCRM.
Correção: `decodeHtml()` em `Normalizar evento CVCRM`, aplicado ao nome da tarefa, ao título da
visita e à interação.

### Eco de nota escapava por causa de quebras de linha achatadas (25/set)

O GHL achata a formatação do corpo da nota: parágrafos separados por `\n\n` voltam separados por
um espaço. Uma anotação real de um corretor virou nota e voltou duplicada para o CVCRM, atribuída
ao usuário da integração. Correção: `integracao.normalizar_texto_eco()`, que colapsa qualquer
sequência de espaços e quebras de linha e apara as pontas, aplicada dos dois lados de toda
comparação de eco por texto.

### A checagem de eco comparava contra o destino errado (28/set)

O mesmo sintoma de 25/set voltou num lead real, o que mostrou que aquele conserto estava
incompleto. Os blocos de eco de `nota_ghl`, `interacao_cvcrm` e dos dois lados de `situacao`
comparavam contra o destino errado desde que foram escritos: um eco só faz sentido comparado com
o que a integração escreveu **na mesma plataforma de onde o evento acaba de chegar**. O bloco de
tarefa e visita sempre esteve certo.

Foi rastreado com `scripts/diagnosticar_contato.py`, que mostra a fila e o log de um contato em
ordem de tempo. Correção: os quatro blocos passaram a comparar com o destino certo, e cada
`SELECT` ganhou `campos_enviados ? 'chave'` para pegar o último envio que de fato tem aquele
campo. A anotação duplicada que já tinha sido criada continua no CVCRM, porque interação não tem
exclusão pela API.

### Compromisso do GHL dispara em toda mudança de status (28/set)

O gatilho "Status do compromisso" dispara na criação e também a cada confirmação, reagendamento
ou cancelamento. Sem tratamento, cada mudança criaria uma visita nova no CVCRM. O campo vem como
`appoinmentStatus`, com o erro de grafia do próprio GHL. Correção: `appointmentId` viaja como
`appointment_id_ghl` só para controle, e `preencher_fila_sync()` descarta o evento se esse id já
gerou uma visita. Limitação aceita: reagendar não atualiza a data da visita no CVCRM.

### Lead novo sem nome e corrida na criação do contato (29/set)

O contato novo aparecia no GHL só com o telefone. O nome passou a viajar como `nome_lead` e é
enviado como `firstName`. Aproveitou-se para mapear mais cinco campos personalizados que a conta
do GHL já tinha: id do lead, id do corretor, id da imobiliária, CPF e nome da situação.

No mesmo teste, dois eventos do CVCRM chegaram com 183 ms de diferença para o mesmo lead novo,
os dois tentaram criar o contato e o segundo levou
`400 "This location does not allow duplicated contacts"`. A resposta do erro já traz o
`contactId` certo em `meta.contactId`. O node `Detectar contato duplicado GHL` extrai esse id e
`Registrar contato recuperado GHL` grava em `depara_contato`, e o despacho seguinte usa `PUT`.

Limitação aceita: apagar um contato à mão no GHL não limpa `id_contato_ghl`. Se acontecer de
verdade, limpe esse campo em `depara_contato`.

### Lead novo ficava sem Oportunidade (30/set)

Toda mudança de situação de um lead novo falhava com `404 Cannot PUT /opportunities/`. Criar o
contato nunca criou a Oportunidade correspondente no pipeline. Correção: depois de criar o
contato, o workflow busca o estágio inicial e cria a Oportunidade (`POST /opportunities/`, status
`open`). Confirmado ao vivo, e a documentação do GHL bateu de primeira.

## Outubro de 2026

### Lead Perdido voltava para a situação 1 sozinho (7 e 8/out)

Alguém marcava o lead como Perdido no CVCRM, o GHL refletia, e 3 ou 4 minutos depois o histórico
mostrava a volta para "Aguardando Atendimento SDR", feita pelo usuário da integração.

Foi uma regressão do conserto de 28/set. Perdido no CVCRM vai para o GHL, que dispara o webhook
de volta com a etapa Perdido. Agora o eco era reconhecido e `situacao_lead` saía do evento, mas a
linha continuava na fila com `campos` vazios. O hash de um evento vazio não bate com nenhum log,
então `eh_eco` dava falso e o n8n enviava um `POST /leads` só com `idlead`, e-mail e telefone. O
CVCRM interpreta isso como reentrada e devolve o lead à situação 1.

Correção: `integracao.v_fila_para_despachar` marca `eh_eco = true` também quando não sobrou nada
para enviar. O workflow não muda, porque o node `Eh eco?` já lê essa coluna. Os leads que
voltaram à situação 1 antes da correção precisaram ser remarcados à mão.

Para confirmar o problema com dados de produção, procure envios vazios para o CVCRM:

```sql
SELECT l.id, l.criado_em, l.contato_id, l.campos_enviados, l.status,
       f.origem, f.tipo_evento
  FROM integracao.log_sync l
  JOIN integracao.fila_sync f ON f.id = l.fila_sync_id
 WHERE l.destino = 'cvcrm'
   AND l.criado_em > now() - interval '2 days'
 ORDER BY l.id DESC
 LIMIT 30;
```

Risco que a correção não cobre: qualquer outro `POST /leads` da integração sobre um lead Perdido
(por exemplo, um evento de tags do GHL com valor real) pode reativá-lo do mesmo jeito. Se
aparecer, o caminho é buscar o lead antes do POST e só despachar se ele não estiver Perdido, ou
mandar sempre o `idsituacao` atual junto.
