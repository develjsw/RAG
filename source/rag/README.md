# RAG 실습 노트북

`source/rag/chroma/` 아래를 **두 개의 축**으로 나눠 정리함

- `provider/` — **provider 축**: 파이프라인은 그대로 두고 LLM·임베딩 provider만 OpenAI ↔ Upstage로 교체
- `preprocessing/` — **전처리 축**: 입력 문서(docx) 전처리 방식만 바꿔가며 retrieval 정확도 비교

---

## provider/ — OpenAI vs Upstage

원본 `tax.docx`(소득세 표가 이미지로 박혀 있음) 기반의 기본 RAG<br>
체인은 강의 그대로 `RetrievalQA` 사용

- [OpenAI](chroma/provider/langchain-chroma-rag-openai.ipynb) — `OpenAIEmbeddings` + `ChatOpenAI` / 벡터 DB `./chroma`
- [Upstage](chroma/provider/langchain-chroma-rag-upstage.ipynb) — `UpstageEmbeddings`(solar-embedding-1-large) + `ChatUpstage` / 벡터 DB `./chroma-upstage` (임베딩 벡터 차원이 OpenAI와 달라 DB를 따로 둠)

---

## preprocessing/ — 전처리 비교

전처리 효과만 보려고 **나머지 조건은 모두 똑같이 고정함** (통제변수)

| 고정 항목 | 값 |
|-----------|-----|
| chain | `create_retrieval_chain` + `create_stuff_documents_chain` |
| retriever | `as_retriever(k=10)` |
| prompt | `langchain-ai/retrieval-qa-chat` |
| LLM | `gpt-4o-mini` |
| 임베딩 | `text-embedding-3-large` |

→ 노트북끼리 **다른 점은 입력 docx 전처리 방식 하나뿐**

- [01-image](chroma/preprocessing/01-image/langchain-chroma-rag-openai.ipynb) — **전처리 전 기준점**: 원본 `tax.docx`는 소득세 표가 **image**라 `Docx2txtLoader`가 못 읽고 누락 → 표가 필요한 질문에서 실패
- [02-table](chroma/preprocessing/02-table/langchain-chroma-rag-openai.ipynb) — docx 파일 안에 이미지 표를 **table 형식** 으로 변환, 다만 `Docx2txtLoader`가 표 셀을 한 줄씩 펼쳐버려 GPT가 표 구조를 이해하기 어려움
- [03-markdown](chroma/preprocessing/03-markdown/langchain-chroma-rag-openai.ipynb) — 표를 **markdown 형식(`| ... |`)** 으로 변환, 행 단위로 묶여 LLM이 가장 잘 이해함
- [04-keyword](chroma/preprocessing/04-keyword/langchain-chroma-rag-openai.ipynb) — 03의 markdown docx에 **keyword dictionary**를 더해 질문 용어를 보정(직장인 → 거주자)한 뒤 검색, `dictionary_chain`(LCEL) → `retrieval_chain` 연결

### markdown > table

같은 표라도 담는 형식에 따라 LLM 이해도가 갈림

- docx table 표: `Docx2txtLoader`가 셀 구분 기호(`|`) 없이 한 줄씩 펼침 → 과세표준과 세율의 짝이 흐려져 LLM이 헷갈림
- docx markdown 표: `| 5,000만원 초과 8,800만원 이하 | 624만원 + 24% |` 처럼 행 단위로 묶여 LLM이 읽기 쉬움
- 검색 순위는 둘 다 세율표가 유사도 약 7위로 비슷 → 차이는 **검색 결과를 LLM이 이해해 답하는 단계**에서 벌어짐

### k=10 과 키워드 사전(04)

- `k=10`이면 정답 세율표가 유사도 약 7위여도 상위 10개 안에 들어와, 키워드 보정 없이도 답이 나올 수 있음
- 키워드 사전은 질문 용어를 문서 용어에 맞춤(직장인 → 거주자) → 실제로 보정 후 세율표 순위가 약 7위에서 **1위**로 올라옴, 작은 k에서도 정확해짐
- 그래서 04는 01–03의 단순 답변 생성은 생략하고 **키워드 보정 → 최종 답변**만 남김
