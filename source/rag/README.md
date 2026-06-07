# RAG 실습 노트북

## 기본 RAG (원본 docx)
`Docx2txtLoader`가 이미지 영역을 읽지 못해 해당 내용이 누락됨 → 이미지/표 기반 질문은 retrieval 실패

- [01-original-docx / OpenAI (3.2)](01-original-docx/langchain-chroma-rag-openai.ipynb)
- [01-original-docx / Upstage (3.2.1)](01-original-docx/langchain-chroma-rag-upstage.ipynb)

## Retrieval 효율 개선을 위한 데이터 전처리
문서를 전처리해 이미지·표 내용을 텍스트로 보존 → retrieval 정확도 개선

- [02-table / OpenAI](chroma/02-table/langchain-chroma-rag-openai.ipynb) — 표를 **table**로 변환한 docx (먼저 시도 / 평탄화로 GPT가 구조 이해 어려움)
- [03-markdown / OpenAI (3.2.2)](chroma/03-markdown/langchain-chroma-rag-openai.ipynb) — 이미지를 **markdown**으로 변환한 docx (개선 / LLM이 더 잘 이해)

### markdown vs table 변환 — 어느 쪽이 나은가

**결론: markdown 변환이 더 낫다.** 강의에서도 표(table)를 먼저 시도했으나, `Docx2txtLoader`가 docx 표를 파이프 없이 셀 단위로 평탄화해서 **GPT가 표 구조(과세표준↔세율 대응)를 이해하기 어려웠고**, markdown 표(`| ... |`)로 바꾸자 LLM이 더 잘 이해함

- 마크다운 표: `| 5,000만원 초과 8,800만원 이하 | 624만원 + 24% |` → 행 단위로 묶여 LLM이 읽기 쉬움
- docx 진짜 표: 셀이 줄단위로 펼쳐짐(파이프 없음) → 과세표준↔세율 대응이 흐려져 LLM이 헷갈림
- 검색(임베딩) 순위는 둘 다 비슷(세율표가 ~7번째). 차이는 **가져온 chunk를 LLM이 이해해 답하는 단계**에서 큼

두 방식 모두 "직장인↔거주자" 용어 불일치로 세율표 chunk 유사도가 낮은 문제는 동일 → 3.6 키워드 사전에서 해결

<!-- 3.6 작업 후 04로 여기에 추가 -->
