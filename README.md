# Handoff AI

업무 문서를 업로드하면 텍스트를 추출·인덱싱하고, watsonx.ai와 Qdrant를 이용해 근거 기반 질문 답변을 제공하는 인수인계 지원 MVP입니다.

## 현재 기능

- 여러 문서 업로드: PDF, DOCX, XLSX/XLSM, TXT, Markdown, CSV
- 비동기 문서 처리 상태: 대기, 처리 중, 완료, 실패
- Qdrant 기반 근거 검색과 watsonx.ai 기반 한국어 Q&A
- 답변별 출처 파일과 근거 원문 확인

GitHub·Slack·Notion 커넥터와 자동 인수인계 문서 생성은 아직 구현 범위에 포함하지 않습니다.

## 구성

- `backend/`: FastAPI, 문서 처리 워커, Qdrant·watsonx.ai 연동
- `web/`: Handoff AI 웹 UI
- `docs/`: 아키텍처 문서

## 로컬 실행

1. `.env.example`을 참고해 프로젝트 루트에 `.env`를 만듭니다. 실제 API 키와 데이터베이스 비밀번호는 Git에 올리지 않습니다.
2. PostgreSQL과 Qdrant를 실행하고 `backend/scripts/initialize_storage.py`로 테이블과 컬렉션을 준비합니다.
3. API와 워커를 각각 실행합니다.

```bash
cd backend
.venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000

# 별도 터미널
.venv/bin/python scripts/run_worker.py
```

4. 웹 앱을 실행합니다.

```bash
cd web
npm install
npm run dev
```

웹 앱은 `http://localhost:3000`, API 문서는 `http://localhost:8000/docs`에서 확인할 수 있습니다.

## API

모든 자료·질문 API는 `X-Workspace-ID` 헤더로 작업 공간을 구분합니다.

- `POST /api/v1/ingestion/uploads`
- `GET /api/v1/ingestion/jobs/{job_id}`
- `GET /api/v1/ingestion/sources`
- `POST /api/v1/rag/questions`

자세한 요청·응답 형식은 FastAPI OpenAPI 문서에서 확인할 수 있습니다.
