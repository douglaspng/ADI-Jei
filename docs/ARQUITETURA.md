# Arquitetura

## Visão em camadas

| Camada | Responsabilidade |
|---|---|
| **UI (React + Tailwind + shadcn/ui)** | Upload de CSV, listagem de títulos, modais de detalhe/erro, painéis de configuração de provedor e Asset ID. |
| **Domínio (TypeScript puro)** | Parser de CSV, validador por linha, gerador de XML ADI, formatação de duração/datas, regras de Asset ID. Sem dependência de framework — testável isoladamente. |
| **Processamento pesado (Web Worker)** | Cálculo de MD5 e CRC32 de arquivos locais sem travar a thread principal. |
| **Edge Functions (Deno)** | Operações que exigem segredo ou rede: leitura de objetos em bucket S3 privado e validação de credenciais administrativas. |
| **Banco (PostgreSQL + RLS)** | Histórico de XMLs, contadores de Asset ID, cache de checksum, tentativas de login. |

## Decisões técnicas relevantes

**PWA em vez de aplicativo desktop.** O público-alvo alterna entre máquinas e sistemas operacionais. Um PWA instalável entrega o mesmo atalho na área de trabalho sem processo de distribuição, assinatura de binários ou atualização manual.

**Lógica de domínio isolada do React.** Parser, validador e gerador de XML são funções puras. Isso permitiu testar as regras mais críticas do negócio sem renderizar componentes e reduziu drasticamente o custo de mudanças no formato do XML.

**Checksum em Web Worker.** Arquivos de vídeo passam facilmente de 1 GB. Calcular na thread principal congelava a interface; o worker mantém a UI responsiva e permite progresso incremental.

**Checksum de S3 em streaming, não em memória.** A Edge Function lê o objeto em chunks e alimenta os digests progressivamente. Não há limite de tamanho por consumo de memória e não existe caminho alternativo "rápido" que produza valores parciais.

**Cache por ETag.** O par `(bucket, chave, etag)` funciona como chave natural de cache. Se o objeto muda, o ETag muda e o cache é naturalmente invalidado — sem necessidade de expiração por tempo.

**ETag ≠ MD5.** Uploads multipart no S3 produzem ETags que não correspondem ao MD5 do arquivo. Derivar o MD5 do ETag foi um bug real do projeto; hoje um teste automatizado impede o retorno dessa abordagem.

**Contador de Asset ID incrementado apenas no download.** Visualizar um título não deve consumir um identificador. O incremento acontece somente quando o XML é efetivamente exportado, evitando lacunas na sequência entregue à operadora.

**Verificação de paridade entre provedores.** Provedores que devem seguir regras idênticas são comparados na inicialização do app; qualquer divergência de formato ou contador gera um alerta visível ao operador.

## Fluxo de geração

```text
CSV
 └─> Parser (normaliza colunas, idiomas, gêneros, duração)
      └─> Validador por linha (campos obrigatórios, formatos, regras de episódio)
           ├─> erros? -> diálogo detalhado, exportação bloqueada
           └─> ok
                └─> Anexar ativos (upload local ou importação S3)
                     └─> Checksums MD5 + CRC32 + tamanho
                          └─> Gerador de XML ADI
                               └─> Preview -> Download (incrementa Asset ID)
                                    └─> Registro no histórico
```
