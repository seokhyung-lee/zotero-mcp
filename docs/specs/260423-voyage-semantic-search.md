# Voyage-backed Semantic Search Patch for zotero-mcp

## 요약

이 작업은 `zotero-mcp`의 semantic search backend에 Voyage provider를 추가해서 `voyage-4-large` embedding과 optional `rerank-2.5` reranking을 지원하도록 만드는 것이다. 기존 MCP tool surface와 semantic search result schema는 유지하고, provider wiring만 확장하는 surgical patch로 제한한다. 읽기와 쓰기 경로, 기존 hybrid Zotero mode, 기존 ChromaDB 기반 indexing 흐름은 그대로 둔다. 이번 변경은 `src/zotero_mcp/chroma_client.py`, `src/zotero_mcp/semantic_search.py`, `src/zotero_mcp/setup_helper.py`, `src/zotero_mcp/cli.py`, `pyproject.toml`, 관련 테스트에 집중한다. 사용자는 `ZOTERO_EMBEDDING_MODEL=voyage` 또는 setup flow를 통해 Voyage backend를 선택할 수 있어야 한다.

## 목표

`zotero-mcp`가 semantic indexing에는 Voyage document embeddings를, query-time retrieval에는 Voyage query embeddings를, optional reranking에는 Voyage reranker를 사용할 수 있게 한다.

## 범위

- `ChromaClient`에 Voyage embedding backend 추가
- query/document asymmetric embedding을 위해 `input_type="document"`와 `input_type="query"`를 분리 적용
- semantic reranker abstraction에 Voyage provider 추가
- config file, existing embedding env flow, setup helper, CLI masking에 Voyage wiring 추가
- semantic extra dependency에 `voyageai` 추가
- Chroma collection compatibility reset logic를 Voyage provider까지 확장
- 기존 result ordering 개선은 허용하되 result schema는 변경하지 않음
- embedding credential surface는 최소화하고 reranker는 config-file source of truth로 유지
- mock 기반 unit test 추가 또는 확장

## 비목표

- MCP tool 이름, 인자 형태, 응답 schema 변경
- semantic search 결과에 `rerank_score` 같은 새 필드 추가
- live API call 기반 end-to-end integration test 추가
- semantic search 외의 Zotero read/write flow 변경
- Voyage custom dimension tuning을 위한 user-facing feature 추가
- unverified `base_url` support 추가
- env-driven Voyage reranker config surface 추가
- 기존 local cross-encoder reranker 제거 또는 동작 방식 변경

## 배경과 현재 컨텍스트

이 저장소는 Python package 구조를 사용하며 semantic search 관련 핵심 구현은 `src/zotero_mcp/` 아래에 모여 있다.

- [src/zotero_mcp/chroma_client.py](/Users/seokhyung/Projects/AI-Related/zotero-mcp/src/zotero_mcp/chroma_client.py)
  현재 `OpenAIEmbeddingFunction`, `GeminiEmbeddingFunction`, `HuggingFaceEmbeddingFunction`, `ChromaClient`, `create_chroma_client()`를 제공한다. `ChromaClient.search()`는 custom embedding function에 대해 `embed_query()`를 우선 사용한다. 현재 custom embedding tuple에는 OpenAI, Gemini, HuggingFace만 포함되어 있다.
- [src/zotero_mcp/semantic_search.py](/Users/seokhyung/Projects/AI-Related/zotero-mcp/src/zotero_mcp/semantic_search.py)
  현재 `CrossEncoderReranker`와 `ZoteroSemanticSearch`가 있다. reranker config는 `enabled`, `model`, `candidate_multiplier`만 읽고, provider 개념은 아직 없다. reranker는 result ordering만 바꾸고 반환 schema는 그대로 유지한다.
- [src/zotero_mcp/setup_helper.py](/Users/seokhyung/Projects/AI-Related/zotero-mcp/src/zotero_mcp/setup_helper.py)
  interactive semantic setup는 `default`, `openai`, `gemini`만 지원한다. semantic config를 저장하고 Claude/standalone env를 구성한다.
- [src/zotero_mcp/cli.py](/Users/seokhyung/Projects/AI-Related/zotero-mcp/src/zotero_mcp/cli.py)
  `setup-info`에서 env를 obfuscate해서 출력한다. 현재 `VOYAGE_API_KEY`는 sensitive key 목록에 없다.
