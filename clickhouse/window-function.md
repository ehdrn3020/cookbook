# Window Function

> 작성일: 2026-03-16

```javascript
SELECT
    *,
    row_number() OVER (ORDER BY salary DESC) AS rowNUM
FROM employee
ORDER BY position
LIMIT 10;
>>> Result
| name | position | salary | rowNUM |
| ---- | -------- | ------ | ------ |
| Kim  | Manager  | 9000   | 1      |
| Park | Analyst  | 8000   | 2      |
| Lee  | Engineer | 7000   | 3      |

SELECT
    player,
    team,
    salary,
    avg(salary) OVER (PARTITION BY team) AS t_avg_salary,
    sum(salary) OVER (PARTITION BY team) AS t_total_salary,
    row_number() OVER (PARTITION BY team ORDER BY salary DESC) AS t_salary_rank,
    rand() AS random_value
FROM salaries
ORDER BY team, team_salary_rank;
>>> Result
| player | team | salary | t_avg_salary | t_total_salary | t_salary_rank | random_value |
| ------ | ---- | -----: | -----------: | -------------: | ------------: | -----------: |
| Park   | A    |    300 |          200 |            600 |             1 |      3819281 |
| Lee    | A    |    200 |          200 |            600 |             2 |      9182731 |
| Kim    | A    |    100 |          200 |            600 |             3 |      5523412 |
| Jung   | B    |    250 |          200 |            400 |             1 |      7123456 |
| Choi   | B    |    150 |          200 |            400 |             2 |      1287345 |
```
