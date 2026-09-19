# Projection

> 작성일: 2026-02-22

- 같은 테이블 안에 "미리 정렬·집계된 서브 테이블"을 숨겨 두고, 쿼리가 맞으면 자동으로 사용 (인덱스)
- aggregation projection을 가장 많이 사용
	- JOIN, 복잡한 CASE WHEN 은 불가능
- Projection은 "INSERT + MERGE 시점"에 원본과 동기화
- 예제

```javascript
/* Projection 추가 */
ALTER TABLE vod_log
ADD PROJECTION p_daily_content
(
    SELECT
        event_date,
        content_id,
        sum(view_cnt) AS view_cnt,
        sum(gift_cnt) AS gift_cnt
    GROUP BY
        event_date,
        content_id
);

/* 기존 데이터에 Projection 생성 */
OPTIMIZE TABLE vod_log FINAL; or
MATERIALIZE PROJECTION 사용

/* 
Projection  사용
Projection은 GROUP BY 결과를 가진 테이블이기 때문에
WHERE가 Projection의 GROUP BY 키에만 걸리면 자동 매핑
*/
SELECT
    event_date,
    content_id,
    sum(view_cnt),
    sum(gift_cnt)
FROM vod_log
WHERE event_date >= '2026-01-01'
GROUP BY event_date, content_id;

/* 사용 확인 */
EXPLAIN 접두사 query를 사용하여 실행계획 확인
```