- [pyproject.toml](/Users/seokhyung/Projects/AI-Related/zotero-mcp/pyproject.toml)
  `semantic` optional dependency set에는 `chromadb`, `sentence-transformers`, `openai`, `google-genai`, `tiktoken`만 포함되어 있다.
- [tests/test_semantic_search_quality.py](/Users/seokhyung/Projects/AI-Related/zotero-mcp/tests/test_semantic_search_quality.py)
  semantic search quality와 provider-specific behavior를 mock 중심으로 검증하는 기존 테스트 모음이 있다. Voyage 관련 테스트를 여기에 붙이는 것이 자연스럽다.

현재 verified handout 기준 목표 상태는 다음과 같다.

- Zotero read path: local API
- Zotero write path: Web API
- semantic embedding: Voyage API
- semantic reranking: Voyage API

## 제약사항

- 저장소 산출물과 코드 주석, docstring, 문서는 English-first 규칙을 따라야 한다.
- chat은 한국어로 유지하되 spec 안의 technical term, file path, class name, function name은 English를 유지한다.
- repo rule상 단일 사용 logic은 불필요하게 helper function으로 분리하지 않는다.
- 기존 MCP tool surface와 result schema는 backward compatible해야 한다.
- 기존 `openai`, `gemini`, local `cross-encoder` behavior를 깨지 않아야 한다.
- networkless test 환경에서도 검증 가능하도록 live Voyage API 의존 테스트는 피한다.
- Chroma collection incompatibility는 silent corruption보다 explicit reset이 우선이다.
- 문서화되지 않았거나 확인되지 않은 `voyageai` SDK surface는 이번 patch에 포함하지 않는다.

## 고려한 접근

### Option 1. Surgical provider patch

- 장점: 현재 구조를 최대한 유지하면서 필요한 provider만 추가할 수 있다.
- 장점: MCP tool surface와 result schema를 건드리지 않아 regression risk가 낮다.
- 장점: brownfield persistence와 setup flow를 크게 흔들지 않는다.
- 단점: provider-specific branching이 조금 늘어난다.
- 채택 여부: 채택

### Option 2. Embedding/reranker abstraction을 먼저 대규모로 재설계

- 장점: 장기적으로 provider 추가가 더 쉬워질 수 있다.
- 단점: 이번 patch 목적 대비 scope가 커지고 regression surface가 넓어진다.
- 단점: brownfield 코드와 테스트를 크게 흔들 가능성이 있다.
- 채택 여부: 미채택

### Option 3. Voyage를 OpenAI-compatible hack으로 우회 연결

- 장점: 초기 코드 변경량이 가장 적을 수 있다.
- 단점: query/document asymmetric retrieval과 reranker support를 정확히 구현할 수 없다.
- 단점: handout가 지적한 품질 문제를 그대로 남긴다.
- 채택 여부: 미채택

## 최종 설계

### 구조 변경

- `src/zotero_mcp/chroma_client.py`
  `VoyageEmbeddingFunction`를 추가한다.
- `src/zotero_mcp/semantic_search.py`
  `VoyageReranker`를 추가하고 `_load_reranker_config()`와 `_get_reranker()`를 provider-aware하게 바꾼다.
- `src/zotero_mcp/setup_helper.py`
  interactive semantic setup와 saved semantic config writing에 Voyage choice를 추가한다.
- `src/zotero_mcp/cli.py`
  `VOYAGE_API_KEY`를 obfuscation 대상에 추가한다.
- `pyproject.toml`
  `semantic` extra에 `voyageai`를 추가한다.
- `tests/test_semantic_search_quality.py`
  Voyage embedding/query/reranker behavior test를 추가한다.
- 필요하면 `tests/test_voyage_config.py`
  setup/config/CLI obfuscation만 따로 검증하는 작은 테스트 파일을 추가한다.

### 데이터 흐름 또는 제어 흐름

