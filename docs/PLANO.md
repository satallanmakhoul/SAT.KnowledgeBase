# SAT Knowledge Base — Plano

Fonte da verdade da especificação: [ARQUITETURA.md](ARQUITETURA.md). Decisões: [DECISOES.md](DECISOES.md).
Atualizar este arquivo ao concluir cada fase.

## Status

| Fase | Escopo | Status |
|---|---|---|
| F0 | Spike de embeddings ONNX (tokenizer + paridade com Python + benchmark) | Não iniciada |
| F1 | Core + Server (ingestão, busca híbrida, RAG, importação em lote) | Não iniciada |
| F2 | App offline + sync | Não iniciada |
| F3 | Cadastro de lição + fila de pendentes | Não iniciada |
| F4 | Autenticação (Entra + fallback Local) | Não iniciada |
| F5 | Sync seletivo + Admin + backup + limpeza de órfãos | Não iniciada |
| Backlog | Anexos em lições | — |

## Premissas fixadas
- Servidor: Windows Server (Kestrel como Windows Service; Tesseract nativo Windows).
- F0 testa os dois modelos: `multilingual-e5-small` e `paraphrase-multilingual-MiniLM-L12-v2`.
- Modelo ONNX empacotado no app; **o mesmo arquivo** (mesma precisão) no servidor e no cliente. Int8 só se o benchmark do F0 mostrar que mantém a qualidade; senão fp32 nos dois lados.
- Lições existentes (~64 arquivos DOCX/PDF/TXT) entram por importação em lote como `LESSON` com o arquivo original, sem conversão para Markdown, e sincronizam sempre.
- Respostas do RAG sempre em português, mesmo com fonte em outro idioma.

---

## 1. Fases

### F0 — Spike de embeddings ONNX
Objetivo: provar que tokenizer + ONNX em .NET reproduzem o Python e escolher modelo, precisão e limiar inicial.

1. `tools/embeddings/` (Python, venv; usar 3.12 se torch não tiver wheel para 3.14):
   - `requirements.txt` (sentence-transformers, optimum[onnxruntime], onnxruntime).
   - `export.py`: exporta os 2 modelos para ONNX fp32 e gera int8 (quantização dinâmica). Saída em `models/` (gitignored) + `manifest.json` (nome, dims, sha256, precisão, prefixos).
   - `reference.py`: `reference.json` com ~40 frases (PT/EN/DE, códigos `F30002`, `16#8090`, `0x0400`, `F-0003`, `E.FE`, texto > 512 tokens) com `input_ids` e vetores do **sentence-transformers (PyTorch)** — a referência é o modelo original, não o ONNX.
2. `src/KB.Core`: `OnnxEmbeddingGenerator : IEmbeddingGenerator<string, Embedding<float>>` (Microsoft.Extensions.AI), tokenizer Microsoft.ML.Tokenizers (SentencePiece/Unigram XLM-R), mean pooling com attention mask, L2, truncamento 512, prefixos `query:`/`passage:` (e5) configuráveis.
3. `tests/KB.Core.Tests/Embeddings`: paridade contra `reference.json` (pulados se os modelos não estiverem baixados).
4. `tools/KB.EmbeddingBench` (console) — benchmark de recuperação:
   - Corpus: passagens sintéticas **+ trechos reais das lições da SAT** (pasta fornecida pelo usuário; corpus real fora do git em `tools/embeddings/data/`).
   - Perguntas: **perguntas reais com gabarito** (chunk/documento esperado) + **perguntas fora da base** (gabarito = nenhum).
   - Métricas por modelo × {fp32, int8}: Recall@5, MRR, latência por consulta, RAM, tamanho do arquivo.
   - Calibração do limiar: distribuição do cosseno máximo para perguntas dentro vs fora da base → τ inicial que maximiza o acerto de "Não encontrado".
5. Critérios de aceite:
   - `input_ids` idênticos ao Python em 100% das frases;
   - cosseno .NET fp32 vs PyTorch ≥ 0,9999;
   - int8 adotado se cosseno médio vs PyTorch ≥ 0,99 **e** queda de MRR/Recall@5 ≤ 2 p.p. vs fp32;
   - latência de consulta em CPU < 100 ms.
