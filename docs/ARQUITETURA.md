# SAT Knowledge Base (KB) — Especificação de Arquitetura

## Contexto
Base de conhecimento interna da SAT Automação (integradora de automação industrial) para o time de desenvolvimento: lições aprendidas e manuais técnicos (inversores, CLPs, robôs etc.).

O uso principal é por técnicos em comissionamento, muitas vezes com internet ruim ou sem internet.

O projeto é 100% novo e não reaproveita código de outros projetos.

## Premissas
- **Conteúdo**: somente documentos internos da SAT e manuais de fabricantes. Não há restrição de confidencialidade para envio à API.
- **Servidor SAT leve**: NÃO roda LLM. Roda a API, o banco e os embeddings em CPU.
- **LLM**: API da Anthropic (a key já existe), abstraída por `IChatClient` (Microsoft.Extensions.AI).
  - O modelo é configurável; o padrão é o Haiku mais recente.
  - Verificar se o SDK oficial da Anthropic para .NET expõe `IChatClient`. Se não expuser, implementar um adapter.
- **Acesso ao servidor**: via LAN ou VPN.
- **Cliente**: notebooks Windows agora; Android/iOS no futuro.
- **Plataforma**: .NET 10 (LTS). Tudo gratuito/open-source, exceto a API da Anthropic.

## Estrutura da solução
```
src/
  KB.Core     — domínio, chunking, busca híbrida, contratos de sync/DTOs (compartilhado)
  KB.Server   — ASP.NET Core Web API
  KB.App      — .NET MAUI Blazor Hybrid (target inicial: Windows)
  KB.UI       — Razor Class Library com os componentes (reuso futuro em versão web)
tests/
  KB.Core.Tests, KB.Server.Tests (xUnit)
docs/
  ARQUITETURA.md, PLANO.md, DECISOES.md
```

## Requisitos funcionais
1. **Consulta em linguagem natural**: primeiro a RESPOSTA DIRETA, depois as fontes (documento + página/seção + botão para abrir o original na página citada).
2. **Responder SOMENTE com base na KB.** Se não houver resultado relevante, responder "Não encontrado na base", sem busca adicional.
3. **Busca híbrida obrigatória**, com fusão por RRF:
   - SQLite FTS5 (BM25), para códigos exatos de falha como `F30002`, `16#8090` e `0x0400`;
   - similaridade vetorial.
4. **Modos de resposta**:
   - Online: busca no servidor → limiar de relevância → LLM gera uma resposta curta com fontes.
   - Offline: busca híbrida local → trecho mais relevante destacado + fontes (sem LLM).
5. **Cadastro de lição por texto** no app (editor Markdown + campos de categoria):
   - Offline: salvar como *Pendente*, já pesquisável via FTS5 local.
   - Online: enviar ao servidor, que indexa (chunks + embeddings) e distribui no próximo sync.
6. **Upload de documentos** (DOCX, PDF, TXT, MD): somente online, para o servidor.
7. **Sync incremental** servidor → cliente, com sync seletivo:
   - lições: sempre sincronizadas;
   - manuais e documentos: conforme as categorias que o usuário assinar no app.
8. **Categorias**: tabelas cadastradas, não texto livre.
   - Fabricante
   - Tipo de Equipamento (CLP, IHM, Inversor, Servo, Robô, Rede, Visão/Sensor, Supervisório, Segurança etc.)
   - Família/Modelo (ex.: SINAMICS G120, PowerFlex 525)
   - Tipo de Documento (Lição Aprendida, Manual, Nota Técnica, Procedimento, Lista de Falhas)
   - Idioma (PT, EN, DE, ...)
   - Mais tags livres.
9. **Formato das lições**: Markdown + frontmatter YAML, com links `[[lição]]` e backlinks. As "relacionadas" são calculadas por similaridade vetorial.

## Servidor (KB.Server)
- **Banco**: SQLite (Microsoft.Data.Sqlite com FTS5) via EF Core ou Dapper. Justificar a escolha em DECISOES.md. Vetores como BLOB float32.
- **Busca vetorial**:
  - força bruta em memória com `System.Numerics.Tensors.TensorPrimitives.CosineSimilarity`;
  - interface `IVectorIndex`, para permitir a troca futura por sqlite-vec.
- **Ingestão**:
  - DOCX via DocumentFormat.OpenXml;
  - PDF via UglyToad.PdfPig, guardando o número da página;
  - OCR via Tesseract quando a página não tiver texto extraível;
  - TXT e MD lidos direto.
- **Chunking**:
  - Lições: por seção/heading.
  - Manuais: por seção/página, com overlap.
  - Tabelas de falhas: detectar e gerar 1 chunk por linha (código + descrição + causa + solução).
- **Embeddings**: modelo multilíngue pequeno em ONNX (ex.: multilingual-e5-small ou paraphrase-multilingual-MiniLM-L12-v2), via Microsoft.ML.OnnxRuntime + Microsoft.ML.Tokenizers.
  - **FAZER UM SPIKE PRIMEIRO (F0)**: validar que o tokenizer do modelo funciona em .NET e que os vetores batem com a referência em Python.
  - O MESMO modelo deve rodar no cliente.