1. 사용자가 `ZOTERO_EMBEDDING_MODEL=voyage` 또는 setup-generated config를 통해 Voyage를 선택한다.
2. `create_chroma_client()`가 `embedding_model == "voyage"`를 감지하고 Voyage embedding config를 조합한다.
3. `ChromaClient._create_embedding_function()`가 `VoyageEmbeddingFunction`를 생성한다.
4. indexing 시 `VoyageEmbeddingFunction.__call__()`은 document batch에 대해 `vo.embed(..., input_type="document")`를 호출한다.
5. search 시 `ChromaClient.search()`는 custom embedding path를 유지하되 Voyage도 `embed_query()` 대상에 포함시켜 `vo.embed(..., input_type="query")`를 사용한다.
6. `ZoteroSemanticSearch.search()`는 reranker enabled 여부를 보고 기존과 동일하게 candidate over-fetch를 수행한다.
7. reranker provider가 `voyage`이면 `VoyageReranker.rerank()`가 `vo.rerank(query, documents, model=..., top_k=...)`를 호출하고 ranked index list만 반환한다.
8. ranked index는 기존처럼 `ids`, `distances`, `documents`, `metadatas` ordering에만 반영된다.
9. enrichment 후 반환 schema는 변경하지 않는다.

### 인터페이스 계약

#### Embedding provider selection

- `semantic_search.embedding_model`
  allowed values에 `"voyage"`를 추가한다.
- `semantic_search.embedding_config`
  Voyage 선택 시 아래 key를 허용한다.
  - `model_name`: default `"voyage-4-large"`
  - `api_key`: optional, env fallback 가능
  - `truncation`: optional, default `true`

#### Environment variables

- `VOYAGE_API_KEY`
- `VOYAGE_EMBEDDING_MODEL`

이 patch에서는 embedding env surface만 추가한다. reranker용 별도 env var는 도입하지 않는다.

#### Reranker config

- `semantic_search.reranker`
  - `enabled`: bool
  - `provider`: default `"cross-encoder"` when omitted
  - `model`: default `"cross-encoder/ms-marco-MiniLM-L-6-v2"` or `"rerank-2.5"` when `provider == "voyage"`
  - `candidate_multiplier`: default `3`
  - `truncation`: optional, default `true`

Voyage reranker의 runtime source of truth는 `semantic_search.reranker` JSON config다. Credential은 reranker config에 중복 저장하지 않고 `semantic_search.embedding_config.api_key` 또는 `VOYAGE_API_KEY`를 재사용한다.

#### Result schema

- semantic search response schema는 변경하지 않는다.
- reranking은 ordering만 바꾸고 별도 `rerank_score` field는 추가하지 않는다.

### Brownfield 통합 방식

- `VoyageEmbeddingFunction`는 기존 `OpenAIEmbeddingFunction`/`GeminiEmbeddingFunction`과 같은 contract를 따라야 한다.
  - `name()`
  - `get_config()`
  - `build_from_config()`
  - `__call__()`
  - `embed_query()`
  - `truncate()`
- `ChromaClient.search()`의 custom embedding tuple에 `VoyageEmbeddingFunction`를 포함시켜야 한다.
- reranker path는 새 abstraction을 만들기보다 기존 `CrossEncoderReranker` 옆에 `VoyageReranker`를 추가하는 수준으로 유지한다.
- setup helper는 기존 `openai`/`gemini` flow와 동일한 UX pattern을 따르되, Voyage-specific question만 추가한다.
- reranker setup는 preserve-first rule을 따른다. 기존 cross-encoder config가 있으면 explicit user opt-in 없이는 덮어쓰지 않는다.

## 구현 세부사항

### 1. `VoyageEmbeddingFunction`

- `voyageai.Client`를 사용한다.
- default model은 `"voyage-4-large"`다.
- `__call__()`은 `input_type="document"`를 사용한다.
- `embed_query()`는 `input_type="query"`를 사용한다.
- `truncation` config는 Voyage API 호출에 그대로 전달한다.
- batch input은 document list 단위로 전달하되, provider limit에 대비해 기존 batch processing 흐름과 충돌하지 않도록 class 내부에서 list input을 그대로 처리한다.
- 이번 patch에서는 separate local token-budget policy를 새로 도입하지 않는다.
  - local `truncate()`는 no-op으로 두거나, existing flow를 깨지 않는 최소 pass-through contract만 유지한다.
  - 실제 길이 제한 enforcement는 Voyage API의 `truncation=true`를 source of truth로 사용한다.

### 2. Chroma collection compatibility reset

