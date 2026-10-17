# Programa Coração de Herói — PROMIS Global Health / PREMs (produção)

Coleta longitudinal de desfechos e experiência relatados por bombeiros militares do CBMERJ,
com dashboard gerencial vinculado. Modelo VBHC · 1ª Policlínica – Campinho.

Este é o repositório de **produção**. O desenvolvimento e a homologação acontecem no
repositório `coracao-de-heroi-dev`; as versões validadas no piloto são promovidas para cá.

- **Formulário:** https://ricardodesouzacarvalho.github.io/coracao-de-heroi/
- **Dashboard:** https://ricardodesouzacarvalho.github.io/coracao-de-heroi/dashboard.html

## O que o sistema mede

| Bloco | Conteúdo |
|---|---|
| PROMIS Global Health | 10 itens de saúde percebida (física e mental), convertidos em escore bruto e T-score. |
| PREMs | Comunicação, decisão compartilhada, coordenação, experiência com os profissionais, jornada dos exames e equipe de navegação. |
| Contexto de medicação | Adesão nos últimos 7 dias e principal barreira ao tratamento. |
| Tabagismo | Situação, tempo de cessação (ex-fumantes), abordagem e conduta de cessação. |
| Recomendação | NPS (0–10) e dois campos abertos. |

## Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `index.html` | Formulário público. Grava no Google Sheets via Apps Script. |
| `dashboard.html` | Dashboard gerencial com **acesso restrito por chave**. Modo demonstração: `dashboard.html?demo=1`. |
| `apps-script/Code.gs` | Backend do Google Apps Script (referência versionada do que roda na planilha de produção). |
| `relatorios/` | Relatórios de análise técnica (v1 e v2) que orientaram as correções. |
| `.gitignore` | Ignora arquivos do macOS. |

## Arquitetura

```
Bombeiro → index.html (GitHub Pages)
                │  POST (respostas + id_envio) + action=check (confirmação) + action=ping
                ▼
        Apps Script (Code.gs)  ──calcula escore, aplica data oficial, carimba no servidor──▶  Google Sheets
                ▲
                │  action=data (exige DASH_TOKEN) → devolve {ok:true, values:[...]}
        dashboard.html (GitHub Pages)  ──lê e agrega no navegador──▶  gráficos e triagem
```

O **escore PROMIS é calculado exclusivamente no servidor** (`Code.gs`): recodificação da dor,
somas brutas e T-score pela tabela oficial (HealthMeasures) são computados na gravação,
sobrescrevendo qualquer valor vindo do cliente. Formulário e dashboard não contêm tabelas de
escore — **fonte única, sem risco de divergência**.

## Configuração do backend de produção

O `Code.gs` deste repositório é a referência do que deve estar implantado no Apps Script
**vinculado à planilha de produção**. Para (re)implantar:

1. Planilha de produção → **Extensões → Apps Script**; cole o conteúdo de `apps-script/Code.gs`.
2. **Configurações do projeto → Propriedades do script:** crie `DASH_TOKEN` = chave longa e
   aleatória (é a chave digitada pela equipe na tela "Acesso restrito" do dashboard).
3. **Implantar → Gerenciar implantações → Editar (lápis) → Nova versão** — mantém a mesma URL
   já referenciada no `SCRIPT_URL` dos arquivos.
4. Ajuste `SHEET_NAME` no `Code.gs` se a aba não se chamar `Respostas`.
5. Crie a aba `Inspecoes` com as datas oficiais (ver "Data oficial da Inspeção de Saúde").
6. Crie a aba `Contatos` (cabeçalho `RG | Nome | Posto/Graduação | Telefone_WhatsApp |
   Link_WhatsApp | E-mail | Observação`) para o controle de envio. As abas `Envios` e
   `Acompanhamento` são criadas automaticamente no primeiro registro feito pelo dashboard.

> O `DASH_TOKEN` **nunca** deve ser colocado neste repositório nem nos arquivos HTML.
> Ele vive apenas nas Propriedades do script (servidor) e é digitado pela equipe no dashboard.

### Verificação rápida do endpoint (sem efeito colateral)

