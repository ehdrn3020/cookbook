# Spring for Kafka

> 작성일: 2025-12-28

- 메세지 기반 POJO(Message-driven POJO) 사용
	- POJO 란 비즈니스 로직 중심 / 단순한 필드 + 메서드 순수 자바 클래스
	- 코드 안에서는 "KafkaConsumer", "ConsumerRecord" 알 수 없고, "비즈니스 로직"만 작성
