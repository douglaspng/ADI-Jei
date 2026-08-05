# ADI Jei — Conversor de Metadados CSV → XML ADI

> Aplicação web (PWA) que automatiza a geração de arquivos **ADI XML** para plataformas de VOD, a partir de planilhas CSV de metadados de filmes e séries.

![status](https://img.shields.io/badge/status-em%20produção-brightgreen)
![stack](https://img.shields.io/badge/stack-React%20%7C%20TypeScript%20%7C%20Supabase-blue)
![pwa](https://img.shields.io/badge/PWA-instalável-purple)

🔗 **Demonstração:** https://adijeicomsat.lovable.app

> ℹ️ **Sobre este repositório:** este é um repositório de **vitrine** (showcase). O código-fonte da aplicação é privado por se tratar de um sistema de uso interno de uma operadora de conteúdo. Aqui você encontra a documentação técnica, decisões de arquitetura e resultados do projeto.

---

## 📖 Visão Geral

Operadoras de TV por assinatura e distribuidoras de VOD precisam entregar, junto de cada filme ou episódio, um arquivo **ADI XML** descrevendo o conteúdo e seus ativos digitais (vídeo, poster, preview). Esse arquivo segue regras rígidas de formatação, identificadores e integridade de arquivos.

O **ADI Jei** substitui esse processo manual: recebe uma planilha CSV, valida os dados linha a linha, calcula checksums reais dos ativos, gera identificadores únicos por provedor e exporta o XML final — individualmente ou em lote.

---

## 🎯 Problema que Resolve

| Antes (manual) | Depois (ADI Jei) |
|---|---|
| XML escrito à mão por título | Geração automática em lote |
| Asset IDs contados manualmente | IDs de 20 caracteres com contador sincronizado por provedor |
| Checksums calculados por ferramentas externas | MD5 e CRC32 calculados no app (local ou direto do S3) |
| Erros descobertos só na entrega | Validação por linha com pop-up detalhado antes da exportação |
| Datas, duração e idiomas inconsistentes | Normalização automática (`HH:MM:SS`, vigência, idioma/legenda) |

---

## 🚀 Funcionalidades

- **Importação de CSV** com validação de colunas obrigatórias, duplicatas e valores inválidos.
- **Geração de XML ADI** com nós `package`, `title`, `movie`, `poster` e `preview`.
- **Asset IDs por provedor** — 20 caracteres, contador incrementado apenas no download.
- **Checksums reais** — MD5 e CRC32 calculados via Web Worker (arquivos locais) ou streaming em Edge Function (S3 privado).
- **Cache de checksum por ETag** — recálculo apenas quando o objeto muda no bucket.
- **Suporte a séries** — colunas opcionais `EPISODE_ID` e `EPISODE_NAME`.
- **Histórico de XMLs** gerados, vinculado ao usuário autenticado.
- **Validação de paridade entre provedores** — alerta automático em caso de divergência de regras.
- **PWA** — instalável em desktop e mobile, interface responsiva.

---

## 🛠️ Stack Técnica

**Frontend:** React 18 · TypeScript 5 · Vite 5 · Tailwind CSS 3 · shadcn/ui (Radix) · React Router 7 · React Hook Form + Zod · TanStack Query

**Backend:** PostgreSQL gerenciado · Autenticação com Row Level Security · Edge Functions em Deno/TypeScript · Object Storage

**Qualidade:** ESLint 9 · Web Workers para processamento pesado · testes automatizados de checksum · CI/CD gerenciado

---

## 📐 Arquitetura

```text
┌──────────────────────────── Navegador (PWA) ────────────────────────────┐
│  Upload CSV → Parser → Validador por linha → Gerador de XML → Download  │
│                    │                                                     │
│                    └── Web Worker: MD5 / CRC32 de arquivos locais        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTPS autenticado
┌───────────────────────────────▼─────────────────────────────────────────┐
│  Edge Functions (Deno)                                                  │
│   • s3-checksum   → streaming do objeto S3 + MD5/CRC32 + cache por ETag │
│   • validate-admin→ autenticação com rate limiting por IP/usuário       │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│  PostgreSQL — histórico de XMLs · contadores de Asset ID ·              │
│  cache de checksum · tentativas de login  (todas com RLS)               │
└─────────────────────────────────────────────────────────────────────────┘
```

Detalhes em [`docs/ARQUITETURA.md`](docs/ARQUITETURA.md).

---

## 📄 Formato do CSV

Planilha UTF-8. Colunas obrigatórias e opcionais documentadas em [`docs/FORMATO-CSV.md`](docs/FORMATO-CSV.md).

Resumo: `TITULO_NACIONAL`, `TITULO_ORIGINAL`, `SINOPSE`, `CLASSIFICACAO`, `DURACAO`, `BILLING_ID`, `ANO`, `ELENCO`, `DIRETOR`, `GENERO_1`, `PAIS`, `IDIOMA`, `LEGENDADO`, `DVB` — e as opcionais `EXTRADATA`, `GENERO_2`, `GENERO_3`, `EPISODE_ID`, `EPISODE_NAME`.

---

## 🔒 Segurança

- Autenticação com senha protegida por hash e rate limiting (bloqueio após 5 tentativas em 15 min).
- **RLS** habilitado em todas as tabelas, com políticas por `auth.uid()`.
- Proteção contra **SSRF** na função de checksum: apenas hosts oficiais da AWS S3 sobre HTTPS.
- Bucket de storage privado com políticas de acesso explícitas.
- Preview de XML renderizado como texto puro, evitando XSS.
- Varreduras de segurança e de dependências executadas de forma recorrente.

Mais detalhes em [`docs/SEGURANCA.md`](docs/SEGURANCA.md).

---

## 🧪 Confiabilidade dos Checksums

O cálculo de integridade é o ponto mais crítico do sistema — um checksum errado invalida a entrega inteira. A suíte automatizada cobre:

- Vetores conhecidos de **MD5 (RFC 1321)** e **CRC32 (IEEE 802.3)**.
- Streaming em chunks de 1 byte a 256 KB, garantindo que não existam digests parciais.
- Payload de 60 MB, provando que não há truncamento em arquivos grandes.
- Prova explícita de que **ETags multipart do S3 não são o MD5 do arquivo** — regressão que já causou dados incorretos e hoje é bloqueada por teste.

---

## 🧩 Evolução do Projeto

1. Conversão simples de CSV para XML com IDs fixos.
2. Padronização de Asset IDs de 20 caracteres por provedor.
3. Remoção de credenciais do código e adoção de hash + variáveis de ambiente.
4. Parser de CSV com validação por linha e diálogo de erros.
5. Reescrita do módulo de checksum com testes automatizados e cache por ETag.
6. Importação de ativos do S3 via Edge Function com proteção contra SSRF.
7. Suporte a séries (`EPISODE_ID` / `EPISODE_NAME`).
8. Verificação automática de paridade entre provedores.
9. SEO, PWA e metadados Open Graph.
10. Hardening contínuo e atualização de dependências.

---

## 👨‍💻 Autor

Desenvolvido por **Henrique Douglas**.

- LinkedIn: [linkedin.com/in/henrique-douglas-08aaa4351](https://www.linkedin.com/in/henrique-douglas-08aaa4351/)

---

## 📄 Licença

Todos os direitos reservados. Ver [LICENSE](LICENSE).