| Chamada GET | Resposta esperada com o `Code.gs` correto |
|---|---|
| `…/exec?action=ping` | `{"ok":true,"pong":"…"}` |
| `…/exec?action=data` (sem token) | `{"error":"unauthorized"}` |
| `…/exec?action=check&envio=teste-00000` | `{"gravado":false}` |
| `…/exec` (sem ação) | `{"error":"acao_desconhecida"}` |

## Data oficial da Inspeção de Saúde

Para que o marco temporal não dependa da memória do participante, o `Code.gs` usa a **data
oficial da Inspeção** sempre que o RG consta da aba `Inspecoes` da planilha de produção
(colunas `rg_cbmerj` | `data_inspecao`, uma linha por RG, data em `AAAA-MM-DD`).

- **Novas respostas:** o servidor substitui a data digitada pela oficial, recalcula
  `dias_desde_inspecao` e o marco, e grava a data digitada em `data_inspecao_informada`.
  `data_inspecao_fonte` indica a origem ("Referência (Inspeção oficial)" ou "Informada pelo
  participante").
- **Respostas já gravadas:** rodar uma vez `corrigirInspecoesExistentes()` no editor do Apps
  Script. A função copia a aba de respostas para um backup, recalcula dias e marco a partir do
  dia de registro de cada resposta e registra no log as respostas anteriores à data oficial e
  os casos de mesmo RG com mais de uma resposta no mesmo marco (nada é apagado).
- RGs fora da lista seguem com a data informada no formulário e vão para a auditoria (ver
  "Programa fechado nos RGs da lista").

> A lista RG → data é dado pessoal de saúde: fica **somente** na planilha. Nunca a coloque
> neste repositório (público) nem nos arquivos HTML.

## Programa fechado nos RGs da lista

O programa é fechado nos RGs da aba `Inspecoes`. Um RG que não consta dela é tratado como erro
de digitação ou participante não incluído:

- a resposta é gravada, com `rg_no_programa` = `Não`;
- o formulário mostra ao participante que o RG informado não está incluído no programa e pede
  contato pelo WhatsApp **(21) 99278-9615** (link `wa.me` com mensagem pronta) para verificar a
  divergência;
- o dashboard exclui essas respostas de **todas** as análises e as lista na seção
  **Auditoria · RGs fora do programa** (identificador mascarado, data e hora de registro, data
  informada, marco e situação do contato).

Resolvida a divergência, corrija o RG na aba `Respostas` (ou inclua o RG na aba `Inspecoes`) e
rode `corrigirInspecoesExistentes()` de novo. Sem a aba `Inspecoes`, nenhum RG é marcado.

O aviso aparece **depois** do envio, na tela de confirmação: uma consulta prévia "este RG está no
programa?" revelaria a qualquer pessoa quem participa de um programa de saúde cardiovascular.

## Controle de envio do formulário

Seção do dashboard que lista os militares do programa (aba `Inspecoes`) cuja janela de
resposta está aberta, pela data oficial da Inspeção e pelas janelas de análise (Inicial 0–60,
90 dias 61–130, 180 dias 140–220, 270 dias 230–310), e os que entram na janela em até 14 dias.

- **Mensagem individualizada** (posto, nome, marco e link do formulário; cerca de 7 minutos e
  "o quanto antes", sem prazo nem data da Inspeção), aberta já preenchida no **WhatsApp** (`wa.me`) ou no **e-mail** (`mailto:`).
  Canal sugerido: e-mail quando houver e-mail válido e exclusivo; senão, WhatsApp.
- **Registro do envio:** o clique grava data, RG, marco e canal na aba `Envios`
  (`action=envio`, exige o token).
- **Baixa automática:** quando a resposta do marco chega, o militar passa para "Respondido".
- Alertas de cadastro: sem contato, sem posto, e-mail repetido em outro militar, observações.
- **Coluna `Observação` da aba `Contatos` (regra de envio):**
  - `Desistência` → **envio suspenso** por qualquer canal (nem e-mail automático, nem botões de
    WhatsApp/e-mail no dashboard). O militar aparece só no filtro "Suspensos (desistência)".
    Se ele já tiver respondido o marco, continua contado como respondido.
  - `Deixou de responder` → **continua recebendo**, com mensagem de **resgate** ("Sentimos a sua
    falta…"), pelo e-mail automático e pelo dashboard.
  - Vazio ou qualquer outro texto → regra normal (o texto aparece como alerta no dashboard).
  A leitura ignora maiúsculas e acentos e vale pelo início do texto (`desist…`, `deixou…`).

### E-mail automático (a cada 3 dias)

`enviarEmailsAutomaticos()` (no `Code.gs`) roda todo dia às 8h (Brasília) e envia, para quem
está na janela, não respondeu o marco e tem e-mail válido e exclusivo na aba `Contatos`: o
**convite** e até **2 lembretes** por marco, com **intervalo de 3 dias** entre contatos ao mesmo
militar (por qualquer canal, inclusive WhatsApp registrado no dashboard). A mensagem enfatiza a
importância da resposta (primeira mensagem do marco: "Está aberto o formulário de acompanhamento
do marco …"; depois de envio registrado para o mesmo marco: "Estamos enviando mais uma vez…";
cerca de 7 minutos, link). Quem está como `Desistência` na coluna `Observação` da aba
`Contatos` não recebe; quem está como `Deixou de responder` recebe a mensagem de resgate.
Para de enviar quando a resposta chega, respeita a cota diária do Gmail
e registra cada envio na aba `Envios` (canal `email-auto`).

- Remetente: **Programa Coração de Herói** · respostas para `coracaodeheroi.1polcampinho@gmail.com`.
  Se o agendamento for criado na conta do programa, sai dela; se for criado em outra conta, sai
  como o programa quando o endereço estiver configurado em "Enviar e-mail como", ou pela
  própria conta com o nome do programa. O modo usado aparece no log de cada execução.
- Ativar: rodar uma vez `instalarEnvioAutomatico()` pela conta que vai enviar (pede autorização
  do Gmail). Conferir sem enviar: `previaEmailsAutomaticos()`. Ver de qual endereço o e-mail sai:
  `testeRemetente()` (envia um e-mail de teste só para a conta que executa e registra o modo no log). Cancelar: `desinstalarEnvioAutomatico()`.
- Ajustes no `Code.gs`: `MAX_LEMBRETES`, `INTERVALO_MIN_DIAS`; no piloto, propriedade do script
  `FORM_URL` com o link do formulário do piloto.

## Respostas repetidas (sem bloqueio)

Desde 15/10/2026 não há bloqueio de respostas repetidas: o mesmo RG pode responder mais de uma
vez o mesmo marco, e todas as respostas são gravadas. No dashboard, as estatísticas (KPIs,
gráfico, PREMs, evolução pareada, comparação independente e participação) usam a **primeira
resposta de cada RG por marco**, pela ordem de registro; as seguintes aparecem contadas no resumo
abaixo do gráfico e continuam na triagem assistencial, para que nenhum sinal novo fique oculto.

## Janelas de aplicação dos marcos

O formulário (e o `Code.gs`) calcula os dias desde a Inspeção de Saúde e grava o marco de
**coleta** de cada resposta:

| Marco | Janela (dias desde a Inspeção) |
|---|---|
| Inicial | 0–21 |
| 90 dias | 50–130 (±40) |
| 180 dias | 140–220 (±40) |
| 270 dias | 230–310 (±40) |
| Fora dos marcos | qualquer valor nos vãos (22–49, 131–139, 221–229) ou após 310 dias |

Respostas fora das janelas são aceitas e gravadas, marcadas como "Fora dos marcos" para
análise separada.

### Janelas de análise do dashboard

O dashboard reclassifica cada resposta pelo dia real (`dias_desde_inspecao`), independentemente
do marco gravado. Desde 15/10/2026 o marco Inicial da **análise** foi ampliado para incluir
todas as primeiras avaliações:

| Marco (análise) | Janela (dias desde a Inspeção) |
|---|---|
| Inicial | 0–60 |
| 90 dias | 61–130 |
| 180 dias | 140–220 |
| 270 dias | 230–310 |
| Fora dos marcos | 131–139, 221–229 ou após 310 dias |

## Dashboard — comportamento

- **Acesso restrito:** exige o `DASH_TOKEN` (guardado só na sessão do navegador).
- **Evolução do PROMIS:** eixo temporal (dias). A linha passa pelos marcos 0/90/180/270; o chip
  "Fora dos marcos" mostra cada resposta fora das janelas como ponto avulso no dia real, e o
  resumo abaixo do gráfico conta essas respostas e as repetidas.
- **Evolução pareada por participante:** compara o mesmo RG entre todos os pares de marcos —
  Inicial → 90, 180 e 270 dias; 90 → 180 e 270 dias; 180 → 270 dias. Fluxo estatístico
  adaptado à distribuição da amostra — Shapiro-Wilk decide entre **teste t pareado** (+ d_z de
  Cohen) e **Wilcoxon de postos sinalizados** (+ r), com **correção de Holm-Bonferroni** sobre
  todas as comparações da tabela (até 12: 6 pares × 2 domínios). Implementação validada contra o `scipy`. Respostas "Fora dos marcos"
  nunca entram nesta tabela.
- **PREMs, Jornada dos exames, Medicação/tabagismo e NPS:** sempre contabilizam as respostas
  "Fora dos marcos" (indicadores de experiência/processo, não dependem da janela). Valores em
  percentual da subseção, com o absoluto no hover.
- **Sinais para acompanhamento assistencial:** gerencial, não diagnóstica. Cada caso (RG + marco)
  com escore ≥ 2 mostra o **nome** do militar (aba `Contatos`; a tela só abre com a chave), os
  sinais e a prioridade, e traz:
  - **Mensagem** pronta no WhatsApp ou no e-mail (e-mail só quando é exclusivo do militar), com
    abertura empática que reconhece que precisamos melhorar, um parágrafo com pergunta aberta
    para cada problema (saúde física, bem-estar emocional, medicamentos, exames pendentes,
    acesso a exames, tabagismo) e o convite para buscarmos a solução juntos. Militar com
    `Desistência` na coluna `Observação` não recebe mensagem.
  - **Situação** do caso: `Não resolvido` (padrão), `Parcialmente resolvido` ou `Resolvido`.
  - **Filtros próprios da seção** (independentes dos chips de marco do topo): situação (em aberto,
    cada situação, todas), prioridade (todas, alta, moderada), marco (todos, Inicial, 90, 180,
    270, fora dos marcos) e quantidade de casos (10, 25, 50 ou 100), por ordem de prioridade.
  - **Prévia da mensagem** ao passar o cursor (ou focar pelo teclado) sobre WhatsApp ou E-mail.
  - A situação e o último contato (data e canal,
    gravado ao clicar no botão da mensagem) ficam na aba **`Acompanhamento`** da planilha
    (`action=acompanhamento`, exige o token), criada automaticamente na primeira alteração.
  - Para sinais de saúde emocional, a mensagem é só a abertura: prefira também ligar ou agendar.

- **Rastreio de câncer de pulmão:** lista os fumantes e ex-fumantes pelas respostas ao formulário
  (com pré-triagem pelo tempo sem fumar informado) e, como os demais, mostra nome, mensagem e
  situação. A mensagem (WhatsApp/e-mail, com prévia ao passar o cursor) explica a TC de tórax de
  baixa dose e pergunta idade, cigarros por dia, anos fumando e, para ex-fumantes, há quantos
  anos parou; para fumantes, oferece também ajuda para parar. Em **Registrar**, a equipe lança as
  respostas: o painel calcula os **maços-ano** e a **indicação** pelo critério USPSTF 2021 (50 a 80
  anos, ≥ 20 maços-ano, fumante atual ou que parou há no máximo 15 anos), e registra data da TC,
  **Lung-RADS** e a conduta sugerida (v2022: 1–2 → nova TC em 12 meses; 3 → 6 meses; 4A → 3
  meses; 4B/4X → avaliação especializada; decisão médica). Situações: A contatar → Aguardando
  resposta → Elegível — agendar TC → TC agendada → TC realizada, ou Não elegível / Recusou. Tudo
  fica na aba **`Rastreio`** da planilha (`action=rastreio`, exige o token), criada automaticamente.
- **Visual:** cabeçalho com o brasão estilizado do CBMERJ (`assets/brasao-cbmerj.webp`, também
  ícone da aba) e paleta derivada do brasão: vermelho do anel, ouro da estrela e aço do campo.
  Os resumos do acompanhamento assistencial e do controle de envio são **gráficos de rosca** com
  o KPI no centro (**% resolvidos** e **% respondidos na janela**, este sobre quem está na janela
  agora; quem abre em breve e os suspensos ficam fora do % e aparecem abaixo da legenda).
  Cores de estado validadas para daltonismo: verde = concluído, ouro = em andamento, vermelho =
  em aberto. Clicar no segmento ou na legenda aplica o filtro da lista.
- **PREMs (radar):** cada marco é um chip com o % geral; clicar mostra ou oculta a camada, passar
  o cursor destaca a camada, e com um só marco ativo os valores de cada dimensão aparecem no
  gráfico. Cores dos marcos validadas (azul, ouro, vermelho, violeta para 270 dias).
- **Participação por marco:** rosca com as respostas por marco (Inicial, 90, 180, 270 e fora dos
  marcos) e o % nas janelas no centro, sobre todas as respostas (independente do filtro).
  - **Tempo do formulário:** média, entre os militares, da parte já percorrida dos 300 dias de
    acompanhamento de cada um (a partir da data oficial da Inspeção), com o término previsto
    (última Inspeção + 300 dias), dias restantes e o % do calendário do programa como referência.
  - **Respostas esperadas até hoje:** respostas recebidas ÷ esperadas, contando por militar cada
    marco cuja janela de análise já abriu (inclusive as abertas), com o detalhe por marco e o %
    só das janelas encerradas. Ficam fora os desistentes e os marcos cuja janela fechou antes do
    início da coleta (data da primeira resposta ao formulário).
  Logo abaixo vem o **Controle de envio do formulário**.
- **Comparação independente entre marcos:** fica oculta; o chip "Mostrar" a exibe.
- **Recomendação do programa:** rosca com o NPS no centro (promotores, neutros, detratores) e
  histograma das notas de 0 a 10. O cartão do topo usa a mesma base (inclui fora dos marcos).

## Segurança e LGPD

- ✅ Leitura de dados (`action=data`) exige token; nenhuma resposta é devolvida sem ele.
- ✅ Nenhum token ou segredo versionado neste repositório.
- ✅ Rascunho local do formulário não persiste identificadores no dispositivo.
- ✅ Campo de confirmação do RG não é gravado na planilha.
- ✅ Identificadores mascarados nos gráficos e na auditoria. Nome e RG aparecem apenas no controle
  de envio e no acompanhamento assistencial, que dependem da chave (`action=data`).
- ✅ A confirmação de envio usa um identificador aleatório do envio (`action=check&envio=…`),
  sem RG.
- ⚠️ A consulta legada `action=check&rg=…&marco=…` permanece apenas para formulários antigos
  em cache: revela se um RG já respondeu um marco. Pode ser removida após a transição.
- ⚠️ O RG segue como identificador por compatibilidade. Migração para código de convite
  pseudonimizado é a evolução recomendada.

## Confiabilidade do envio

- `POST` em `no-cors` seguido de confirmação ativa pelo id do envio (`action=check&envio=…`): a
  tela de sucesso só aparece após o Apps Script localizar a resposta gravada, e mostra o marco
  registrado pelo servidor (o oficial, quando o RG tem data de referência). Se o Apps Script
  ainda for a versão anterior (sem confirmação por id), o formulário usa a consulta legada.
- Escalas 0–10 acessíveis por teclado; rascunho local com restauração.

## Diferença para o repositório de desenvolvimento

Os arquivos aqui são idênticos aos validados no dev, exceto:

- `index.html`: cabeçalho sem o selo "Piloto / homologação", `<title>` de produção e
  `versao_formulario` sem o sufixo `DEV`;
- `index.html` e `dashboard.html`: `SCRIPT_URL` apontando para o Apps Script/planilha de
  **produção** (não o do piloto).

## Pendências conhecidas (para versões futuras)

- Migrar o RG para código de convite pseudonimizado.
- Validar metodologicamente a redação adaptada dos itens 1, 2 e 6 do PROMIS antes de comparações
  externas definitivas.
- Remover a consulta legada `action=check&rg=…&marco=…` após a transição.

## Desenvolvimento local

```bash
python3 -m http.server 8011
# http://127.0.0.1:8011/index.html
# http://127.0.0.1:8011/dashboard.html?demo=1
```
