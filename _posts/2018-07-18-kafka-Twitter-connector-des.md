---
layout: post
title:  "kafka twitter connector 실습"
subtitle:   "kafka twitter connector 실습해보자"
categories: data
tags: kafka
comments: true
---

## kafka twitter connector 패캠 데엔스 가이드 프로젝트

#### 가이드 프로젝트에서 사용할 내용
스파크 스트림으로 트위터 실시간 조사한걸 가지고 카프카로

실시간 트위터 스트림, 카프카, 스파크 스트리밍 까진 사용하고

그외에 ( 형태소분석, word2vec) nlp처리 , (machine learning) spark MLlib, 결과 시각화는 개인/파티 별






### Kafka
스트리밍 처리는 생각보다 실용성이 떨어진다.

데이터를 가지고 뭐를 할껀지 생각하면, 회사의 전략을 세운다 치면 지금의 데이터 까지는 필요하지 않을꺼다.

주기를 작년부터 이번달 까지 월별 데이터를 본다거나, 회사의 마케팅 전략을 위해 본다면
일단위로 보는게 괜찮을텐데

실시간으로 대응할만한 일이 무엇이 있을지 생각하면 그리 많지 않다.

실시간으로 대응 할 만한 분석이 없진 않다.
실시간 요금 계산이라던가 배차 관리는 실시간으로 할 필요가 있다.**( 설계 , 구조 를 잘 짜놔야 한다)**

kafka는 메시지를 효율적으로 관리하는 툴로 여러 소스에서 발생한 데이터를 1개 혹은 여러개의 타겟 시스템으로 전송하는 경우에 사용하는 유틸이다
소스가 타겟 시스템에 직접연결하여 데이터를 n:n 으로 전송하면 여러 이유르 문제가 생길 수 있고 시스템 안정성이 떨어진다

이를 해결 하기 위해 kafka는 중앙 관리 시스템으로 데이터를 전부 모아서 타겟 시스템으로 전송하는 역할을 하며 시스템 안정성을 높이는 컨셉으로 나옴

* 빠르다
	* 메세지를 묶어서 배치로 관리
	* TCP 를 통한 binary API
	* 캐시를 통해 consumer에 데이터 제공

* 안정적
	* 메세지가 유실 되지 않도록 디스크 저장
	* 메세지 순서 보장
	* topic를 N으로 replicate

* 구성
	* 메세지를 topic 이라는 카테고리를 분류
	* 소스는 producer, 타겟 시스템은 consumer
	* zookeeper가 kafka 클러스터를 관리

