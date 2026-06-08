# DOCX → RAG 파이프라인 단계별 (Chroma 기준)

> 패키지가 왜 이렇게 나뉘는지는 [langchain-packages.md](langchain-packages.md)<br>
> 실행 가능한 전체 노트북: [source/rag/chroma](../../source/rag/chroma)(provider·preprocessing) / 처음부터 따라 만드는 템플릿: [langchain-rag-template.ipynb](../../source/rag/chroma/template/langchain-rag-template.ipynb)

---

## 설치

```bash
pip install langchain langchain-openai langchain-chroma langchain-text-splitters langchain-community langchain-classic docx2txt chromadb
```

---

## 1. Docx Loader

공식: https://reference.langchain.com/python/langchain-community/document_loaders/word_document/Docx2txtLoader

- `Docx2txtLoader` = 표준 (가볍고 의존성 단순, 대부분의 RAG에서 채택)
- `.doc` 지원이나 문서 구조(제목/본문 구분)가 필요할 때만 `UnstructuredWordDocumentLoader`

```python
from langchain_community.document_loaders import Docx2txtLoader

loader = Docx2txtLoader("./tax.docx")
raw_docs = loader.load()
```

---

## 2. Text Splitter

공식: https://reference.langchain.com/python/langchain-text-splitters/character/RecursiveCharacterTextSplitter

⚠️ `from langchain.text_splitter import ...` 는 deprecated → 반드시 `langchain_text_splitters`에서 import

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
docs = splitter.split_documents(raw_docs)
```

---

## 3. Embeddings (OpenAI)

공식: https://docs.langchain.com/oss/python/integrations/embeddings/openai

⚠️ `from langchain_community.embeddings import OpenAIEmbeddings` deprecated → `langchain_openai`에서 import

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-large")  # 최신 권장
# 비용 절감: model="text-embedding-3-small"
# text-embedding-ada-002 는 레거시 → 지양
```

---

## 4. Vector Store (Chroma)

공식: https://docs.langchain.com/oss/python/integrations/vectorstores/chroma

⚠️ `from langchain_community.vectorstores import Chroma` deprecated → `langchain_chroma` 사용

```python
from langchain_chroma import Chroma

# 최초 1회: 생성 + 영구 저장
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    persist_directory="./chroma_db",
    collection_name="chroma-tax",
)

# 재사용 (2회차부터): 저장된 DB 로드
# vectorstore = Chroma(
#     persist_directory="./chroma_db",
#     collection_name="chroma-tax",
#     embedding_function=embeddings,
# )
```

---

## 5. Retriever (유사도 검색)

공식: https://docs.langchain.com/oss/python/langchain/knowledge-base

```python
# similarity: 기본, 가장 유사한 k개 반환
retriever = vectorstore.as_retriever(search_type="similarity", search_kwargs={"k": 4})

# mmr: 다양성 + 유사도 균형 (실무 선호)
# retriever = vectorstore.as_retriever(search_type="mmr", search_kwargs={"k": 4, "fetch_k": 20})

# 점수 임계값 필터링
# retriever = vectorstore.as_retriever(search_type="similarity_score_threshold", search_kwargs={"score_threshold": 0.7})

result = retriever.invoke("찾고 싶은 내용")
```

---

## 6. 답변 생성 (RAG 체인)

⚠️ v1에서 `RetrievalQA`·`create_retrieval_chain` 등은 `langchain.chains` → `langchain_classic.chains`로 이동 (deprecated 아님, 단 `RetrievalQA`는 deprecated)

```python
from langsmith import Client
from langchain_openai import ChatOpenAI
from langchain_classic.chains.combine_documents import create_stuff_documents_chain
from langchain_classic.chains import create_retrieval_chain

# 프롬프트: v1에선 langchain.hub 제거 → langsmith로 pull (공개 프롬프트는 플래그 필요)
prompt = Client().pull_prompt("langchain-ai/retrieval-qa-chat", dangerously_pull_public_prompt=True)

llm = ChatOpenAI(model="gpt-4o-mini")
combine_docs_chain = create_stuff_documents_chain(llm, prompt)
rag_chain = create_retrieval_chain(retriever, combine_docs_chain)

answer = rag_chain.invoke({"input": "소득세 납세 의무"})  # 출력 키: input / context / answer
```

---

## deprecated → 권장 import 매핑

| 항목 | deprecated (경고/제거) | 현재 권장 |
|------|------------------------|-----------|
| Embeddings | `langchain_community.embeddings` | `langchain_openai` |
| Chroma | `langchain_community.vectorstores` | `langchain_chroma` |
| TextSplitter | `langchain.text_splitter` | `langchain_text_splitters` |
| 체인 (`RetrievalQA` 등) | `langchain.chains` | `langchain_classic.chains` |
| Prompt Hub | `from langchain import hub` | `langsmith.Client().pull_prompt(...)` |
| 임베딩 모델 | `text-embedding-ada-002` | `text-embedding-3-large` / `-small` |

---

전체를 한 셀씩 따라 실행하려면 → [langchain-rag-template.ipynb](../../source/rag/chroma/template/langchain-rag-template.ipynb)