6. Plano B do tokenizer: Unigram próprio lendo `tokenizer.json` em C# puro (deve rodar em Android/iOS no futuro — sem binding nativo).
7. Registrar resultado, modelo, precisão e τ inicial em DECISOES.md; seção do README sobre gerar/validar o modelo.

### F1 — Core + Server
- Solução `SAT.KnowledgeBase.slnx`, `Directory.Build.props` (net10.0, nullable, file-scoped, warnings as errors), Central Package Management.
- KB.Core: entidades/DTOs, schema SQL compartilhado + migrações (`PRAGMA user_version`), chunking (lição por heading; manual por seção/página com overlap; tabela de falhas 1 chunk/linha), normalização de `fault_code`, FTS5 query builder, `IVectorIndex` (força bruta com `TensorPrimitives.CosineSimilarity`), `HybridSearcher` (RRF), `RelevanceGate`.
- KB.Server: ingestão (OpenXml, PdfPig + página, Tesseract por+eng+deu, TXT/MD) em fila de background (`Channel`), arquivos por SHA-256, RAG com `IChatClient` (Anthropic; verificar `IChatClient` no SDK oficial, senão adapter), prompt restrito ao contexto e **resposta sempre em português**, saída JSON estruturada, Serilog, endpoints de categorias/documentos/busca/ask. Auth em modo dev (usuário fixo) até F4.
- **Importação em lote** `tools/KB.Import` (CLI): varre uma pasta (DOCX/PDF/TXT/MD), lê manifesto opcional (CSV/JSON: arquivo → doc_type, fabricante, tipo, modelo, idioma, tags; padrão `LESSON`), envia via `POST /documents` (dedupe por SHA-256, idempotente, relatório de sucesso/falha). Usada para os ~64 arquivos existentes.
- Testes: chunking, RRF, limiar, detecção de tabela de falhas, normalização de códigos e query builder FTS.

### F2 — App offline + sync
- KB.UI (RCL) + KB.App (MAUI Blazor Hybrid, Windows). SQLite local com o mesmo schema de leitura do Core.
- Endpoint de delta + cliente de sync (automático ao ficar online + botão), download de arquivos por hash, indicador online/offline (`/api/v1/health`).
- Telas: Consulta (filtros), Resultado, Visualizador (PDF no WebView2 `#page=N`; outros via app padrão).
- Busca offline: híbrida local + trecho destacado + fontes.
- Testes: delta de sync (servidor e aplicação no cliente).

### F3 — Cadastro de lição + pendentes
- Editor Markdown + frontmatter + campos de categoria; `[[lição]]` + backlinks; relacionadas por similaridade.
- Offline: salva *Pendente* (FTS local). Online: POST idempotente por `clientId`. Tela Pendentes.

### F4 — Autenticação
- Pré-requisito: definir HTTPS na LAN/VPN (decisão pendente 10).
- Server: Microsoft.Identity.Web (escopo `access_as_user`), usuário criado no 1º login pelo `oid`, `Auth:BootstrapAdmins`, perfis no banco; modo `Local` (PasswordHasher + JWT) via `Auth:Mode`.
- App: MSAL (silent → interactive), cache DPAPI, acesso offline com conta em cache. Tela Login.

### F5 — Sync seletivo + Admin + operação
- Assinatura de categorias no app; servidor filtra o delta; troca de assinatura → ressync do escopo novo + remoção local do que saiu.
- Telas Admin: upload (inclui **upload múltiplo**, reaproveitando a lógica do KB.Import), categorias, usuários/perfis.
- **Backup do servidor**: SQLite Online Backup API (`SqliteConnection.BackupDatabase`) para snapshot consistente + cópia da pasta de arquivos; agendado (BackgroundService, retenção configurável) e acionável pelo Admin.
- **Limpeza de órfãos**: job que remove arquivos cujo SHA-256 não é referenciado por nenhum documento ativo (após soft delete), com período de carência configurável e log.
- README completo (setup do servidor, Entra, modelo ONNX, instalação do app, **backup/restore**).

### Backlog pós-F5
- Anexos (fotos/prints) em lições novas criadas pelo app.

