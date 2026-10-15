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
                │  POST (respostas dos itens) + action=check (dedup) + action=ping
                ▼
        Apps Script (Code.gs)  ──calcula escore, deduplica sob lock, carimba no servidor──▶  Google Sheets
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

> O `DASH_TOKEN` **nunca** deve ser colocado neste repositório nem nos arquivos HTML.
> Ele vive apenas nas Propriedades do script (servidor) e é digitado pela equipe no dashboard.

### Verificação rápida do endpoint (sem efeito colateral)

| Chamada GET | Resposta esperada com o `Code.gs` correto |
|---|---|
| `…/exec?action=ping` | `{"ok":true,"pong":"…"}` |
| `…/exec?action=data` (sem token) | `{"error":"unauthorized"}` |
| `…/exec?action=check&rg=000&marco=0%20dias` | `{"duplicado":false}` |
| `…/exec` (sem ação) | `{"error":"acao_desconhecida"}` |

## Data oficial da Inspeção de Saúde

Para que o marco temporal não dependa da memória do participante, o `Code.gs` usa a **data
oficial da Inspeção** sempre que o RG consta da aba `Inspecoes` da planilha de produção
(colunas `rg_cbmerj` | `data_inspecao`, uma linha por RG, data em `AAAA-MM-DD`).

- **Novas respostas:** o servidor substitui a data digitada pela oficial, recalcula
  `dias_desde_inspecao` e o marco, e grava a data digitada em `data_inspecao_informada`.
  `data_inspecao_fonte` indica a origem ("Referência (Inspeção oficial)" ou "Informada pelo
  participante"). A deduplicação e a confirmação de envio (`action=check`) usam o marco oficial.
- **Respostas já gravadas:** rodar uma vez `corrigirInspecoesExistentes()` no editor do Apps
  Script. A função copia a aba de respostas para um backup, recalcula dias e marco a partir do
  dia de registro de cada resposta e registra no log as respostas anteriores à data oficial e
  os casos de mesmo RG no mesmo marco (nada é apagado).
- RGs fora da lista seguem com a data informada no formulário.

> A lista RG → data é dado pessoal de saúde: fica **somente** na planilha. Nunca a coloque
> neste repositório (público) nem nos arquivos HTML.

## Janelas de aplicação dos marcos

O formulário calcula os dias desde a Inspeção de Saúde e classifica cada resposta:

| Marco | Janela (dias desde a Inspeção) |
|---|---|
| Inicial | 0–21 |
| 90 dias | 50–130 (±40) |
| 180 dias | 140–220 (±40) |
| 270 dias | 230–310 (±40) |
| Fora dos marcos | qualquer valor nos vãos (22–49, 131–139, 221–229) ou após 310 dias |

Respostas fora das janelas são aceitas e gravadas, marcadas como "Fora dos marcos" para
análise separada.

O dashboard reclassifica cada resposta pelo dia real (`dias_desde_inspecao`) sob as janelas
vigentes, de modo que respostas antigas, gravadas sob a regra anterior (0–15 / ±30), sejam
reavaliadas pela regra atual.

## Dashboard — comportamento

- **Acesso restrito:** exige o `DASH_TOKEN` (guardado só na sessão do navegador).
- **Evolução do PROMIS:** eixo temporal (dias). A linha passa pelos marcos oficiais 0/90/180/270
  e pelos **marcos intermediários 45/135/225 dias** (marcadores vazados), que aglutinam as
  respostas dos vãos entre janelas (22–49, 131–139 e 221–229 dias). Os intermediários existem
  **somente no gráfico**: essas respostas seguem classificadas como "Fora dos marcos" e ficam fora
  dos indicadores do topo, da evolução pareada e da comparação independente. O chip "Fora dos
  marcos" mostra, além disso, cada resposta fora das janelas como ponto avulso no dia real.
- **Evolução pareada por participante:** compara o mesmo RG entre marcos. Fluxo estatístico
  adaptado à distribuição da amostra — Shapiro-Wilk decide entre **teste t pareado** (+ d_z de
  Cohen) e **Wilcoxon de postos sinalizados** (+ r), com **correção de Holm-Bonferroni** para as
  comparações múltiplas. Implementação validada contra o `scipy`. Respostas "Fora dos marcos"
  nunca entram nesta tabela.
- **PREMs, Jornada dos exames, Medicação/tabagismo e NPS:** sempre contabilizam as respostas
  "Fora dos marcos" (indicadores de experiência/processo, não dependem da janela). Valores em
  percentual da subseção, com o absoluto no hover.
- **Triagem assistencial:** lista mascarada (•••nnn) de sinais para acompanhamento — gerencial,
  não diagnóstica.

## Segurança e LGPD

- ✅ Leitura de dados (`action=data`) exige token; nenhuma resposta é devolvida sem ele.
- ✅ Nenhum token ou segredo versionado neste repositório.
- ✅ Rascunho local do formulário não persiste identificadores no dispositivo.
- ✅ Campo de confirmação do RG não é gravado na planilha.
- ✅ Identificadores mascarados na interface do dashboard.
- ⚠️ `action=check` (deduplicação) permanece aberto por necessidade do fluxo: revela apenas se um
  RG já respondeu um marco. Mitigação futura: consulta por hash (SHA-256 + sal).
- ⚠️ O RG segue como identificador por compatibilidade. Migração para código de convite
  pseudonimizado é a evolução recomendada.

## Confiabilidade do envio

- `POST` em `no-cors` seguido de confirmação ativa (`action=check`): a tela de sucesso só aparece
  após o Apps Script localizar a resposta gravada.
- Deduplicação dupla: pré-check no cliente + verificação sob lock no servidor.
- Escalas 0–10 acessíveis por teclado; rascunho local com restauração; retry no pré-check com
  timeout de 20 s.

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
- Consulta de duplicidade por hash em vez do RG em texto.

## Desenvolvimento local

```bash
python3 -m http.server 8011
# http://127.0.0.1:8011/index.html
# http://127.0.0.1:8011/dashboard.html?demo=1
```
