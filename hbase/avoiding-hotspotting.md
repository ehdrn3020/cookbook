# Hotspotting 피하기

> 작성일: 2025-11-24

## Hotspotting란?

- 클러스터 여러 노드 중 특정 노드(RegionServer) 몇 개에만 트래픽이 몰리는 현상
- Hotspotting을 피하기 위해 Row Key 디자인 필요
- 대표적인 나쁜 패턴 : 단조 증가(monotonically increasing) → Ex) timestamp, auto-increment ID(1,2,3,...)

## Hotspotting 방지 기법

- Salting (랜덤 prefix 붙이기) : foo0001, foo0002 → a-foo0001, b-foo0002
- Hashing (해시 기반 prefix) : hash(bno) % 16 → 00~0F 사이의 prefix 부여
- Key 뒤집기 / 일부 뒤집기 : 자주 변하는 부분(예: timestamp의 하위자리)을 앞쪽에 위치