Cada fase: commits pequenos por etapa; este arquivo atualizado ao concluir.

---

## 2. Modelo de dados (SQLite)

IDs: `TEXT` GUID v7 (`Guid.CreateVersion7()`), para o cliente criar lições offline sem colisão. Categorias `INTEGER`. Datas UTC ISO-8601.

### Servidor
| Tabela | Colunas principais |
|---|---|
| `users` | id, entra_oid (UQ, null no Local), email (UQ), display_name, role (`Admin`/`User`), password_hash (null no Entra), is_active, created_at, last_login_at |
| `manufacturers` | id, name (UQ), version, is_deleted |
| `equipment_types` | id, name (UQ), version, is_deleted |
| `models` | id, manufacturer_id, equipment_type_id, name, version, is_deleted |
| `doc_types` | id, code (`LESSON`,`MANUAL`,`TECH_NOTE`,`PROCEDURE`,`FAULT_LIST`), name, version |
| `languages` | code (PK: `pt`,`en`,`de`…), name, version |
| `tags` | id, name (UQ, normalizado) |
| `documents` | id, doc_type_id, title, slug (UQ p/ lições, alvo de `[[...]]`), manufacturer_id?, equipment_type_id?, model_id?, language_code, file_sha256?, file_name?, mime?, file_size?, page_count?, content_md? (lições criadas no app: markdown com frontmatter = fonte da verdade; lições importadas: null + arquivo original), source (`App`/`Upload`/`Import`), status (`Processing`/`Ready`/`Failed`), error?, created_by, created_at, updated_at, **version**, **is_deleted** |
| `document_tags` | document_id, tag_id (PK composta) |
| `document_links` | source_id, target_slug, target_id? (resolvido) → backlinks |
| `chunks` | id, document_id, ordinal, kind (`Section`/`Page`/`FaultRow`), section?, page_start?, page_end?, fault_code? (como no documento), fault_code_norm? (maiúsculo, sem `-` `.` espaço), text, token_count, embedding BLOB (float32 LE) |
| `chunks_fts` | FTS5 external content sobre `chunks` (text, section, fault_code_norm) + title; `tokenize="unicode61 remove_diacritics 2 tokenchars '#_'"` (mantém `16#8090`, `0x0400`, `F30002` como token único). **`-` e `.` não são tokenchars.** Triggers de sync |
| `sync_counter` | linha única `value` — versão global monotônica; toda mutação de documento/categoria recebe `++value` na mesma transação |
| `embedding_model` | name, sha256, dims (troca exige reindexação) |
| `ask_log` | id, user_id, question, found, top_score, latency_ms, created_at (calibração do limiar) |
| `backups` | id, started_at, finished_at, path, size, status, error? |

Tombstones = `is_deleted=1` + nova `version`. Sync **por documento**: quando a versão muda, o cliente substitui todos os chunks → chunks sem versão própria.

Arquivos: `{Storage:Root}/files/{sha[0..2]}/{sha}`, deduplicados; removidos só pelo job de órfãos.

### Cliente (além do espelho de categorias/documents/chunks/chunks_fts)
| Tabela | Colunas |
|---|---|
| `sync_state` | key/value: `last_version`, `subscriptions_hash`, `model_sha256`, `last_sync_at` |
| `subscriptions` | kind (`Manufacturer`/`EquipmentType`/`Model`), category_id |
| `pending_lessons` | client_id, markdown, metadados, status (`Pending`/`Sending`/`Failed`), error?, created_at, server_id? — também indexadas no FTS local (documento com `status=Pending`) |
| `local_files` | sha256, path, size, downloaded_at |

### Códigos de falha (FTS)
- Indexação: o detector de tabela de falhas extrai `fault_code` e grava `fault_code_norm` (ex.: `F-0003` → `F0003`, `E.FE` → `EFE`, `16#8090` → `16#8090`).
- Consulta: o query builder detecta termos com formato de código (letras+dígitos misturados, `0x…`, `…#…`, ou alfanuméricos separados por `-`/`.`) e gera `("F-0003" OR fault_code_norm:F0003)` — a frase FTS5 casa os tokens adjacentes `F` `0003` no texto, e a coluna normalizada casa a forma sem separador. Demais termos: escapados e combinados normalmente.

