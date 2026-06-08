# RAG 실습 노트북

`source/rag/chroma/` 아래를 **두 개의 축**으로 분리해 정리함

- `provider/` — **provider 축**: 동일 파이프라인에서 LLM/임베딩 provider만 OpenAI ↔ Upstage 교체
- `preprocessing/` — **전처리 축**: docx 전처리 방식만 바꿔가며 retrieval 정확도 비교

---

## provider/ — OpenAI vs Upstage

원본 `tax.docx`(이미지 표) 기반 기본 RAG<br>
강의 그대로 `RetrievalQA` 사용

- [OpenAI](chroma/provider/langchain-chroma-rag-openai.ipynb) — `OpenAIEmbeddings` + `ChatOpenAI` / `./chroma`
- [Upstage](chroma/provider/langchain-chroma-rag-upstage.ipynb) — `UpstageEmbeddings`(solar-embedding-1-large) + `ChatUpstage` / `./chroma-upstage` (임베딩 차원이 OpenAI와 달라 DB 분리)

---

## preprocessing/ — 전처리 비교

**전처리 효과만 보기 위해 나머지 변수를 모두 고정함**(통제변수)

| 고정 항목 | 값 |
|-----------|-----|
| chain | `create_retrieval_chain` + `create_stuff_documents_chain` |
| retriever | `as_retriever(k=10)` |
| prompt | `langchain-ai/retrieval-qa-chat` |
| LLM | `gpt-4o-mini` |
| 임베딩 | `text-embedding-3-large` |

→ 노트북 간 **유일한 차이는 입력 docx(전처리 방식)** 뿐

- [01-image](chroma/preprocessing/01-image/langchain-chroma-rag-openai.ipynb) — **전처리 전 baseline**: 원본 `tax.docx`의 소득세 표가 **이미지**라 `Docx2txtLoader`가 못 읽어 누락 → 표 기반 질문 실패
- [02-table](chroma/preprocessing/02-table/langchain-chroma-rag-openai.ipynb) — 표를 **table**로 변환한 docx, docx 진짜 표라 `Docx2txtLoader`가 셀을 평탄화 → GPT가 표 구조 이해 어려움
- [03-markdown](chroma/preprocessing/03-markdown/langchain-chroma-rag-openai.ipynb) — 이미지를 **markdown**으로 변환한 docx, 마크다운 표(`| ... |`)라 LLM이 더 잘 이해함
- [04-keyword](chroma/preprocessing/04-keyword/langchain-chroma-rag-openai.ipynb) — markdown docx + **키워드 사전**으로 질문 보정(직장인→거주자) 후 RAG, `dictionary_chain`(LCEL) → `retrieval_chain` 연계

### markdown vs table — markdown이 나음

`Docx2txtLoader`는 docx 진짜 표를 파이프 없이 셀 단위로 평탄화 → 과세표준↔세율 대응이 흐려져 LLM이 헷갈림<br>
markdown 표(`| ... |`)는 행 단위로 묶여 LLM이 잘 이해함<br>
검색 순위는 둘 다 세율표 ~7위로 비슷 — 차이는 **가져온 chunk를 LLM이 이해하는 단계**에서 큼

### k=10 과 키워드 사전(04)

`k=10`이면 "직장인↔거주자" 불일치에도 ~7위 세율표가 컷에 들어와 보정 없이도 답이 나올 수 있어, 키워드 효과가 두드러지지 않음<br>
키워드 사전의 본질은 질문 용어를 문서 용어로 맞춰 **검색 순위를 끌어올리는 것**(작은 k에서도 정확)임<br>
04는 01~03 전제라 보정 전 비교는 생략하고 **보정 → 최종 답변**만 남김 — 효과를 크게 보려면 04에서만 k를 낮춰 실험