- 현재 모델명만 비교하는 best-effort reset logic를 provider-aware signature 비교로 확장한다.
- primary source of truth는 Chroma가 저장한 `config_json_str` 안의 `embedding_function` persisted config다.
- custom embedding class는 기존 pattern대로 `name()`과 `get_config()`를 제공하고, reset logic는 이 persisted 값을 읽어 normalized compatibility signature를 만든다.
- compatibility signature 최소 항목:
  - embedding provider name
  - `model_name`
  - `truncation`
- 이번 iteration에서는 Voyage custom `output_dimension` feature를 user-facing하게 열지 않으므로 compatibility signature에는 포함하지 않는다.
- fallback order는 다음과 같다.
  1. `config_json_str`에서 provider-aware signature 읽기
  2. 기존 stored `model_name` best-effort check
  3. collection create/query 시점의 existing embedding-conflict exception 기반 reset
- 이 설계는 sidecar file을 새로 만들지 않고 현재 brownfield persistence pattern 안에서 끝낸다.

### 3. `create_chroma_client()` config merge

- precedence는 기존 provider들과 동일하게 유지한다.
  - explicit config value
  - environment variable
  - hardcoded default
- `embedding_model == "voyage"`일 때:
  - `api_key`는 `VOYAGE_API_KEY`
  - `model_name`은 `VOYAGE_EMBEDDING_MODEL` 또는 `"voyage-4-large"`
  - `truncation`은 config value가 없으면 `true`
- config merge는 기존 OpenAI/Gemini path처럼 partial-fill 방식이어야 하며, 이미 config file에 들어 있는 값을 env가 덮어쓰지 않아야 한다.

### 4. `VoyageReranker`

- `semantic_search.py`에 `VoyageReranker` class를 추가한다.
- public method는 기존 reranker와 맞춘 `rerank(query: str, documents: list[str], top_k: int) -> list[int]` 형태를 유지한다.
- 내부에서는 `voyageai.Client.rerank(...)`를 호출하고, 반환 결과에서 document index ordering만 추출한다.
- default model은 `"rerank-2.5"`다.
- `truncation` config를 API call에 전달한다.
- credential resolution은 다음 precedence를 따른다.
  1. `semantic_search.embedding_config.api_key`
  2. `VOYAGE_API_KEY`
- 이번 patch에서는 reranker-specific env surface를 추가하지 않는다.

### 5. Reranker config loading

- `_load_reranker_config()` default config는 backward compatibility를 위해 local cross-encoder를 기본값으로 유지한다.
- `provider` key가 없으면 기존 config는 `"cross-encoder"`로 해석한다.
- `_get_reranker()`는 다음 규칙을 따른다.
  - `enabled == false`면 `None`
  - `provider == "cross-encoder"`면 기존 `CrossEncoderReranker`
  - `provider == "voyage"`면 `VoyageReranker`
  - unknown provider면 clear `ValueError`

### 6. Setup helper UX

- semantic embedding model menu에 Voyage를 추가한다.
- Voyage 선택 시 다음을 묻는다.
  - embedding model name, default `"voyage-4-large"`
  - API key
- reranking은 setup에서 optional question으로 노출한다.
  - existing reranker config가 있으면 현재 `enabled/provider/model` summary를 먼저 보여준다.
  - prompt는 preserve-first로 설계한다.
  - 사용자가 명시적으로 변경을 원할 때만 reranker config를 덮어쓴다.
  - Voyage reranker를 enable하는 경우 `semantic_search.reranker`에만 `provider = "voyage"`, `model = "rerank-2.5"`, `candidate_multiplier = 3`, `truncation = true`를 저장한다.
- 기존 semantic config가 있을 때는 Voyage config와 reranker config summary를 보여주되, cross-encoder config는 explicit opt-in 없이 바꾸지 않는다.

### 7. CLI and env display

- `cli.py`의 sensitive key 목록에 `VOYAGE_API_KEY`를 추가한다.
- `setup-info` 출력은 기존 single-line env JSON shape를 유지한다.
- `setup_helper.update_claude_config()`와 standalone config writer는 Voyage embedding choice만 env에 반영한다.
- Voyage reranker choice는 env가 아니라 saved JSON config의 `semantic_search.reranker`에만 반영한다.

### 8. Documentation scope