> 자세한 내용은
> [kafka slidshare](https://www.slideshare.net/jhols1/kafka-atlmeetuppublicv2)



## 가이드 프로젝트 정리

### 트위터를 검색하여 kafka 를 거쳐서 스파크 쉘에 출력하기



#### 카프카 설치
카프카 다운로드 [kafak download](http://http://kafka.apache.org/downloads/)
>scala version 맞춰서 받는다 2018-07-18 아직은 kafka_2.11.1.1.0 버전이 더 안정성 있다고 얼핏 들은...**확실치 않다**

압축 풀고 터미널에서 zookeeper 와 kafka 를 실행시킨다

##### zookeeper
```
$KAFKA/bin/zookeeper-server-start.sh $KAFKA/config/zookeeper.properties
```
##### kafka
```
$KAFKA/bin/kafka-server-start.sh $KAFKA/config/server.properties
```

Kafka Twitter connector 를 사용한 예제를 조금 수정하여 테스트 한다
[kafka-connect-twitter](https://github.com/Eneco/kafka-connect-twitter)

소스가 업데이트가 안된지 꽤지났고 그사이 kafka 버전업됨.

누군가 pull-request 로 소스 변경사항을 올렸지만 반영이 되지 않았다

pom.xml 을 수정한다

```
<scala.version>본인의 스칼라 버전</scala.version>
<confluent.version>4.1.1</confluent.version>
<kafka.version>1.1.0</kafka.version>
<guava.version>23.0</guava.version>
```

src/main/scala/com/eneco/trading/kafka/connect/twitter/TwitterSourceConnector.scala 파일에서 아래 부분을

```
import org.apache.kafka.connect.connector.{Task, Connector}
```

아래와 같이 수정 및 추가한다

```
import org.apache.kafka.connect.connector.{Connector, Task}
import org.apache.kafka.connect.source.SourceConnector
```

수정 한 후

twitter-source.properties.example 파일을 twitter-source.properties 복사 하여

기존에 만들어 놓은 본인의 트위터 앱

>페이스북 처럼 트위터도 내부적으로 앱을 만들어서 auth 인증 키 값을 가져온다. 이에 대한 부분은 포스팅 여기에 포스팅 하기에는 내용이 너무 길어지니 따로 찾아보시길...

에서 Consumer Key (API Key) , Consumer Secret (API Secret) , Access Token	, Access Token Secret 를 복사해서 붙여넣어준다

```
twitter.consumerkey=Consumer Key (API Key)
twitter.consumersecret=Consumer Secret (API Secret)
twitter.token=Access Token
twitter.secret=Access Token Secret
```

검색어, output type을 설정한다

```
topic=twitter
output.format=string
language=en
track.terms=soyou,kpop,sistar
```

최종적 으론 아래와 같다

```
name=twitter-source
connector.class=com.eneco.trading.kafka.connect.twitter.TwitterSourceConnector
tasks.max=1
topic=twitter
twitter.consumerkey=Consumer Key (API Key)
twitter.consumersecret=Consumer Secret (API Secret)
twitter.token=Access Token
twitter.secret=Access Token Secret

# set output.format to string to output string key/values, it defaults to structured
output.format=string

language=en
# stream.type=sample
stream.type=filter
track.terms=soyou,kpop,sistar
# San Francisco OR New York City
# track.locations=-122.75,36.8,-121.75,37.8,-74,40,-73,41
# bbcbreaking,bbcnews,justinbieber
# track.follow=5402612,612473,27260086
```
topic 은 카프카에서 전달할 topic name

language 는 검색에 사용될 언어

track.terms 가 검색어

track.locations 가 위치
나머지는 파악중

설정이 끝났다 실행시켜 보자

outdate된 소스파일이 있어 수정, maven으로 빌드

> $KAFKA 는 kafka 가 설치된 폴더

```
mvn clean package
export CLASSPATH=`pwd`/target/kafka-connect-twitter-0.1-jar-with-dependencies.jar
$KAFKA/bin/connect-standalone.sh connect-simple-source-standalone.properties twitter-source.properties
```

mvn 빌드를 하고 classpath 를 잡아준다.

twitter-source.properties 를 kafka standalone.sh 으로 실행시킨다

트위터 에서 검색된 내용을 kafka 에서 스파크 쉘로 보내주는거니깐 스파크 쉘 실행

```
./spark-shell --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.3.0
```

```
val kafka_df = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "localhost:9092").option("subscribe", "twitter").load()
val kafka_df_string = kafka_df.select($"key".cast("STRING"), $"value".cast("STRING"))
import org.apache.spark.sql.streaming.Trigger
val output = kafka_df_string.writeStream.outputMode("update").format("console").option("truncate", "false").trigger(Trigger.ProcessingTime("5 seconds")).start()
```

실행된 spark-shell 에서 한줄씩 입력하면 실행중인 트위터에서 검색된 내용이 kafka 를 거쳐 spark-shell 에 출력된다


twitter-source.properties 가 돌고 있는 터미널 창에는 아래와 같은 메시지가 블라블라 나오고

```
pool-2-thread-1] INFO com.eneco.trading.kafka.connect.twitter.TwitterSourceConfig - TwitterSourceConfig values:
        batch.size = 100
        batch.timeout = 0.1
        language = [ko]
        output.format = string
        stream.type = filter
        topic = twitter
        track.follow = []
        track.locations = []
        track.terms = [soyou, kpop, sistar]
        twitter.app.name = KafkaConnectTwitterSource
        twitter.consumerkey =
        twitter.consumersecret = [hidden]
        twitter.secret = [hidden]
        twitter.token =
```



결과값은 아래와 같이 나온다.

***검색어를 한글로 해보고 싶었는데 실행하다보니 한글이 깨져서 넘어간다.ㅠㅠ***

```
Batch: 5
-------------------------------------------
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------+
|key          |value                                                                                                                                       |
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------+
|byuntr       |RT @kyungsoosama: They are truly the PROUD FACES OF KPOP 💯💯💯

#EXO @weareoneEXO

https://t.co/HbHD52yPF0                                |
|jik16559470  |RT @mrboi_x: MOVIE: TWO SISTAR IN LAW HORNY https://t.co/LwVxTErgbw                                                                         |
|FallsforRM   |RT @firstsight_jk: Please preorder your albums ONLY on the Amazon and Target links provided by big hit if you're in the US. You can still s…|
|jeonchaotic  |wth i hate yall so much                                                                                                                     |
|arianaxjoonie|RT @firstsight_jk: Please preorder your albums ONLY on the Amazon and Target links provided by big hit if you're in the US. You can still s…|
|chateaurixeu |RT @buditcheoyahae: 1 YEAR WITH QUEEN "THE WAR"
- EXO becomes the first Kpop group to debut number 1 on Melon after the system change
-accu…|
|oshsayhun    |RT @KBSWorldTV: #EXO appeared on the Large Scale LED SHOW at BURJ KHALIFA and It's the FIRST Non-Royalty to be in the LED SHOW!!!! #EXO #Du…|
|btsicula     |RT @firstsight_jk: Please preorder your albums ONLY on the Amazon and Target links provided by big hit if you're in the US. You can still s…|
|queenNira17  |RT @SubjectKpop: JYP is probably one of the best boss you see in the Kpop industry

He is willing to fight for fair competition in the ind…|
+-------------+--------------------------------------------------------------------------------------------------------------------------------------------+

```

#### 차후 어떤 자유 프로젝트를 할 껏인지 고민좀 해보자..
### 이것도 조금 분석해야겠고
# 어떤 프로젝트를 해야할지 겁나 막막하다
