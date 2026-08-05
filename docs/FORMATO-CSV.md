# Formato do CSV

Planilha codificada em **UTF-8**. A primeira linha contém os nomes das colunas.

## Colunas obrigatórias

| Coluna | Descrição |
|---|---|
| `TITULO_NACIONAL` | Título em português do Brasil. |
| `TITULO_ORIGINAL` | Título original do conteúdo. |
| `SINOPSE` | Sinopse curta. |
| `CLASSIFICACAO` | Classificação indicativa. |
| `DURACAO` | Duração em minutos; convertida para `HH:MM:SS`. |
| `BILLING_ID` | Identificador de billing. |
| `ANO` | Ano de lançamento. |
| `ELENCO` | Atores principais, separados por vírgula. |
| `DIRETOR` | Nome do diretor. |
| `GENERO_1` | Gênero principal. |
| `PAIS` | País de origem. |
| `IDIOMA` | Idioma principal do áudio. |
| `LEGENDADO` | `Sim`, `Leg`, `Legendado` → legendado. `NAC`, `Nacional`, `Não` → áudio nacional. |
| `DVB` | `Sim` inclui metadados explícitos de idioma (`en` / `pt`). |

## Colunas opcionais

| Coluna | Descrição |
|---|---|
| `EXTRADATA` | Dado adicional, renderizado apenas quando preenchido. |
| `GENERO_2` | Gênero adicional. |
| `GENERO_3` | Gênero adicional. |
| `EPISODE_ID` | ID numérico do episódio (apenas dígitos). |
| `EPISODE_NAME` | Título do episódio. |

## Regras de validação

- Colunas obrigatórias não podem estar vazias.
- `DURACAO` e `ANO` aceitam apenas valores numéricos válidos.
- `EPISODE_ID` aceita somente caracteres numéricos.
- `EPISODE_ID` e `EPISODE_NAME` devem ser preenchidos em conjunto: se um existir, o outro é obrigatório.
- Linhas duplicadas são sinalizadas.
- Qualquer linha com problema abre um diálogo detalhado indicando a linha, a coluna e o motivo, antes de qualquer exportação.

## Asset IDs

Cada provedor gera identificadores de exatamente 20 caracteres:

```text
PREFIXO(4) + dígito de classe(1) + BLOCO CENTRAL(11) + CONTADOR(4)
```

O dígito de classe distingue o tipo de ativo (package, title, movie, poster, preview). O contador é sincronizado com o histórico já emitido e incrementado apenas no momento do download.