- README 변경은 최소 범위로 제한한다.
- semantic search section에 지원 embedding provider 목록에 Voyage를 추가한다.
- semantic dependency 설명에 Voyage를 포함한다.
- environment variable 목록에 Voyage embedding 관련 항목을 추가한다.
- reranker 설정은 config-file based라는 점을 예시 JSON과 함께 짧게 설명한다.
- rerank ordering은 result schema를 바꾸지 않는다는 점을 짧게 명시한다.

## Acceptance Criteria

- `semantic_search.embedding_model = "voyage"` 설정 시 `ChromaClient`가 HuggingFace fallback이 아니라 `VoyageEmbeddingFunction`를 생성한다.
- document indexing은 Voyage `input_type="document"`를 사용한다.
- query search는 Voyage `input_type="query"`를 사용한다.
- reranker config에서 `provider = "voyage"`와 `enabled = true`를 주면 Voyage reranker가 result ordering에 반영된다.
- semantic search response schema는 기존과 동일하고 `rerank_score` field는 추가되지 않는다.
- `VOYAGE_API_KEY`가 `setup-info` 출력에서 obfuscate된다.
- `zotero-mcp setup` 또는 `setup --semantic-config-only`에서 Voyage choice를 설정할 수 있다.
- embedding provider 또는 Voyage model compatibility가 바뀌면 Chroma collection reset path가 동작한다.
- existing reranker config는 setup에서 explicit opt-in 없이는 유지된다.
- reranker config의 runtime source of truth는 env가 아니라 `semantic_search.reranker` JSON config다.
- 기존 OpenAI, Gemini, local cross-encoder behavior는 회귀하지 않는다.

## Edge Cases and Failure Modes

- `VOYAGE_API_KEY`가 없는데 Voyage provider를 선택한 경우 clear error를 낸다.
- reranker provider가 unknown string이면 silent fallback하지 말고 error를 낸다.
- reranker enabled인데 document candidate list가 비어 있으면 rerank call 없이 기존 빈 결과를 반환한다.
- candidate count가 `top_k`보다 작아도 safe slicing이 유지되어야 한다.
- config file에 reranker `provider`가 없는 legacy case는 existing cross-encoder behavior로 유지해야 한다.
- setup helper는 existing cross-encoder reranker config를 explicit user opt-in 없이 Voyage로 치환하면 안 된다.
- Chroma stored metadata를 읽지 못해도 semantic search가 완전히 막히지 않도록 기존 fallback reset logic를 유지한다.
- live API test가 없는 환경에서도 mocks만으로 provider-specific branch를 검증할 수 있어야 한다.

## Verification Plan

- Unit tests
  - `tests/test_semantic_search_quality.py`에 Voyage embedding document/query mode test 추가
  - 같은 파일에 Voyage reranker ordering test 추가
  - 같은 파일 또는 새 테스트 파일에 `create_chroma_client()` Voyage env merge precedence test 추가
  - 같은 파일 또는 새 테스트 파일에 `VOYAGE_API_KEY` obfuscation test 추가
  - legacy reranker config에서 `provider`가 없을 때 cross-encoder fallback이 유지되는지 test 추가
  - missing `VOYAGE_API_KEY` error path test 추가
  - setup helper가 existing reranker config를 preserve하는지 test 추가
- Static checks
  - `ruff check src tests`
- Targeted test run
  - `pytest tests/test_semantic_search_quality.py`
  - setup/CLI 전용 테스트를 새 파일로 분리했다면 그 파일도 함께 실행
- Optional manual validation
  - local config에 Voyage provider를 넣고 `zotero-mcp setup-info` 출력 shape 확인
  - live key가 있다면 small-scope `update-db --limit 5`와 semantic search를 manual smoke test로 확인

## Risks and Follow-ups

- `voyageai` SDK constructor surface가 문서와 실제 설치 버전 사이에서 다를 수 있다.
- Chroma stored embedding config introspection은 provider별로 brittle할 수 있다.
- setup helper에 reranker prompt를 추가하면 interactive flow가 길어질 수 있다.
- Follow-up candidate:
  - custom `output_dimension` support
  - rerank relevance를 별도 field로 노출하는 schema evolution
  - live integration smoke test harness

## Open Questions

- 없음. `critic` review에서 제기된 `base_url`와 reranker env ambiguity는 이번 revised spec에서 scope 밖으로 정리했다.

## Next Step

이 revised spec을 사용자에게 제시하고, approval 또는 targeted change 요청을 받는다.
