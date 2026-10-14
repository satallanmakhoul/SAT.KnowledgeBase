# Decisões técnicas

Registro de decisões relevantes (data, decisão, motivo). Mais recentes no fim.

## Tomadas

| # | Data | Decisão | Motivo |
|---|---|---|---|
| D01 | 2026-10-14 | Servidor em **Windows Server** (Kestrel como Windows Service; Tesseract com binários nativos Windows). | Infraestrutura disponível na SAT. |
| D02 | 2026-10-14 | Acesso a dados com **Dapper + Microsoft.Data.Sqlite** (não EF Core). Migrações por scripts SQL embarcados + `PRAGMA user_version`. | FTS5, triggers e vetores em BLOB ficam mais simples em SQL explícito; EF Core não modela tabelas virtuais FTS5; o mesmo schema é compartilhado por servidor e cliente. |
| D03 | 2026-10-14 | IDs `TEXT` GUID v7 (`Guid.CreateVersion7()`); categorias `INTEGER`. | Cliente cria lições offline sem colisão; GUID v7 é ordenável por tempo. |
| D04 | 2026-10-14 | F0 avalia **dois modelos**: `multilingual-e5-small` e `paraphrase-multilingual-MiniLM-L12-v2`, em fp32 e int8. | Escolha baseada em paridade com Python e benchmark de recuperação com dados reais. |
| D05 | 2026-10-14 | **Mesmo arquivo ONNX** (mesma precisão) no servidor e no cliente; int8 só se mantiver a qualidade no benchmark, senão fp32 nos dois lados. | Servidor vetoriza documentos e cliente vetoriza a pergunta; precisões diferentes divergem os vetores e degradam a busca. |
| D06 | 2026-10-14 | Modelo ONNX **empacotado no app**. | Funciona offline desde a instalação (uso principal em campo sem internet). |
| D07 | 2026-10-14 | Referência de paridade do F0 é o **sentence-transformers (PyTorch)**, não o ONNX exportado. | Valida ao mesmo tempo a exportação ONNX e a implementação .NET. |
| D08 | 2026-10-14 | **IChatClient da Anthropic**: verificar suporte no SDK oficial em F1; adapter próprio se faltar. Modelo padrão `claude-haiku-5-5`, configurável. | Spec exige abstração via Microsoft.Extensions.AI. |
| D09 | 2026-10-14 | RAG **responde sempre em português**, mesmo com fonte em outro idioma. | Usuários são técnicos brasileiros; manuais são frequentemente EN/DE. |
| D10 | 2026-10-14 | Códigos de falha: FTS5 `unicode61 remove_diacritics 2 tokenchars '#_'`; **`-` e `.` não são tokenchars**. Coluna `fault_code_norm` (maiúsculo, sem separadores) na indexação; o query builder gera frase FTS5 + match em `fault_code_norm` para termos com formato de código. | Cobre `F30002`, `16#8090`, `0x0400`, `F-0003`, `E.FE` sem quebrar a tokenização de texto comum (datas, versões, palavras hifenizadas). |
| D11 | 2026-10-14 | Sync **por documento**: versão global monotônica + tombstone (`is_deleted`); ao mudar a versão do documento, o cliente substitui todos os seus chunks. | Chunks não precisam de versão própria; delta simples e consistente após reindexação. |
| D12 | 2026-10-14 | Delta em **JSON + gzip**, embeddings em base64 (float32 LE). MessagePack só se houver gargalo medido. | Simplicidade e depuração; ~2 KB por chunk em 384 dims. |
| D13 | 2026-10-14 | **Arquivos originais baixados no sync**: lições sempre; demais conforme assinatura. | Abrir o original offline é o caso principal. |
| D14 | 2026-10-14 | Assinatura por **Fabricante, Tipo de Equipamento e Família/Modelo**; documento entra se casar com qualquer assinatura. | Recorte natural do trabalho de comissionamento. |
| D15 | 2026-10-14 | Lições com **publicação direta** (sem aprovação); edição só pelo autor ou Admin, com concorrência otimista (`baseVersion`). | Reduz atrito no cadastro em campo. |
| D16 | 2026-10-14 | **Importação em lote** (`tools/KB.Import`, CLI em F1; upload múltiplo no Admin em F5). Lições existentes (~64 arquivos DOCX/PDF/TXT) entram como `LESSON` com o arquivo original, sem conversão para Markdown, e sincronizam sempre. | Aproveita o acervo atual sem retrabalho de conversão. |
| D17 | 2026-10-14 | Benchmark do F0 usa passagens sintéticas **e** trechos reais das lições da SAT, com perguntas reais com gabarito e perguntas fora da base para calibrar o limiar. | Escolha de modelo e limiar representativa do uso real. |
| D18 | 2026-10-14 | F5 inclui **backup** (SQLite Online Backup API + cópia da pasta de arquivos, agendado e sob demanda) e **limpeza de arquivos órfãos** (SHA sem referência ativa, com carência). | Operação segura do servidor; soft delete não libera disco sozinho. |

## Pendentes

| # | Tema | Quando |
|---|---|---|
| P1 | Modelo de embeddings e precisão (fp32/int8) | F0 |
| P2 | Tokenizer: Microsoft.ML.Tokenizers vs Unigram próprio em C# | F0 |
| P3 | Limiar de relevância τ (inicial no F0, refinado com `ask_log`) | F0/F1 |
| P4 | Parâmetros de chunking (~350 tokens, overlap ~15%) e heurística de tabela de falhas, com manuais de exemplo do usuário | F1 |
| P5 | HTTPS na LAN/VPN (CA interna ou autoassinado) | Antes da F4 |

## Backlog
- Anexos (fotos/prints) em lições novas criadas pelo app (pós-F5).
