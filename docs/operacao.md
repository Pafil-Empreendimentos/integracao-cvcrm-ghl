# Operação

Como a integração fica de pé na VM de produção e o que olhar quando os webhooks param de
chegar.

## Onde cada peça roda

| Peça | Onde | Observação |
|---|---|---|
| n8n | Docker dentro do WSL2, na VM Windows de produção | Porta 5678. O `docker-compose.yml` está no repositório `automacao-reporting`, que sobe o mesmo n8n |
| PostgreSQL | Windows da mesma VM | Banco `pafil_dw`, schema `integracao` |
| Túnel da Cloudflare | Container `pafil-tunel-n8n`, no mesmo compose do n8n | Publica só `https://hooks.pafiladm.com.br/webhook/...` para o CVCRM, o GHL e o Blip chamarem |

A VM não tem IP público. Por isso os serviços (que são nuvem) só alcançam o n8n por um túnel.
O túnel liga para fora, na Cloudflare: nenhuma porta é aberta no firewall.

## O túnel da Cloudflare

Substituiu o ngrok gratuito, cujo domínio mudou três vezes numa mesma tarde. O endereço agora
é fixo, em `pafiladm.com.br` (o mesmo domínio do portal), e só os caminhos que começam com
`/webhook/` chegam ao n8n. O editor do n8n (`/`, `/rest`) não fica público, o que o ngrok não
garantia.

Peças:

- Um túnel próprio do n8n no painel da Cloudflare (Zero Trust, Networks, Tunnels), separado do túnel do portal. Se um cair, o outro segue de pé.
- A rota publicada nesse túnel: `hooks.pafiladm.com.br`, caminho `^/webhook/`, serviço HTTP `n8n:5678`. Tudo que não casar com o caminho cai na regra final de 404.
- O serviço `tunel` no `docker-compose.yml` do `automacao-reporting`, com o token do túnel em `TUNNEL_TOKEN_N8N` no `.env` da VM.
- Sem Access nesse endereço, de propósito: CVCRM, GHL e Blip não conseguem fazer login. A proteção é o caminho restrito e, onde o serviço permite, o segredo no cabeçalho (o webhook do Blip usa Header Auth).

Endereços dos webhooks (todos em `https://hooks.pafiladm.com.br`):

| Quem chama | Caminho |
|---|---|
| CVCRM (Novo lead, Alteração de situação, Associar atendente, Nova tarefa) | `/webhook/cvcrm-evento` |
| GHL, workflow com ação Webhook | `/webhook/ghl-evento` |
| GHL, mudança de estágio da Oportunidade | `/webhook/ghl-oportunidade-evento` |
| GHL, nota adicionada | `/webhook/ghl-nota-evento` |
| GHL, tarefa adicionada | `/webhook/ghl-tarefa-evento` |
| GHL, status do compromisso | `/webhook/ghl-compromisso-evento` |
| Blip (ação do Builder, repositório `integracao-cvcrm-blip`) | `/webhook/blip-lead` |

O endereço de teste do n8n (`/webhook-test/...`) não é público. Para testar um workflow antes de
ativá-lo, chame `http://localhost:5678/webhook-test/<caminho>` de dentro da VM.

### Trocar do ngrok para a Cloudflare (feito uma vez)

A ordem importa: o ngrok só é desligado depois que os dois lados novos provarem que chegam.

1. Criar o túnel e a rota no painel da Cloudflare (você, no navegador).
2. Colocar o token em `TUNNEL_TOKEN_N8N` no `.env` do `automacao-reporting` (você, na VM) e subir o compose.
3. Testar de fora da rede: `https://hooks.pafiladm.com.br/` deve dar 404 (não a tela do n8n).
4. Trocar o endereço no CVCRM (Configurações, Integrações, webhooks, campo Endereço) e no GHL (ação Webhook de cada workflow), usando a tabela acima.
5. Conferir no n8n (Executions) que chegaram eventos novos pelos dois.
6. Só então desativar a tarefa agendada `ngrok - integracao Pafil` e encerrar o `ngrok.exe`.

## O n8n também precisa estar de pé

O túnel só encaminha para o n8n. Se o n8n estiver fora do ar, o túnel pode estar perfeito e
mesmo assim nada funciona.

O n8n roda em Docker dentro do WSL2. Mantenha sempre uma janela de terminal WSL aberta e
nunca digite `exit` nela: fechar o último terminal anexado derruba o WSL2 inteiro, e com ele
os containers (inclusive o túnel).

Ao investigar uma falha de webhook, confira nesta ordem:

1. `docker ps` (dentro do WSL) mostra `pafil-n8n` e `pafil-tunel-n8n` rodando?
2. `docker logs --tail 20 pafil-tunel-n8n` mostra a conexão registrada na Cloudflare?
3. `http://localhost:5678` no navegador da VM abre o n8n?
4. Só então investigue a rota no painel da Cloudflare.

## Verificação rápida

| Pergunta | Como checar |
|---|---|
| O túnel está de pé de verdade? | Abra `https://hooks.pafiladm.com.br/` de um dispositivo fora da rede (celular no 4G). O esperado é 404. Erro 1033 ou 502 da Cloudflare significa túnel ou n8n fora do ar |
| O conector está conectado? | `docker logs --tail 20 pafil-tunel-n8n`, procure `Registered tunnel connection`. No painel, Tunnels mostra o túnel como Healthy |
| O n8n está de pé? | `docker ps` numa janela WSL aberta, ou `http://localhost:5678` no navegador da VM |
| Um webhook específico chegou? | n8n, Executions, filtrando pelo workflow |
| A fila está andando? | `SELECT id, origem, status, tentativas, erro_msg FROM integracao.fila_sync ORDER BY id DESC LIMIT 10;` |
| Um node HTTP parece certo mas falha sempre | Veja a aba Settings do node (On Error deve ser "Continue using error output") e se as duas saídas, sucesso e erro, chegam em `Rotular despacho`. Recriar um node colando JSON costuma perder as duas coisas |
| O que aconteceu com um contato específico? | `python scripts/diagnosticar_contato.py --idlead <id>` |

## Reconciliação: desligada de propósito

O workflow tem um segundo caminho, independente dos webhooks: o `Gatilho reconciliacao`, que
roda a cada 6 horas e varre `bronze.leads` atrás de leads que a integração ainda não conhece.

Está **desativado** desde 18/set/2026. Quando foi ligado, começou a puxar leads reais de
produção para a fila com a integração ainda em validação. Antes de reativar, decida
conscientemente se a integração já está pronta para tocar em leads reais dos dois lados e
acompanhe de perto as primeiras execuções.

Como proteção extra, o escopo de empreendimento também está dentro da view
`integracao.v_reconciliacao_cvcrm`. Mesmo que alguém religue o gatilho por engano, o dano
fica restrito ao piloto.

## Pendência de governança

O login do CVCRM usado na API v3 é pessoal. O ideal é pedir ao CVCRM uma conta de serviço só
para a integração.
