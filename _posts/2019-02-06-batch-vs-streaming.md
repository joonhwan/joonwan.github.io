---
layout: post
title:  "Batch Processing과 Streaming Processing"
date:  2020-02-06 09:41:36 +0900
tags: concept, sw, bigdata
---

[원본글](https://thenewstack.io/the-big-data-debate-batch-processing-vs-streaming-processing/)을 읽고 요약

## Batch Processing 과 Streaming Processing 의 차이

기본적으로

- Batch Processing : 일정 **시간 범위** 내의 데이터 포인트**들**. "Window of Data" 로 처리. "Large **Volumes** of Data"용. 
- Stream Processing : 특정 **시각**의 데이터 포인트. "Fast Data".  실제 처리를 위해 *마이크로 배치 스타일* 혹은 *실시간* 으로 처리된다.  "Large **Quantities** of Data"용 

의 개념이다. 

따라서, 덩치가 큰 데이터("Large **Volume** Data", 역주 : 갯수가 많은것이 아니라... 데이터 단위 하나하나의 크기가 큰...) 혹은 레거시 시스템내 데이터 소스들(역주 : 이를테면 RDBMS?, FileSystem?)에 있는 데이터들은 Stream 방식으로 처리하기 힘드므로 Batch Processing방식을 취한다. Batch Processing 를 위한 데이터는, 모든 데이터가 일련의 저장소, DB, 혹은 파일시스템에 처리 전에 적재가 된 다음 처리되는게 보통이다. 이런 점 때문에, IT부서 사람들은 종종 분석단계를 시작하기에 앞서 데이터가 다 적재되기를 앉아서 기다리는 일이 종종있다. 결국, 데이터 단위 1개의 발생이 후 실제 처리후 결과를 얻는 과정을 *실시간* 으로 보기는 어렵다. 

반면, Apache Beam 이나 Spark 같은 플랫폼은 입력되는 많은 양의 데이터("Large **Quantities** of Data". 데이터 단위 단위에 따라 실시간의 결과를 얻을 수 있는 Streaming Processing 플랫폼이다. 

## "빅데이터 논쟁" ... 어떤걸 써야 하는걸까?


결론은... Streaming이 화두이고, 많은 작업을 이걸로 하려 하지만, 모든 경우에 잘 들어맞는것은 아니다. Batch가 필요한 경우도 있다. 비즈니스 목적을 잘 생각해 보자. 

또, Streaming 방식의 경우에는 지속적으로 변화할 수 있는 데이터의 접근성(Accessibility)문제(역주 : 데이터 포맷이 일부 변경?) 및 품질문제(역주: 데이터의 값이 잘못들어감?) 에 대한 대응방안이 있어야만 적용이 가능하다(역주 : 물론 Batch 처리도 동일한 문제가 있겠지만, 이것들은.. 문제 수정후 다시 처리하면 그만이다).

터빈엔진에서 끝없이 측정되어 나오는 센서의 값 같은것은 Streaming 처리에 아주 적합한 경우이다. 


> 빅데이터 논쟁에서 멀어지자. 내가 지금 망치 하나를 가지고 있다고 해서, 이거 하나로 모든일을 하려하지 말자


