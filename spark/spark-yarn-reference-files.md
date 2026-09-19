# Spark Yarn 실행 참조 파일

> 작성일: 2025-10-28

- hadoop : Yarn, FileIO를 HDFS로 사용 ( core-site.xml / hdfs-site.xml / yarn-site.xml )
- hive : Hive 카탈로그/권한/파티션 메타데이터를 사용 ( hive-site.xml )
- spark-env.sh : 에 아래 경로 추가 하여 참조할 수 있도록 설정
	- export HADOOP_CONF_DIR=/rnd/hadoop/default/etc/hadoop
	- export YARN_CONF_DIR=/rnd/hadoop/default/etc/hadoop
	- export HIVE_CONF_DIR=/rnd/hive/default/conf
