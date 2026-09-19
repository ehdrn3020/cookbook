# pip-tools

> 작성일: 2026-01-01

docker 빌드시 의존성 충돌 대비를 위한 pip-tools

- pip-tools는 requirements.in 패키지를 확인해 모든 하위 의존성을 살펴 서로 동시에 만족 가능한 버전 조합 계산
- 실행 예제
	- 가상 환경에 pip-tools 설치 → pip install pip-tools
	- requirements.in 작성
	- pip-compile 실행 → pip-compile requirements.in
	- requirements.txt.에 없는 패키지 제거 → pip-sync requirements.txt
