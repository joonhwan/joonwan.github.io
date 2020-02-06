---
layout: post
title:  "Data Pipeline 기초?!"
date:  2020-02-06 10:57:35 +0900
tags: concept, sw, bigdata
---

[원문](https://towardsdatascience.com/data-pipelines-luigi-airflow-everything-you-need-to-know-18dc741449b7) 내용 번역/정리해봄.

**DISCLAIMER** : 원문이 2018년 자료임. 

## WMS? PipeLine?

WMS 는 Workflow Management System 의 약자. Pipelining을 통해 목적을 달성하는 시스템.

## WMS 가 필요한 이유

데이터를 옮기고 변경시켜서 어떤 작업을 수행하는 건 모든 회사/소프테워가 하는일.  이를테면,

> 아마존 S3 저장소내 수 많은 로그들을 주기적으로 가져와서 유용한 정보를 추출하여 DB 에 넣는다 

와 같은 작업. 

비즈니스 초기단계에는 이걸 수작업으로 하지만, Scale Up을 해야 되는 상황이 되면, 이 작업을 자동화하여 프로그램을 만든다음 cron(일정 시간이 되면 특정 명령어를 실행하는 도구)같은 거에 실어서 주기적으로 수행되게 한다. .... 그러다 이걸로도 충분하지 않은 상황이 들이닥친단다(역주 : 무슨 상황일까...매번 하는 일이 동일하지 않은 경우...인가... ). 이런 상태가 되면 필요한게 **WMS**

이제 WMS들에 머가 있는지 알아본다.

## [AirFlow](https://airflow.apache.org/)

![](https://upload.wikimedia.org/wikipedia/commons/thumb/d/de/AirflowLogo.png/220px-AirflowLogo.png)

AirBnB 에 서 만든 시스템. 다른 WMS와 다른 점이라면... 

> A key differentiator is the fact that Airflow pipelines are defined as code and that tasks are instantiated dynamically.

란다(나머지 글을 다 읽으면 이게 무슨 뜻인지 이해 된단다)

Airflow 에서는 Workflow는 비순환그래프(DAG, Directed Acyclic Graph)로 기술한다(DAG에 대해서는 [여기](https://m.blog.naver.com/occidere/220921661731) 참고. 위상정렬이라는 개념도 함께 알아두면 도움).  그래프의 각 노드는 

- Operator : *연산*을 실행한다.
- Sensor : 프로세스 혹은 자료구조의 상태를 체크한다. 

의 2가지 범주가 있다.  

Airflow를 구성하는 중요한 3개의 요소는 

- Metadata Database : Workflow 및 Task의 상태를 저장한다. 
- Scheduler : 위 Metadata Database의 내용을 바탕으로 다음 작업이 무엇인지 확인하고 수행시킨다. 
- Executer : 메시지 큐 프로세스이며, 보통 [Celery](http://www.celeryproject.org/) 를 사용하여 어떤 Worker가 작업을 수행할 것인지를 결정한다. 

![](https://miro.medium.com/max/800/1*-wDxsSr5_2NyOHxysroYyA.png)

Celery Executor를 쓰면, 작업들을 여러 컴퓨터에 나누어 분산처리가 가능하다(물론 한 컴퓨터에서 처리도 가능)

Airflow에는 UI가 제공되어, 이를 통해 DAG를 모니터링하고, 어떤 작업이 수행중인지 등을 알 수 있다. 

Airflow는 *set it and forget it* 접근방식을 취한다. 즉, DAG를 설정하면 더이상 제어할게 없다(지정된 반복주기/시간에 실행된다)

## [Luigi](https://github.com/spotify/luigi)

![](https://raw.githubusercontent.com/spotify/luigi/master/doc/luigi.png)

Spotify 에서 만든 pipeline 도구. Airflow와 비교하면 이해도가 올라간단다. 

Luigi 도 Airflow와 마찬가지로, Workflow 를 task들과 이들간 의존성으로 기술한다. 

Luigi를 구성하는 2개의 중요한 핵심은... 

- Task  : 어떤 연산작업. 다른 Task들의 결과를 소비하여 작업을 하기도 한다. 
- Target : Task를 수행결과로 생성되는 파일. 

![](https://miro.medium.com/max/695/1*vVV3tw3ZS1XQLZCNAfrMeQ.png)

## 실제 구성해보기 

먼저 Luigi로 한다음, Airflow를 해본다. 

### Simple Workflow 만들어보기

아래 `luigi.Task` 로 부터 상속받은 2개의 python class가 Task이다. 

```py
import luigi


class WritePipelineTask(luigi.Task):

    def output(self):
        return luigi.LocalTarget("data/output_one.txt")

    def run(self):
        with self.output().open("w") as output_file:
            output_file.write("pipeline")


class AddMyTask(luigi.Task):

    def output(self):
        return luigi.LocalTarget("data/output_two.txt")

    def requires(self):
        return WritePipelineTask()

    def run(self):
        with self.input().open("r") as input_file:
            line = input_file.read()

        with self.output().open("w") as output_file:
            decorated_line = "My "+line
            output_file.write(decorated_line)
```

위 Task는 다음과 같은 일을 한다.

- `WritePipelineTask` 
  - 입력 : 없음
  - 출력 : `pipeline`이라는 문자열
- `AddMyTask` 
  - 입력 : 임의 문자열
  - 출력 : 임의 문자열 앞에 "My" 를 덧붙임

`AddMyTask` 의 경우는 `require()` 가 있는데, 이것이 의존성을 지칭하며, `WritePipelineTask`의 생성자를 호출하여 객체를 만든다. 
코드로 읽기가 꽤 수월하게 구성되어 있다.

(역주: 위 코드를 실제로 실행하려면, `luigi -module my_task AddMyTask --local-scheduler` 이런식으로 명령을 쳐야 한다. 자세한 내용은 [공식 문서](https://luigi.readthedocs.io/en/stable/example_top_artists.html#running-this-locally) 참고)


이걸 이제 Airflow로 만들면 아래와 같다.

```py
import os

from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator

OUTPUT_ONE_PATH = "data/output_one.txt"
OUTPUT_TWO_PATH = "data/output_two.txt"


def decorate_file(input_path, output_path):
    with open(input_path, "r") as in_file:
        line = in_file.read()

    with open(output_path, "w") as out_file:
        out_file.write("My "+line)


default_args = {
    "owner": "lorenzo",
    "depends_on_past": False,
    "start_date": datetime(2018, 9, 12),
    "email": ["l.peppoloni@gmail.com"],
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 0,
}


dag = DAG(
    "simple_dag",
    default_args=default_args,
    schedule_interval="0 12 * * *",
    )

t1 = BashOperator(
    task_id="print_file",
    bash_command='echo "pipeline" > {}'.format(OUTPUT_ONE_PATH),
    dag=dag)

t2 = PythonOperator(
    task_id="decorate_file",
    python_callable=decorate_file,
    op_kwargs={"input_path": OUTPUT_ONE_PATH, "output_path": OUTPUT_TWO_PATH},
    dag=dag)

t1 >> t2
```

`DAG` 클래스 객체를 먼저 생성한다음, 여기에 속한 2개의 operator 객체(`BasicOperator`객체 및 `PythonOperator`객체. 일부로 서로 다른 Operator 형을 사용해 보았다. 이럴수도 있다는 걸 보여주기 위해... )를 생성하였다. 
생성된 operator 객체들간의 의존성은 맨 마지막 `t1 >> t2` 로 지정되었다. 

(역주: 상기 스크립트를 실행하려면, `airflow.cfg` 설정파일에 지정된 "DAG 폴더"상에 상기 스크립트를 `my_task.py` 라는 이름으로  저장하고, `airflow run my_task runme_0 2020-12-11` 처럼 명령을 터미널에서 입력해야 한다. 상세내용은 [공식 문서](https://airflow.apache.org/docs/stable/tutorial.html#running-the-script) 참고)

위에서 알 수 있듯이, Airflow 에는 input 또는 output의 개념이 없다. 2개의 operator간에는 그 어떤 정보 공유도 없다(역주: 하지만, `OUTPUT_ONE_PATH` 라는 파일의 경로 정보가 공유되었다... 멀 보고 정보공유가 없다고 한지 잘 모르겠다).  만일 정보공유가 필요하다면, 그 2개의 Operator는 1개의 Operator로 구성되어야 한다는 것이 Airflow의 방식이다. 


### 좀 더 복잡한 workflow

50개의 텍스트파일에 `pipeline` 문자열을 기록한다음, 거기에 다시 `My ` 문자열을 붙이는 작업을 수행해 본다. 


Luigi의 경우에는 parallelism ? 을 위해 다음과 같이 `AllTasks` 라는 이름의 WrapperTask를 생성하고 `require`를 통해 50개의 `WritePipelineTask`/`AddMyTask` 쌍으로 구성된 작업을 엮는다. 

```py
import luigi

NUMBER_OF_FILES = 50


class WritePipelineTask(luigi.Task):
    flow_id = luigi.Parameter()

    def output(self):
        return luigi.LocalTarget("data/output_one_{}.txt".format(self.flow_id))

    def run(self):
        with self.output().open("w") as output_file:
            output_file.write("pipeline")


class AddMyTask(luigi.Task):
    flow_id = luigi.Parameter()

    def output(self):
        return luigi.LocalTarget("data/output_two_{}.txt".format(self.flow_id))

    def requires(self):
        return WritePipelineTask(flow_id=self.flow_id)

    def run(self):
        with self.input().open("r") as input_file:
            line = input_file.read()

        with self.output().open("w") as output_file:
            decorated_line = "My "+line
            output_file.write(decorated_line)


class AllTasks(luigi.WrapperTask):

    def requires(self):
        for i in range(NUMBER_OF_FILES):
            yield AddMyTask(flow_id=str(i))
```

위 처럼 한다음,  `luigi --module my_task AllTasks --workers 4`  로 실행하면, 4개 프로세스가 떠서 10개 작업을 나누어 처리한다. 
실제상황에서는 위처럼 할 수도 있고, 또는  `run()` 메소드 안에서 아예 따른 시스템(예를 들면 Spark)에 병렬작업을 위임할 수 도 있겠다. 

Airflow에서는 어떻게 할까

위에서 `DAG` 은 코드내에서 동적으로 생성된다고 했었던 걸 기억하는가? 이말의 의미는.. 다음과 같은 걸 할 수 있다는 것이다. 

```py
import os

from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator

DATA_FOLDER = "data"


def decorate_file(input_path, output_path):
    with open(input_path, "r") as in_file:
        line = in_file.read()

    with open(output_path, "w") as out_file:
        out_file.write("My "+line)


default_args = {
    "owner": "lorenzo",
    "depends_on_past": False,
    "start_date": datetime(2018, 9, 20),
    "email": ["l.peppoloni@gmail.com"],
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 0,
}


dag = DAG(
    "multiple_files_dag",
    default_args=default_args,
    schedule_interval="0 12 * * *",
    )

for i in range(10):
    output_one_path = os.path.join(DATA_FOLDER, "output_one_{:d}.txt".format(i))
    output_two_path = os.path.join(DATA_FOLDER, "output_two_{:d}.txt".format(i))

    t1 = BashOperator(
        task_id="print_file_{:d}".format(i),
        bash_command='echo "pipeline" > {}'.format(output_one_path),
        dag=dag)

    t2 = PythonOperator(
        task_id="decorate_file_{:d}".format(i),
        python_callable=decorate_file,
        op_kwargs={"input_path": output_one_path, "output_path": output_two_path},
        dag=dag)

    t2.set_upstream(t1)
```

단순위 Simple Workflow의 예에서 `t1`/`t2` Task 2개를 만들던것처럼 하되, `for` 루프를 돌면서 DAG을 생성한다!. (cf. Luigi는 Class 선언으로 DAG을 구성하기 때문에, 객체 단위로 하려면 WrapperTask 를 써야 했었다). 이렇게 하면 다음과 같이 여러 쌍의 Branch 들이 생긴다. 

![](https://miro.medium.com/max/384/1*4xY3fyTu5reN2zIpYEUchQ.png). 

Airflow의 경우 parallel 처리는 시스템(정확히는 Airflow의 Executor)이 알아서 해준단다(위 각 Branch가 동시에 실행된다)

## 결론

정리하자면 

- **Luigi** 는...
  - pipeline에 기반하였고, 각 task의 input/output은 정보를 공유하고 서로 연결된다. 
  - Target기반 접근(Target은 Task의 결과물-통상 파일-)
  - UI는 꼭 필요한 기능만 단순하게 구현되었고, 프로세스를 실행하거나 하는 기능은 없다.
  - 내장된 Trigger가 없다(`crontab` 파일을 편집해서 일정주기마다 실행되게 하는 식으로 구성해야 한단다)
  - 분산 실행은 지원하지 않음(역주 : 병렬실행은 지원하지만...)
  - 재미있는 pp 자료가 있다. [LUIGI CENTRAL SCHEDULER - 2015](https://www.arashrouhani.com/luigid-basics-jun-2015/#/)

- **Airflow** 는 ...
  - DAG 표현에 기반
  - Task간 정보공유가 없으므로, 최대한 병렬화를 할 수 있단다(위상정렬/순서만 잘 정하면 된다)
  - Task간 통신 수단이 마땅치 않다.
  - 설정하면 분산 실행할 수 있는 Executor가 있다
  - Scheduler가 있으므로, *set it and forget it* 할 수 있다(스스로 trigger한다)
  - 실행상태를 확인하고, 작업중인 Task를 조작할 수 있는 강력한 UI가 있다

