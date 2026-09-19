# Stateful Transformations

> 작성일: 2026-02-04

참조 : [Data Pipelines & ETL](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/etl/)

- Rich Functions → state 접근, lifecycle 관리 ( stateful에서 필수 )
	- open() → state초기화 / 각 subtask가 시작될 때 딱 1번
	- close() → 리소스 정리 / 외부연결해제
	- getRuntimeContext() → 이벤트마다 실행할 로직 ( map / flatMap )

```javascript
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class RichFunctionExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<String> input = env.fromElements("flink", "rich", "function");
        DataStream<String> result = input.map(new MyRichMapper());
        result.print();

        env.execute("Rich Function Example");
    }

    public static class MyRichMapper extends RichMapFunction<String, String> {

        @Override
        public void open(Configuration parameters) throws Exception {
            System.out.println("초기화 수행");
            System.out.println("subtask index = " + getRuntimeContext().getIndexOfThisSubtask());
        }

        @Override
        public String map(String value) throws Exception {
            return value.toUpperCase();
        }

        @Override
        public void close() throws Exception {
            System.out.println("종료 처리 수행");
        }
    }
}
```