---

## 3. Contratos da API (`/api/v1`, JSON, gzip)

Auth: Bearer (Entra ou JWT Local), escopo `access_as_user`. `[A]` = Admin.

| Método | Rota | Request → Response |
|---|---|---|
| GET | `/health` (anônimo) | → `{ status, version, embeddingModel: { name, sha256, dims } }` |
| POST | `/auth/local/login` (só `Mode=Local`) | `{ username, password }` → `{ accessToken, expiresAt }` |
| GET | `/me` | → `{ id, email, displayName, role }` |
| POST | `/ask` | `{ question, filters? }` → `{ found, answer /*sempre PT*/, sources: [{ docId, title, page, section, chunkId, snippet }], latencyMs }` |
| POST | `/search` | `{ query, filters?, topK? }` → `{ hits: [{ chunkId, docId, title, page, section, snippet, score, ftsRank?, vecScore? }], passedThreshold }` |
| GET | `/categories` | → `{ manufacturers[], equipmentTypes[], models[], docTypes[], languages[], tags[] }` |
| POST/PUT/DELETE | `/categories/{kind}/{id?}` [A] | CRUD (DELETE = tombstone; 409 se em uso) |
| GET | `/documents?filters&page&pageSize` | → lista paginada de metadados |
| GET | `/documents/{id}` | → metadados + `contentMd` (lição do app) + backlinks |
| POST | `/documents` [A] multipart | 1..N `file` + `metadata` (por arquivo ou comum) → `202 [{ fileName, id, status, duplicateOf? }]` (usado pelo KB.Import e pelo upload múltiplo) |
| PUT | `/documents/{id}` [A] | metadados (`baseVersion` → 409 em conflito) |
| DELETE | `/documents/{id}` [A] | → 204 (tombstone) |
| POST | `/documents/{id}/reindex` [A] | → 202 |
| GET | `/files/{sha256}` | stream com suporte a `Range` |
| POST | `/lessons` | `{ clientId, markdown, categories… }` → `201 { id, version }` (idempotente por `clientId`) |
| PUT | `/lessons/{id}` | `{ markdown, baseVersion }` → `{ version }` / 409 (autor ou Admin) |
| GET | `/lessons/{id}/related?k=5` | → `[{ docId, title, score }]` |
| POST | `/sync/delta` | `{ sinceVersion, subscriptions: [{ kind, id }], modelSha256, limit }` → `{ toVersion, hasMore, categories: {…com isDeleted}, documents: [{ …metadados, version, isDeleted, chunks: [{ id, ordinal, kind, section, pageStart, pageEnd, faultCode, faultCodeNorm, text, embedding /*base64 float32*/ }] }] }` — `LESSON` (do app ou importada) sempre, com arquivo; demais filtrados por assinatura; `modelSha256` divergente → 409 |
| GET | `/admin/users` [A] | → lista |
| PUT | `/admin/users/{id}` [A] | `{ role, isActive }` |
| POST | `/admin/backups` [A] | → `202 { id }`; GET `/admin/backups` → histórico |
| POST | `/admin/maintenance/orphans` [A] | `{ dryRun }` → `{ files: [{ sha256, size }], freedBytes }` |

Busca híbrida (Core; `/search`, `/ask` e offline): FTS5 top-50 (bm25) + vetorial top-50 → RRF (k=60) → top-K (8). Limiar: passa se `maxCosine ≥ τ` (τ do benchmark F0, refinado com `ask_log`) **ou** match FTS de termo com formato de código. Sem passar → `found=false` / "Não encontrado na base", sem chamar o LLM.

---

## 4. Decisões técnicas
Registradas em [DECISOES.md](DECISOES.md). Pendentes: modelo/precisão (F0), tokenizer (F0), limiar τ (F0/F1), chunking e tabela de falhas (F1), HTTPS na LAN/VPN (antes da F4).

## 5. Insumos aguardados do usuário
- Pasta com lições reais + perguntas reais com gabarito, incluindo perguntas fora da base (F0).
- Manuais de exemplo (F1).
- Pasta dos ~64 documentos para importação (F1).
