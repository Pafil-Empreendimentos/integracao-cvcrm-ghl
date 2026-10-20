# Operação

Como a integração fica de pé na VM de produção e o que olhar quando os webhooks param de
chegar.

## Onde cada peça roda

| Peça | Onde | Observação |
|---|---|---|
| n8n | Docker dentro do WSL2, na VM Windows de produção | Porta 5678. O `docker-compose.yml` está no repositório `automacao-reporting`, que sobe o mesmo n8n |
| PostgreSQL | Windows da mesma VM | Banco `pafil_dw`, schema `integracao` |
| Túnel ngrok | Tarefa agendada do Windows na VM | Expõe `localhost:5678` na internet para o CVCRM e o GHL conseguirem chamar os webhooks |

A VM não tem IP público nem domínio próprio. Por isso os dois serviços (que são nuvem)
só alcançam o n8n por um túnel.

## O túnel ngrok

O processo roda como tarefa agendada, não como uma janela aberta:

- Nome da tarefa: `ngrok - integracao Pafil`
- Programa: `C:\pafil\ngrok\ngrok.exe`, argumentos `http 5678 --url "https://"`, iniciar em `C:\pafil\ngrok`
- Disparador: ao iniciar o sistema
- Configuração: executar com o usuário conectado ou não, reiniciar a cada 1 minuto em caso
  de falha, até 999 vezes

Esta é uma solução de contorno. O plano é trocar por um subdomínio da própria empresa e
desligar o ngrok.

### O domínio gratuito do ngrok muda

A documentação do ngrok diz que o domínio gratuito é estável entre reinícios. Na prática,
ele mudou três vezes numa mesma tarde, provavelmente por haver mais de um processo
`ngrok.exe` rodando ao mesmo tempo. Por isso:

- Depois de qualquer reinício do processo, confirme o domínio de novo, pelos dois lados.
- No painel local (`http://127.0.0.1:4040`, em Configuration, Tunnels, URL) o domínio atual
  aparece. Mas o painel pode dizer "online" com o túnel de fato fora do ar. A prova real é
  abrir a URL pública de um dispositivo que não esteja nessa rede. Se aparecer
  `ERR_NGROK_3200`, o túnel está offline.
- Se o domínio mudar, atualize **os dois lugares**, ou o webhook correspondente para de
  chegar sem erro visível (a plataforma tenta, desiste em silêncio):
  - CVCRM: Configurações, Integrações, webhooks, campo Endereço, `https://<dominio-atual>/webhook/cvcrm-evento`
  - GHL: no workflow de integração, ação Webhook, campo URL, `https://<dominio-atual>/webhook/ghl-evento`
    (e as URLs dos demais webhooks listados no README)

## O n8n também precisa estar de pé

O ngrok só encaminha para `localhost:5678`. Se o n8n estiver fora do ar, o túnel pode estar
perfeito e mesmo assim nada funciona.

O n8n roda em Docker dentro do WSL2. Mantenha sempre uma janela de terminal WSL aberta e
nunca digite `exit` nela: fechar o último terminal anexado derruba o WSL2 inteiro, e com ele
os containers.

Ao investigar uma falha de webhook, confira nesta ordem:

1. `docker ps` (dentro do WSL) mostra o container do n8n rodando?
2. `http://localhost:5678` no navegador da VM abre o n8n?
3. Só então investigue o túnel e o domínio.

## Verificação rápida

| Pergunta | Como checar |
|---|---|
| O túnel está de pé de verdade? | Abra a URL pública de fora da rede. `ERR_NGROK_3200` significa offline |
| Qual é o domínio atual? | `http://127.0.0.1:4040`, Configuration, Tunnels, URL |
| O ngrok está rodando? | Gerenciador de Tarefas, procure `ngrok.exe`. Deve haver só um processo |
| A tarefa agendada rodou sem erro? | Agendador de Tarefas, `ngrok - integracao Pafil`, aba Histórico |
| O n8n está de pé? | `docker ps` numa janela WSL aberta, ou `http://localhost:5678` no navegador da VM |
| Um webhook específico chegou? | `http://127.0.0.1:4040`, Traffic Inspector, filtrando por `/webhook/cvcrm-evento` ou `/webhook/ghl-evento` |
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

A conta do ngrok precisa ter como e-mail de recuperação algo que a Pafil controle, e não uma
conta pessoal. É uma dependência barata de resolver agora e cara de descobrir depois que quem
criou a conta saiu da empresa. Ainda não foi confirmado qual e-mail criou a conta atual.

O mesmo vale para o login do CVCRM usado na API v3 (hoje é pessoal). O ideal é pedir ao
CVCRM uma conta de serviço só para a integração.
