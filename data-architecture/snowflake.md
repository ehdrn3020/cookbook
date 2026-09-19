# SnowFlake

> 작성일: 2026-09-19

| 사용자 질문 / 작업 | 주로 쓰는 기능 |
| --- | --- |
| `"어제 방송시간 TOP 10 알려줘"` | **Cortex Analyst** → Text-to-SQL |
| `"방송시간 집계 장애가 발생하면 어떻게 복구하지?"` | **Cortex Search** → 운영문서 / Runbook RAG |
| `"A 로그명세에서 이 필드값 의미와 언제 추가됐어?"` | **Cortex Search** → 로그 명세 / 변경이력 문서 검색 |
| `"이 PDF 장애보고서에서 장애일시, 원인, 조치내용을 추출해줘"` | **Document AI / AI_EXTRACT** → 문서에서 필드 구조화 추출 |
| `"이 계약서/보고서를 읽어서 표와 주요 항목을 뽑아 테이블에 저장해줘"` | **Document AI / AI_PARSE_DOCUMENT + AI_EXTRACT** |
| `"TOP 10 중 장애 이력이 있는 방송만 찾아서 대응방법까지 알려줘"` | **Cortex Agents** → Analyst + Search 조합 |
| `"장애보고서 PDF에서 장애 원인을 추출하고, 실제 장애 이력과 비교해서 대응방법까지 알려줘"` | **Cortex Agents** → Document AI + Analyst + Search 조합 |

## Cortex Analyst

- 동의어(synonyms) : 명세값의 동의어
- Verified Query : 검증된 여러 쿼리로 정확성 높임
- Custom Instructions : 회사만의 SQL 규칙

```javascript
① Table / Column 명세
       ↓
② 관계
       ↓
③ Dimension / Fact / Metric 정의
       ↓
④ 회사 용어 / Synonym
       ↓
⑤ Verified Question + SQL
       ↓
⑥ 회사 SQL 규칙 / Custom Instructions
```

## Cortex Search

- 공통검색스키마 변환 : AI_PARSE_DOCUMENT → chunking → Cortex Search
- AI_EXTRACT: 메타데이터를 AI로 추출, extraction score 기능
- 여러 사내 시스템에서 데이터를 가져와서 회사 공통 metadata schema로 정규화하는 ingestion 계층은 여전히 설계가 필요
