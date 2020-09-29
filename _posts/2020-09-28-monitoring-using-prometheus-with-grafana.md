---
layout: post
title: Prometheus와 Grafana로 모니터링 시스템 구축하기(1)
excerpt_separator:  <!--more-->
categories:
  - Monitoring
tags:
  - Monitoring
  - Prometheus
  - Grafana
---

## 개요

사람이 직접 모니터링 하는 시스템은 참 가혹합니다.

몽고DB 클라우드 모니터링도 24시간 이내의 모니터링만을 무료로 지원하고, New Relic은 유료로 사용하면 서버당 $140달러(2019.12.16 기준)이라는데 이걸 쓰면 모니터링비가 우리 서버비를 훌쩍..(?)

기본적으로 세팅한 `munin-node`를 통해 간단하게 서버 상태는 체크하고 있었으나

단순한 이미지로 띡 하고 나타나는 불친절함과 사람이 항상 보아야 하는 모니터링은 비효율적이라 판단했습니다. (물론 munin-node도 알림은 가능하지만)

나보다도 먼저 시스템 장애를 알고 고쳐달라 찾아오는 동료들을 보며(...) 이대로 안되겠다 싶어 새 모니터링 시스템을 도입하기로 마음먹고 발견한 프로메테우스(Prometheus)와 그라파나(Grafana)!

모니터링 시스템 구축에서 가장 주저하게 되는 부분은 잘 돌아가고 있는 기존 시스템에 모니터링용 데몬이 해를 끼치지 않을까라는 막연한 생각때문이었는데요.

프로메테우스는 각 서버에 Exporter를 띄워 Metric만 노출해주면, 중앙 프로메테우스 서버에서 가져가는 Pull 방식을 사용하고 있어 이러한 걱정을 조금 덜어낼 수 있었습니다.

Pull 방식이라 Metric 수집에 장애가 걱정되어 도입하지 않으셨다는 분도 계셨으나, 사용해보니 제가 쓰는 규모에서는 Metric 수집에 이상은 없었고 이전과 크게 부하가 증가하지 않았으며, Metric이 수집되지 않았을 경우를 장애로 판단하는 Alert도 설정이 가능해 크게 개의치 않았습니다.

## 본격 세팅

본격적으로 세팅해 보겠습니다.

서버 한 대도 아껴야 하는 우리는 서버 한 대에 몰아넣어 세팅해보기로 합니다 ㅎㅎ (어짜피 클러스터링도 지원하지 않는 프로메테우스)

### 작업 순서
오늘의 작업 순서는 다음과 같습니다.

1. Prometheus와 Grafana 설치
2. Prometheus Exporter 설치
3. Prometheus - Prometheus Exporter 연결
4. Grafana - Prometheus 연결
5. 대시보드 감상

### Docker Compose로 Prometheus 설치하기
편리하고 언제나 재현 가능한 환경을 만들기 위해 Docker Compose를 통해 환경을 구성하고자 합니다.

