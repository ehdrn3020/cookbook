# SBT IntelliJ 환경

> 작성일: 2025-12-19

```javascript
my_project/
├─ build.sbt            ← ⭐ 핵심 (프로젝트 정의)
├─ project/             ← sbt 자체 설정
│  ├─ build.properties  ← sbt 버전
│  └─ plugins.sbt       ← sbt 플러그인
├─ src/
│  ├─ main/
│  │  ├─ scala/         ← ⭐ Scala 소스
│  │  ├─ java/          ← Java 소스 (선택)
│  │  └─ resources/     ← 설정 파일
│  └─ test/
│     ├─ scala/
│     └─ resources/
├─ target/              ← ⭐ 컴파일 결과물 (자동 생성)
├─ .bsp/                ← IDE 연동 정보
└─ .idea/               ← IntelliJ 설정
```

- File → Project Structure 탭에서 scala 환경 및 모듈 확인
- `spark.implicits._` 는 Spark Scala에서 "타입 기반 API(Dataset, DataFrame)"를 쓰기 위해
