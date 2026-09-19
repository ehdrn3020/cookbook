# Gradle Project 구조

> 작성일: 2025-12-28

```javascript
project-root                    ← 프로젝트 최상위 디렉터리
 ├─ build/                       ← Gradle 빌드 결과물 (자동 생성, 삭제 가능)
 │                                └─ 컴파일된 class, 리소스, JAR, 테스트 리포트
 ├─ gradle/                      ← Gradle Wrapper 관련 파일
 │                                └─ 팀원 간 동일한 Gradle 버전 보장
 ├─ src/                         ← 애플리케이션 소스 루트
 │   ├─ main/                    ← 실제 실행되는 애플리케이션 코드
 │   │   ├─ java/                ← Java 소스 코드 (main(), 서비스, 설정 등)
 │   │   └─ resources/           ← 설정 파일 및 정적 리소스 (yml, xml, log 설정)
 │   └─ test/                    ← 테스트 코드 영역
 │                                └─ 단위/통합 테스트 (빌드 결과물에 포함 안 됨)
 ├─ .gitignore                   ← Git에서 제외할 파일/디렉터리 목록
 ├─ build.gradle                 ← 빌드 스크립트 (의존성, 플러그인, Java 버전)
 ├─ gradlew                      ← Gradle Wrapper 실행 스크립트 (Linux/Mac)
 ├─ gradlew.bat                  ← Gradle Wrapper 실행 스크립트 (Windows)
 ├─ README.md                    ← 프로젝트 설명 및 사용 방법 문서
 └─ settings.gradle              ← 프로젝트 이름 및 멀티모듈 구성 정의
```