- **RAG**:
  - top-K híbrido → limiar configurável → prompt restrito ao contexto recuperado;
  - saída estruturada em JSON: `{ found, answer, sources[{ docId, title, page, section, chunkId }] }`;
  - o LLM retorna `found=false` se o contexto for insuficiente.
- **Arquivos originais**: armazenados em disco, identificados por hash SHA-256.
- **Sync**:
  - versão monotônica (long) por registro e tombstones para exclusões;
  - o endpoint de delta entrega chunks + embeddings já calculados;
  - arquivos baixados por hash;
  - filtro pelas categorias assinadas.
- **Perfis**:
  - Admin: categorias, upload, exclusão, gestão de usuários.
  - Usuário: consulta e cadastro de lição.

## Autenticação

### Entra ID (registro já criado)
- TenantId: `767cc0c3-b762-4b65-9a60-85b71419d188`
- ClientId: `3f75fc6d-e78f-41fc-a751-18a26d593dc5`
- Um único registro para o app e a API: locatário único, cliente público, sem client secret.
- Escopo exposto: `api://3f75fc6d-e78f-41fc-a751-18a26d593dc5/access_as_user`
- Redirect URI (desktop): `http://localhost` (MSAL com o navegador do sistema).

**KB.Server** (Microsoft.Identity.Web) — `appsettings.json`:
```json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "767cc0c3-b762-4b65-9a60-85b71419d188",
  "ClientId": "3f75fc6d-e78f-41fc-a751-18a26d593dc5",
  "Audience": "api://3f75fc6d-e78f-41fc-a751-18a26d593dc5"
}
```
- Validar o escopo `access_as_user` nos endpoints.
- Identificar o usuário pelo claim `oid`. No primeiro login, criar o usuário no banco com perfil Usuário.
- Perfis gerenciados no banco da aplicação, NÃO via App Roles do Entra.
- Primeiro Admin definido por configuração: `Auth:BootstrapAdmins` (lista de e-mails).

**KB.App** (Microsoft.Identity.Client):
- `PublicClientApplicationBuilder` com TenantId, ClientId e RedirectUri `http://localhost`.
- Escopos: `["api://3f75fc6d-e78f-41fc-a751-18a26d593dc5/access_as_user"]`.
- Fluxo: `AcquireTokenSilent`; se falhar, `AcquireTokenInteractive`.
- Cache de token persistido com Microsoft.Identity.Client.Extensions.Msal (criptografado via DPAPI no Windows).
- Offline: liberar o acesso se o cache tiver uma conta de login anterior válido.

### Fallback
- Modo "Local": usuário/senha no banco, hash com PasswordHasher do ASP.NET Identity ou BCrypt, emitindo JWT.
- Seleção do modo por configuração: `Auth:Mode` = `Entra` | `Local`.
- Usar o modo Local caso a política do tenant bloqueie o consentimento.

## Cliente (KB.App)
- MAUI Blazor Hybrid, com os componentes Razor em KB.UI.
- **Banco local SQLite**: chunks, vetores, FTS5, fila de pendentes, estado do sync e categorias assinadas.
- **Embedding offline**: a pergunta é vetorizada com o mesmo ONNX do servidor (modelo empacotado no app ou baixado no primeiro sync).
- **Conectividade**: detectar se o servidor está alcançável e mostrar um indicador online/offline na UI.
- **Sync**: automático ao ficar online, mais um botão manual.
- **Abrir o original**:
  - PDF no WebView2 com `#page=N`;
  - demais formatos no app padrão do Windows.

## Telas mínimas
- Consulta (com filtros de categoria)
- Resultado (resposta + fontes)
- Visualizador de documento/lição
- Nova lição
- Pendentes de sync
- Configuração de sync seletivo
- Admin (upload, categorias, usuários)
- Login

## Requisitos não funcionais
- **Logs**: Serilog.
- **Configuração**: appsettings + user-secrets para a API key (`Anthropic:ApiKey`), que nunca é commitada.
- **Testes unitários**: chunking, fusão RRF, limiar e delta de sync.
- **README** com:
  - setup do servidor;
  - configuração do Entra;
  - geração/validação do modelo ONNX;
  - instalação do app.

## Fases
- **F0**: spike de embeddings ONNX em .NET (tokenizer + validação contra Python).
- **F1**: Core + Server (ingestão, busca híbrida, RAG).
- **F2**: App offline + sync.
- **F3**: cadastro de lição + fila de pendentes.
- **F4**: autenticação (Entra + fallback Local).
- **F5**: sync seletivo + telas Admin.

## Forma de trabalho
1. Antes de codar, apresentar:
   - plano por fases;
   - modelo de dados (tabelas);
   - contratos da API;
   - decisões técnicas pendentes.
   
   Aguardar aprovação e então salvar o plano em `docs/PLANO.md`.
2. Commits pequenos por fase.
3. Registrar as decisões técnicas em `docs/DECISOES.md`.
4. Em caso de dúvida, perguntar em vez de assumir.