[프로메테우스 설치 방법]([https://prometheus.io/docs/prometheus/latest/installation/](https://prometheus.io/docs/prometheus/latest/installation/)) 에서 여러 환경별로 설치 방법을 제공하니 참고하시구요.

적당한 폴더를 생성하여 `docker-compose.yml` 을 작성합니다. 크게 커스터마이징해서 사용하는 것이 아니라면 가능하면 각 업체에서 제공하는 공식 이미지를 사용하시는 게 정신건강에 좋습니다.

```yaml
version: '2'

services:
  prometheus:
    image: 'prom/prometheus:latest'
    ports:
      - '9090:9090'
    volumes:
      - './config/prometheus:/etc/prometheus'
  grafana:
    image: 'grafana/grafana:latest'
    ports:
      - '3000:3000'
```

간단하게 내용을 살펴보면 프로메테우스와 그라파나 컨테이너를 각각 9090, 3000번 포트에 띄웁니다.

프로메테우스 config 파일을 위해 volume을 연결해주었습니다.

### Prometheus 설정

다음으로 `./config/prometheus` 폴더를 만들어 `prometheus.yml` 파일을 작성해줍니다.

```yaml
global:
  scrape_interval:     15s # 15초마다 수집
  evaluation_interval: 15s # 15초마다 평가 

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus' # 수집하는 서버 역할명을 써줍니다

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```

설정에 보시면 alertmanagers가 있는데요. Grafana 없이 Prometheus만으로도 특정 시점에 alert을 할 수 있습니다만, Grafana를 이용할 거라 넘어갑니다.

이렇게 기본 예제에 따라 작성해주면 프로메테우스 자기 자신을 모니터링하게 됩니다. 일단 궁금하니 `docker-compose up` 을 통해 서비스를 시작해 웹 UI로 접속해봅니다.

### Node exporter 연결

암튼 우리는 프로메테우스를 모니터링하려고 프로메테우스를 설치한 게 아니니 모니터링하고자 하는 서버를 위한 Exporter를 설치하고 연결해주어야 합니다.

가장 기본적인 Exporter는 Node Exporter(node.js 아님 주의)로, 리눅스 서버의 기본적인 메트릭(CPU, RAM, Disk 등)을 수집할 수 있도록 메트릭을 노출해줍니다.

[Node Exporter]([https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter))도 Docker를 통해 설치할 수 있으나, 기존 서버는 도커 설치도 안되어 있고 얘를 설치하려고 도커를 설치하는 것도 배꼽이 더 커지는 격이라 직접 설치하도록 하겠습니다.

Go로 작성되어 직접 빌드하여 사용하거나, 이미 만들어진 릴리즈를 다운받아(추천) 사용하시면 됩니다.(몰라서 빌드해서 씀)

RHEL/CentOS 기준으로 아래와 같이 설치합니다.

```bash
curl -Lo /etc/yum.repos.d/_copr_ibotty-prometheus-exporters.repo https://copr.fedorainfracloud.org/coprs/ibotty/prometheus-exporters/repo/epel-7/ibotty-prometheus-exporters-epel-7.repo
yum install node_exporter
```

이제 `node_exporter` 명령을 통해 바로 exporter를 띄울 수 있습니다! 서비스에 등록해서 사용하거나 nohup을 통해 띄워놓으시면 됩니다.

[xxx.xxx.xxx.xxx:9100](http://xxx.xxx.xxx.xxx:9100) 에 들어가보니 정상적으로 동작하고 있네요!

자 이제 다시 prometheus.yml로 돌아가 방금 추가한 서버를 모니터링 대상에 추가해줍니다.

```yaml
...

scrape_configs:
  - job_name: 'prometheus' # 수집하는 서버 역할명을 써줍니다

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'web-server'
    static_configs:
    - targets: ['xxx.xxx.xxx.xxx:9100', 'yyy.yyy.yyy.yyy:9100'] # 같은 역할의 클러스터는 배열로 한번에 추가해줍니다.

```

이런 식으로 역할(job) 별로 추가할 수 있습니다.

이렇게 추가해줄 경우 추가 후에 항상 프로메테우스를 재시작해줘야 하는데요. kubenetes나 Consul을 사용하면 Dynamic하게 관리도 가능하다고 합니다. 아니면 파일 기반으로 서버 목록을 관리하여 해당 json 파일만 업데이트하면 재시작 없이 동적으로 새 서버를 추가할 수 있습니다. 자세한 내용은 [파일 기반의 Dynamic 타겟 구성]([https://prometheus.io/docs/guides/file-sd/](https://prometheus.io/docs/guides/file-sd/)) 을 참고해주세요. (~~저는 그냥 재시작할거라서~~)

추가가 완료되었다면 `docker-compose restart prometheus` 를 통해 재시작해주면 새로 등록한 서버 메트릭을 추가로 수집하고 있는 것을 웹 콘솔에서 확인할 수 있습니다!

자 이제 잠깐 쉬시고, 아까 띄워놓고 잊고 있었던 그라파나를 다음에 다뤄보도록 하겠습니다.