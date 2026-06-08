# LangChain 패키지 구조 (2026-06 기준)

> 왜 여러 패키지로 쪼개졌는지 + 무엇을 설치해야 하는지 정리<br>
> 설치·단계별 사용법은 [langchain-rag-pipeline.md](langchain-rag-pipeline.md), 공식문서 링크는 [langchain-links.md](langchain-links.md) 참고

---

## 왜 쪼개졌나 — 패키지 진화

### Phase 1 — 거대 단일 패키지 (2022.10 ~ 2023.12)
- `pip install langchain` 하나로 LLM 연동·문서 로더·벡터스토어·체인·에이전트 전부 포함
- Harrison Chase가 2022.10 개인 프로젝트로 시작
- 한계: 외부 SDK(OpenAI 등) 변경 시 전체가 흔들림 / `0.0.x` breaking change 난무 / `import langchain` 한 번에 의존성 수백 개 / 유지보수 불가 수준으로 비대해짐

### Phase 2 — 3개로 분리 (2023.12 발표 → 2024.01 v0.1.0)
| 패키지 | 역할 |
|--------|------|
| `langchain-core` | 핵심 추상화만 (`BaseMessage`, `Runnable`, `Document` 등) |
| `langchain-community` | 3rd-party 통합 전부 (로더·벡터스토어·임베딩) |
| `langchain` | 체인·에이전트 등 고수준 조합 도구 |

분리 이유
- 버전 안정성 — core는 엄격 관리, 외부 연동은 따로 버전 관리
- 의존성 분리 — OpenAI SDK가 바뀌어도 `langchain-openai`만 업데이트하면 됨
- 유지보수 책임 분산 — 각 통합 제공사가 직접 패키지 관리

### Phase 3 — 전용 파트너 패키지 확산 (2024 ~ 2025)
- `langchain-community` 안에 있던 통합들이 하나씩 독립 패키지로 분리 (`langchain-openai`, `langchain-anthropic`, `langchain-chroma`, `langchain-google-*` …)
- 각 회사·커뮤니티가 직접 관리

### Phase 4 — langchain-community 아카이브 (2026.05.26)
- `langchain-community` 저장소가 아카이브됨 → 신규 기능·버그픽스 없이 레거시화

---

## 2026-06 권장 구조 (이 프로젝트 설치본)

| 용도 | 패키지 | 버전 | 상태 |
|------|--------|------|------|
| 핵심 추상화 (모두가 공유, 자동 설치) | `langchain-core` | 1.4.1 | ✅ 필수 |
| 고수준 체인·에이전트·RAG | `langchain` | 1.3.4 | ✅ 메인 |
| 레거시 체인 보관 (`RetrievalQA`, `create_retrieval_chain`) | `langchain-classic` | 1.0.7 | ⚠️ 레거시 체인 전용 |
| OpenAI LLM + 임베딩 | `langchain-openai` | 1.2.2 | ✅ OpenAI 관리 |
| Upstage LLM + 임베딩 | `langchain-upstage` | 0.7.7 | ✅ Upstage 관리 |
| Chroma 벡터스토어 | `langchain-chroma` | 1.1.0 | ✅ Chroma 관리 |
| 텍스트 분할기 | `langchain-text-splitters` | 1.1.2 | ✅ 별도 관리 |
| 3rd-party 잡다한 통합 | `langchain-community` | 0.4.2 | ❌ archived (레거시) |

> `langchain-classic`: v1에서 `RetrievalQA`·`create_retrieval_chain`·`create_stuff_documents_chain` 등 레거시 체인이 `langchain.chains` → `langchain_classic.chains`로 이동함 (자세한 v1 변경점은 루트 `CLAUDE.md`)

---

## langchain 하나로 퉁칠 수 없는 이유

- `langchain` 1.x는 **조합 도구(체인·에이전트)만** 담음 — LLM·벡터스토어 **구현체는 미포함**

```python
# langchain만 설치하면 아래가 전부 ImportError
from langchain_openai import ChatOpenAI        # langchain-openai 필요
from langchain_chroma import Chroma            # langchain-chroma 필요
from langchain_text_splitters import RecursiveCharacterTextSplitter  # langchain-text-splitters 필요
```

- 설계 철학이 **"core는 작게, 구현체는 각자 패키지로"** 로 굳어짐

---

## 전체 생태계 맵 (참고)

### LLM / 임베딩 파트너
| 패키지 | 제공사 |
|--------|--------|
| `langchain-openai` | OpenAI (GPT, Embeddings) |
| `langchain-anthropic` | Anthropic (Claude) |
| `langchain-google-genai` | Google Gemini |
| `langchain-google-vertexai` | Google Vertex AI |
| `langchain-mistralai` | Mistral |
| `langchain-groq` | Groq |
| `langchain-ollama` | Ollama (로컬 LLM) |
| `langchain-huggingface` | HuggingFace |
| `langchain-aws` | AWS Bedrock |
| `langchain-azure-ai` | Azure AI |
| `langchain-xai` | xAI (Grok) |
| `langchain-deepseek` | DeepSeek |

### Vector DB 파트너
| 패키지 | 제공사 |
|--------|--------|
| `langchain-chroma` | Chroma (로컬·경량) |
| `langchain-pinecone` | Pinecone (클라우드) |
| `langchain-weaviate` | Weaviate |
| `langchain-qdrant` | Qdrant |
| `langchain-milvus` | Milvus |
| `langchain-mongodb` | MongoDB Atlas |
| `langchain-elasticsearch` | Elasticsearch |
| `langchain-redis` | Redis |
| `langchain-pgvector` | PostgreSQL pgvector |

### 확장 프레임워크
| 패키지 | 역할 |
|--------|------|
| `langgraph` (1.2.4) | 에이전트 오케스트레이션 (상태 기반 워크플로우) |
| `langsmith` (0.8.9) | 모니터링·디버깅·평가 플랫폼 |
| `deepagents` | 고수준 에이전트 (플래닝·파일시스템 등 내장) |
| `agentevals` | 에이전트 평가 도구 |

---

## 이 프로젝트 적용

- `langchain-community`에서 실제로 쓰는 건 **`Docx2txtLoader` 단 1개** (전용 패키지 없이 아직 community에만 존재)
- 나머지는 모두 전용 패키지로 import (`langchain-openai` / `langchain-chroma` / `langchain-text-splitters`)
- 최소 설치

```bash
pip install langchain langchain-openai langchain-chroma langchain-text-splitters langchain-community langchain-classic docx2txt chromadb
```

- `langchain-community` — `Docx2txtLoader` 때문에 당장은 포함 (archived라 장기적으론 불안)
- `langchain-classic` — `RetrievalQA` / `create_retrieval_chain` 때문에 포함
- 장기 대안: 로더를 **Docling / Unstructured**로 교체하면 `langchain-community` 완전 제거 가능
