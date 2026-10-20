# Integração CVCRM e GoHighLevel

Sincroniza, nos dois sentidos, o que acontece com um lead no CVCRM (o CRM da Pafil)
e no GoHighLevel (GHL, a ferramenta de relacionamento e campanhas). Quando algo muda
de um lado, a mudança aparece no outro sem ninguém digitar duas vezes.

O projeto roda sobre o [n8n](https://n8n.io) e usa o mesmo PostgreSQL da plataforma de
dados ([data-hub-pafil](https://github.com/arturvillasboas/data-hub-pafil)), em um
schema próprio chamado `integracao`.

## O que sincroniza

| Do CVCRM para o GHL | Do GHL para o CVCRM |
|---|---|
| Lead novo vira contato (com nome, telefone e e-mail) e ganha uma Oportunidade | Mudança de estágio da Oportunidade vira mudança de situação do lead |
| Situação do lead vira estágio da Oportunidade | Tags |
| Interações (anotações) viram notas do contato | Notas do contato viram interações (anotações) |
| Tarefas viram Tasks | Tasks viram tarefas |
| Visitas viram Compromissos no calendário | Compromissos viram visitas |
| Mídia, corretor, empreendimento, id do lead, id da imobiliária e CPF viram campos personalizados | |

O dono de cada campo (quem tem a palavra final) fica na tabela `integracao.dono_campo`.
Mudar uma regra desse tipo é um `UPDATE` no banco, sem mexer no workflow.

## Como funciona

```
CVCRM ──webhook──┐
                 ├──► n8n ──► integracao.fila_sync ──► despacho (a cada 2 min) ──┬──► API do CVCRM
GHL ───webhook───┘                (Postgres)                                    └──► API do GHL
```

1. CVCRM e GHL avisam o n8n por webhook sempre que algo muda.
2. O n8n normaliza o evento e grava na fila (`integracao.fila_sync`).
3. A cada 2 minutos um gatilho lê a fila já tratada pela view `integracao.v_fila_para_despachar`.
   É no banco que ficam as decisões: quem é dono do campo, se o evento é só um eco da
   própria integração e o que realmente precisa ser enviado.
4. O n8n envia para a plataforma de destino e registra o resultado em `integracao.log_sync`.

Toda a regra de negócio mora em SQL (`sql/integracao.sql`). O n8n só recebe, envia e
repete. Isso deixa o workflow simples de ler e permite migrar a lógica para outro
orquestrador no futuro.

### Evitando o loop de eco

Se uma nota criada no GHL vira interação no CVCRM, o CVCRM avisa que existe uma
interação nova, e sem cuidado ela voltaria como nota para o GHL, para sempre. A
integração compara o que está chegando com o que ela mesma acabou de escrever naquela
plataforma e descarta o eco. Foi a parte que mais deu trabalho, e os casos em que
falhou estão contados em [`docs/licoes-aprendidas.md`](docs/licoes-aprendidas.md).

## Estrutura do repositório

```
.
├── sql/integracao.sql            # schema integracao: tabelas, funções e views (idempotente)
├── workflow/workflow_n8n.json    # workflow do n8n, sem credenciais, para importar
├── scripts/
│   ├── aplicar_integracao.py     # aplica o SQL no banco e confere os objetos
│   ├── diagnosticar_contato.py   # mostra o caminho de um contato na fila e no log
│   └── exportar_leads_ghl.py     # gera o CSV de leads para importar no GHL
└── docs/
    ├── operacao.md               # túnel, n8n, verificações do dia a dia
    ├── licoes-aprendidas.md      # incidentes e comportamentos reais das APIs
    └── chamado-cvcrm-nova-interacao.md
```

## Pré-requisitos

- O banco PostgreSQL da plataforma de dados já em pé. O schema `integracao` fica no mesmo
  banco e lê `bronze.leads`. O script de exportação de leads também usa `gold.fato_leads`.
- Um n8n com acesso ao Postgres e uma URL pública para receber os webhooks (hoje, um túnel
  ngrok na VM, descrito em [`docs/operacao.md`](docs/operacao.md)).
- Acesso de administrador no CVCRM (webhooks e API) e no GHL (token de integração).
- Python 3.10 ou mais novo para os scripts.

## Instalação

1. Instale as dependências e configure o acesso ao banco:

   ```bash
   python -m venv .venv
   .venv\Scripts\activate          # no Linux/macOS: source .venv/bin/activate
   pip install -r requirements.txt
   copy .env.example .env          # preencha PG_* com os dados do banco
   ```

2. Crie o schema `integracao` (pode rodar de novo quando quiser, é idempotente):

   ```bash
   python scripts/aplicar_integracao.py
   ```

3. No n8n, importe `workflow/workflow_n8n.json` (Workflows, Import from File). O arquivo
   vem sem segredos, então crie e associe estas credenciais antes de ativar:

   | Para quê | Tipo no n8n | Observação |
   |---|---|---|
   | Postgres (nodes `Gravar fila`, `Fila para despachar` e os demais) | Postgres | Banco da plataforma de dados |
   | Chamadas à API v1 do CVCRM | Header Auth | O workflow espera o header `email` no node e o `token` na credencial |
   | Chamadas à API do GHL | Bearer Auth | Token de integração privada do GHL. O node já manda `Version: v3` |
   | Criação de tarefa e visita no CVCRM | (sem credencial) | Usa a API v3, que exige login. Preencha e-mail e senha direto no node `Login CVCRM v3` e nunca no git |

4. Cadastre os webhooks nas duas plataformas, apontando para a URL pública do n8n:

   | Plataforma | Caminho do webhook |
   |---|---|
   | CVCRM (Novo lead, Alteração de situação, Associar atendente, Nova tarefa) | `/webhook/cvcrm-evento` |
   | GHL (workflow com ação Webhook) | `/webhook/ghl-evento` |
   | GHL, mudança de estágio da Oportunidade | `/webhook/ghl-oportunidade-evento` |
   | GHL, nota adicionada | `/webhook/ghl-nota-evento` |
   | GHL, tarefa adicionada | `/webhook/ghl-tarefa-evento` |
   | GHL, status do compromisso | `/webhook/ghl-compromisso-evento` |

   Restrinja os webhooks do CVCRM aos empreendimentos do piloto. Webhook com o campo
   "Empreendimentos" em branco vale para todos, e foi assim que 90 contatos indevidos
   foram criados no GHL em setembro (veja as lições aprendidas).

5. Ative os gatilhos e acompanhe a fila:

   ```sql
   SELECT id, origem, status, tentativas, erro_msg
     FROM integracao.fila_sync ORDER BY id DESC LIMIT 10;
   ```

## No dia a dia

- Para entender por que um contato se comportou de um jeito, rode
  `python scripts/diagnosticar_contato.py --idlead 12345` (ou `--telefone +5511999998888`).
  Ele imprime a fila e o log desse contato em ordem de tempo.
- Para importar uma amostra de leads no GHL, use `scripts/exportar_leads_ghl.py` (veja o
  cabeçalho do arquivo para os parâmetros).
- Se os webhooks pararam de chegar, o motivo mais comum é o túnel ou o n8n fora do ar. A
  checagem passo a passo está em [`docs/operacao.md`](docs/operacao.md).

## Situação atual

A integração está em piloto, restrita a um empreendimento. Lead novo, situação,
interações, tarefas e visitas já funcionam nos dois sentidos. Está pendente:

- liberar para os demais empreendimentos (hoje o escopo está travado na view
  `integracao.v_reconciliacao_cvcrm` e nos filtros dos webhooks);
- mapear corretor do CVCRM para calendário e usuário do GHL (hoje a visita usa um
  calendário de teste);
- reativar a reconciliação a cada 6 horas, que está desligada de propósito;
- trocar o login pessoal usado na API v3 do CVCRM por uma conta de serviço;
- substituir o túnel ngrok por um domínio da própria empresa.

O acompanhamento fica nas issues deste repositório.
