# Chamado de suporte: webhook "Nova interação" (gatilho -1) não dispara

## Resumo

O webhook de Lead configurado com o gatilho -1 ("Nova interação") nunca
chama a URL cadastrada quando uma interação (anotação) nova é criada num
lead, mesmo com o webhook ativo e funcionando corretamente para outros
gatilhos.

## Como reproduzir

1. Cadastrar um webhook de Lead com gatilho -1 (Nova interação), apontando
   para qualquer URL que receba requisições (usamos um endpoint próprio
   para testar).
2. Criar uma anotação/interação nova em qualquer lead vinculado a esse
   webhook (pela tela "Gestão de Leads", botão Anotação).
3. Verificar se a URL recebeu alguma requisição.

## O que esperávamos

Uma requisição POST chegando na URL cadastrada, assim que a interação é
salva, do mesmo jeito que acontece com os outros gatilhos.

## O que aconteceu

Nenhuma requisição chega. Testamos isso repetidas vezes, em leads
diferentes, ao longo de várias semanas, criando dezenas de anotações de
teste (inclusive interações do tipo WhatsApp, geradas automaticamente pelo
sistema) e nenhuma delas disparou o webhook.

Pra confirmar que o problema é específico desse gatilho e não da nossa
configuração, testamos o mesmo webhook (mesma URL, mesmo lead) com outros
gatilhos do Lead, como mudança de situação e Associar Atendente, e esses
dispararam normalmente, sem nenhum problema.

Também confirmamos pela própria API (`GET
/api/v1/integracoes/webhooks/gatilhos`) que o código -1 corresponde
mesmo a "Nova interação" e que não existe nenhum outro gatilho
equivalente disponível para esse evento.

## O que pedimos

Que o time técnico confirme se o gatilho -1 (Nova interação) está
funcionando na plataforma hoje e, se não estiver, que seja corrigido.
Ficamos à disposição para reproduzir o teste em conjunto ou passar mais
detalhes do webhook cadastrado, se ajudar na investigação.
