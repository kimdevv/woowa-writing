# Grafana로 모니터링 환경을 구축해 보자.  

# 서론: Why Grafana?

핏토링 팀은 데모데이를 일주일 앞두고 있는 상황이다. 데모데이 예상 인원 100명을 동시에 수용할 수 있는지 확인하기 위해 개발 서버에 부하 테스트를 진행하고자 했다. 따라서 아래와 같은 이유로 모니터링 서버가 필요했다.

- 개발 서버에서 부하 테스트 시, 서버에 얼마나 부하가 들어왔는지 확인하기 위해서
- 코드 내 병목 지점을 찾아 이를 개선하기 위해서
- 운영 서버에서 오류가 발생했을 때 이를 로그로 남겨 추적하기 위해서
- 이외 서버 관련 지표를 빠르게 확인하기 위해서

기존에 핏토링 팀은 아래 서비스들을 활용하여 모니터링을 진행하고 있었다.

- CloudWatch Logs: 개발 서버 및 운영 서버 로그 확인
- CloudWatch Metrics: 개발 서버 및 운영 서버 시스템 메트릭 확인
- CloudWatch Dashboard: 위에서 수집한 로그 및 메트릭 시각화
- Pinpoint APM: 애플리케이션 요청 별 trace 확인, 애플리케이션 메트릭 확인

그러나 위 서비스를 활용하여 모니터링을 해보니, 아래와 같은 문제점들을 경험했다.

- 비용 문제 상 Pinpoint와 관련된 모든 서비스들(Web, Collector, HBase, ZooKeeper)을 하나의 EC2 인스턴스에 모아서 사용해야 했다. 그러다보니 메모리가 부족해져서 인스턴스가 종종 꺼지는 현상이 발생했다.
- 각 서비스들은 각각의 로그가 만들어진다. 대용량 데이터를 넣고 테스트할 때 로그의 크기가 굉장히 커져서 인스턴스가 꺼지는 현상이 발생했다.
- 비교적 높은 러닝 커브로 인해 Pinpoint APM에서 제공하는 기능들을 제대로 활용하지 못했다. 때문에 애플리케이션 수준의 메트릭을 확인하지 못하는 상황이었다.
- 서버 모니터링을 하나의 대시보드에서 할 수 없고, 목적에 따라 CloudWatch나 Pinpoint 대시보드로 이동해야만 했다.

이 중에서 가장 큰 문제는 EC2 메모리 부족으로 모니터링 서버가 계속해서 꺼지는 현상이었다. 대략 20분 당 1회 단위로 인스턴스가 꺼지는 상황이 반복되었고, 그럴 때마다 수동으로 모니터링 서버 인스턴스를 다시 켜 주어야 했다.

데모데이에서는 약 100명의 인원들이 동시에 우리 서비스에 접속한다. 그러나 구현한 기능들이 100% 온전하게 작동될 것이라는 보장은 없었고, 사용자들이 예상하지 못한 오류를 접할 지도 모르는 상황이었다. 그럴 때마다 우리가 수동으로 오류가 발생했던 상황을 기록할 수는 없으니, 당시 상황을 기록할 수 있도록 모니터링 서버가 필요했다. 이러한 상황에서 20분마다 한 번씩 꺼지는 모니터링 서버 인스턴스는 우리에게 치명적인 리스크였다.

모니터링에 가장 널리 사용되는 서비스로 Grafana가 있다고 한다. Grafana에 대해 조금 찾아보니, 현재 상황에서 우리는 아래와 같은 이점을 얻을 수 있을 것 같았다.

- 널리 사용되는 서비스인만큼 레퍼런스가 굉장히 많다. 덕분에 러닝 커브가 비교적 낮을 것이라 판단했고, Pinpoint APM을 사용할 땐 불가능했던 애플리케이션 수준 메트릭이 가능할 것 같았다.
- 모든 모니터링을 하나의 Grafana 서비스에서 관리할 수 있다. 덕분에 여기저기 대시보드를 이동할 필요가 없다.
- 비교적 큰 메모리를 차지하는 Pinpoint APM 및 주변 서비스들과는 달리 Grafana와 주변 서비스가 차지하는 메모리가 굉장히 적다. 덕분에 모니터링 서버가 꺼지는 현상을 방지할 수 있다.

이 정도의 이점만으로도 Grafana를 도입할 이유는 충분하다고 판단했다. 바로 Grafana 도입을 시작해보자.  

# 본론

## 1. Grafana가 뭘까?

우리의 목표는 **CloudWatch와 Pinpiont APM으로 관리하던 모니터링 서버와 독립된 별개의 Grafana 모니터링 서비스를 구축하는 것**이다. 그러기 위해서는 Grafana, Loki, Prometheus, Tempo 총 네 개의 서비스에 대해 먼저 공부해야 한다.

### Grafana

- 수집된 **데이터를 보기 좋게 시각화**해서 보여주는 도구
- CloudWatch Dashboard 대체 가능

즉 Grafana는 데이터를 보여주는 **대시보드 역할**만을 담당하며, 시각화할 데이터의 수집은 다른 서비스에 의존해야 한다.

우리는 기존에 Log, Metric, Trace를 모니터링하고 있었다. 이러한 데이터들을 수집하기 위해서는 어떤 툴을 사용해야 할까?

### Loki

- 서버 **로그**를 수집 및 저장해주는 도구.
- 로그 검색, 필터링, 라벨 기반 인덱싱 기능 등을 지원한다.
- Grafana가 완벽히 지원하는 로깅 서비스.
- 서버에서는 `Promtail`이라는 에이전트를 통해 로그를 Loki로 전달한다.
- CloudWatch Logs 대체 가능

### Prometheus

- 시계열 **메트릭** 수집 및 저장해주는 도구.
    - CPU, 메모리, 요청 수, Latency 등
- Prometheus가 대상 서버로 가서 메트릭을 가져오는 방식 (Pull 방식).
    - 서버에 `node_exporter`을 설치해 두면 Prometheus가 주기적으로 값을 가져와 저장한다.
- CloudWatch Metrics 대체 가능

### Tempo

- **분산 트랜잭션**을 수집 및 조회하는 도구.
- 각 요청의 전체 call chain을 저장하여 Latency나 오류 추적 등을 할 수 있다.
- Tempo UI를 통해서 간단하게 시각화해서 볼 수 있지만, Grafana와 연동해서 볼 수도 있다.
- Pinpoint APM 대체 가능

총 이렇게 네 가지 서비스를 사용하면 기존의 모니터링 환경을 완벽히 대체할 수 있다.

Log, Metric, Trace 수집 관련 도구는 이 외에도 굉장히 많다. 그러나 이 서비스들을 선택한 이유는 **이미 가장 많이 사용되고 있는 서비스들이라 레퍼런스가 많고 무료라는 점**이다.

## 2. 모니터링 서버에 Grafana 설치

가장 먼저 모니터링 서버에 Grafana를 설치해 주었다.

### 1) Docker & Docker Compose 설치

```bash
# Docker,  Docker Compose 설치
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo apt install -y docker-compose

# Docker 서비스 시작 및 자동 실행 설정
sudo systemctl enable docker
sudo systemctl start docker
```

### 2) Grafana Docker Compose 파일 생성

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "80:3000"
    environment:
      - GF_SECURITY_ADMIN_USER={그라파나 관리자 계정}
      - GF_SECURITY_ADMIN_PASSWORD={그라파나 관리자 비밀번호}
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  grafana-data:
```

Grafana Web UI의 기본 포트 번호는 `3000`이었는데, 현재 보안 그룹(`project-public`)에서는 해당 포트가 열려 있지 않아서 외부 포트를 `80`으로 매핑해 주었다.

### 3) Docker 실행

```bash
docker-compose up -d
```

이제 Grafana 설치는 모두 끝났다. 그러나 아직 Loki나 Prometheus 같은 데이터 소스들이 없어서 아무 것도 표시할 수 없다. 단지 데이터를 보여주는 대시보드만 띄운 것이다.

CloudWatch에서 사용하던 데이터를 Grafana와 연결할 수도 있다고 한다. 그러나 CloudWatch 서비스는 더이상 사용하지 않을 계획이므로 따로 연결하지는 않았다.

<img width="1804" height="1844" alt="1" src="https://github.com/user-attachments/assets/35ce06d6-01a7-4e4b-b604-18157cbca41e" />


## 3. 모니터링 서버에 Loki 설치

### 1) Docker Compose 수정

Loki 역시 Docker로 띄울 것이다. 그러기 위해서 앞서 만들어 뒀던 `docker-compose.yml`을 아래와 같이 수정해 주었다. Loki 컨테이너를 추가한 것이다.

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "80:3000"
    environment:
      - GF_SECURITY_ADMIN_USER={그라파나 관리자 계정}
      - GF_SECURITY_ADMIN_PASSWORD={그라파나 관리자 비밀번호}
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

  loki:
    image: grafana/loki:3.4.1
    container_name: loki
    ports:
      - "8080:3100"
    volumes:
      - ./loki:/mnt/config
    command:
      - "-config.file=/mnt/config/loki-config.yaml"
    restart: unless-stopped

volumes:
  grafana-data:
```

Loki의 기본 포트 번호는 `3100`이었는데, 현재 보안 그룹에서는 해당 포트가 열려 있지 않아서 외부 포트를 `8080`으로 매핑해 주어야 했다.

### 2) Loki 설정 파일 다운로드

```bash
wget <https://raw.githubusercontent.com/grafana/loki/v3.4.1/cmd/loki/loki-local-config.yaml> -O loki-config.yaml
```

(이후 해당 설정 파일을 자신의 모니터링 환경에 맞게 적절하게 수정해 주자)

### 3) Docker 실행

```bash
docker-compose up -d
```

## 4. dev 서버에 Promtail 설치

Promtail은 로그 파일을 Loki로 전달해주는 에이전트 서비스이다. 로그 파일의 경로를 지정해 주면 해당 로그 파일이 업데이트될 때마다 새로운 내용이 Loki로 전송된다.

### 1) Promtail 설치

```bash
sudo wget <https://github.com/grafana/loki/releases/download/v3.4.1/promtail-linux-arm64.zip>
sudo unzip promtail-linux-arm64.zip
sudo mv promtail-linux-arm64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

Promtail을 docker으로 띄울 수도 있다. 그러나 docker로 띄우게 될 경우 순수 서버 경로에 있는 애플리케이션 로그 파일에 접근하기 힘들 것 같아서 docker로 띄우지 않기로 했다.

### 2) Promtail 설정 파일 생성하기

```bash
sudo mkdir -p /etc/promtail # Promtail 설정 파일 표준 경로
cd /etc/promtail
sudo wget <https://raw.githubusercontent.com/grafana/loki/v3.4.1/clients/cmd/promtail/promtail-docker-config.yaml> -O config.yaml
```

다운받은 `config.yaml`을 아래와 같이 수정했다.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml # Promtail이 로그 파일을 어디까지 읽었는지 오프셋을 기록해두는 곳

clients:
  - url: http://{모니터링 서버 주소}/loki/api/v1/push # Loki 서버 주소

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: fittoring_logs
      __path__: /home/ubuntu/fittoring/logs/*.log # 로그 파일 경로
  pipeline_stages:
    - multiline:
        firstline: '^\\s*\\{\\s*$|^\\s*\\{\\s*"event"|^No mapping for '
```

### 3) 오프셋 저장 용 positions 디렉토리 만들기

```bash
sudo mkdir -p /var/lib/promtail
sudo chown -R $(whoami) /var/lib/promtail
```

여기서 오프셋이란 Promtail이 로그 파일을 어디까지 읽었는지를 기록해둔 정보다. 이 오프셋을 활용하여 Promtail은 로그 파일을 통째로 Loki에 전달하는 게 아니라, 마지막에 읽은 부분부터 새로 추가된 부분만 Loki로 전송한다.

### 4) 부팅 시 Promtail 자동 실행되도록 설정

```bash
sudo vi /etc/systemd/system/promtail.service
```

내용은 아래와 같다.

```bash
[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yaml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 5) Promtail 서비스 적용 및 실행

```bash
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
sudo systemctl status promtail
```

### 6) Grafana 대시보드 생성 후 Loki와 연결

이제 로그가 보이기 시작한다.

<img width="2904" height="1844" alt="2" src="https://github.com/user-attachments/assets/bd60c329-ac12-465a-883a-97c1a1d1bfa1" />


## 5. dev 서버에 Node Exporter 설치

Loki와 Promtail을 설치하여 Grafana를 통해 로그를 확인할 수 있게 되었다. 즉 CloudWatch Logs를 대체했다.

이제 CloudWatch Metrics를 대체하기 위해 Prometheus와 Node Exporter을 설치해 주어야 한다.

- Prometheus는 메트릭을 수집하는 역할으로, 모니터링 서버에 설치해야 한다.
- Node Exporter은 각 서버의 메트릭을 제공하는 역할으로, dev 서버에 설치해야 한다.

### 1) Node Exporter 설치

```bash
sudo wget <https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-arm64.tar.gz>
sudo tar xvfz node_exporter-*.*-arm64.tar.gz
sudo mv node_exporter-1.9.1.linux-arm64 /usr/local/node_exporter
```

### 2) 부팅 시 Node Exporter 자동 실행 설정

```bash
sudo vi /etc/systemd/system/node_exporter.service
```

내용은 아래와 같다.

```bash
[Unit]
Description=Node Exporter
After=network.target

[Service]
ExecStart=/usr/local/node_exporter/node_exporter --web.listen-address=":443"
Restart=always

[Install]
WantedBy=multi-user.target
```

Node Exporter의 기본 포트는 9100이다. 그러나 dev 서버의 보안 그룹에는 9100이 열려있지 않으므로 443 포트로 대신 열어주었다.

### 3) 서비스 시작

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

sudo systemctl status node_exporter
```

## 6. 애플리케이션 수준 메트릭 관련 설정

Prometheus를 활용하면 JVM과 같은 애플리케이션 수준 메트릭도 확인할 수 있다.

이를 위해서 스프링부트 앱에 Micrometer Prometheus 레지스트리 의존성을 설치해 주어야 한다.

```java
implementation 'io.micrometer:micrometer-registry-prometheus'
```

그리고 `build.gradle`에서 프로메테우스 관련 경로를 허용해준다.

```java
...

management:
  endpoints:
    web:
      base-path: /
      exposure:
        include: health,info,prometheus
      path-mapping:
        health: healthcheck
        prometheus: prometheus-provider

...
```

이제 `/prometheus-provider` 경로로 접근하면 프로메테우스에게 제공할 메트릭이 표시된다.

디폴트 값인 `actuator/prometheus`를 그대로 안 쓰고 `/prometheus-provider`으로 바꾼 이유는 `actuator/prometheus`를 그대로 사용하게 되면 다른 사용자에게 메트릭이 쉽게 노출될 것 같았기 때문이다(아무래도 스프링 액추에이터가 널리 사용되는 라이브러리이다 보니 개발자라면 해당 url을 쉽게 유추할 수 있을 것).

## 7. Prometheus 설치

### 1) docker-compose.yml 수정

다른 서비스들과 함께 쉽고 켜고 끌 수 있도록 Prometheus 역시 docker으로 띄울 것이다.
그래서 아래와 같이 `docker-compose.yml` 파일을 수정해 주었다.

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "80:3000"
    environment:
      - GF_SECURITY_ADMIN_USER={그라파나 관리자 계정}
      - GF_SECURITY_ADMIN_PASSWORD={그라파나 관리자 비밀번호}
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

  loki:
    image: grafana/loki:3.4.1
    container_name: loki
    ports:
      - "8080:3100"
    volumes:
      - ./loki:/mnt/config
    command:
      - "-config.file=/mnt/config/loki-config.yaml"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    restart: unless-stopped

volumes:
  grafana-data:
  prometheus:
```

Prometheus는 외부에서 접근할 일이 없기 때문에 9090 포트를 따로 다른 포트와 매핑하지 않아도 된다.

### 2) Prometheus 설정 파일 생성

`home/ssm-user/grafana/prometheus/prometheus.yml` 경로에 아래와 같은 파일을 만들어준다.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'os_metrics' # OS, 서버 인스턴스 메트릭 수집 경로 (Node Exporter로부터 가져온다)
    scheme: https
    static_configs:
      - targets: ['{dev 서버 Node Exporter 경로}']
    tls_config:
      insecure_skip_verify: true

  - job_name: 'app_metrics' # 애플리케이션 메트릭 수집 경로 (Spring Actuator을 통해 제공받는다)
    scheme: https
    metrics_path: '/prometheus-provider'
    static_configs:
      - targets: ['{dev 서버 Node Exporter 경로}']
    tls_config:
      insecure_skip_verify: true
```

### 3) 도커 재실행

이제 Prometheus도 실행된다. 메트릭을 Grafana에 띄울 수 있게 됐다. 심지어 본래 확인할 수 없었던 애플리케이션 수준 메트릭도 쉽게 확인할 수 있게 되었다.

<img width="2904" height="1844" alt="3" src="https://github.com/user-attachments/assets/e9424cda-0f27-4626-bff5-1db4428cfc4f" />


## 8. Tempo 설치

### 1) docker-compose.yml 수정

마지막으로 APM을 위해 모니터링 서버에 Tempo를 설치해 주어야 한다. Tempo는 Trace(애플리케이션에서 발생한 요청의 흐름)를 저장하는 서비스이다.

모니터링 서버에는 지금 굉장히 많은 서비스(Grafana, Loki, Prometheus)가 Docker에서 돌아가고 있다. Docker compose로 한 번에 켜고 끄기 쉽도록 Tempo도 Docker로 띄워 보자.

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    ports:
      - "80:3000"
    environment:
      - GF_SECURITY_ADMIN_USER={그라파나 관리자 계정}
      - GF_SECURITY_ADMIN_PASSWORD={그라파나 관리자 비밀번호}
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

  loki:
    image: grafana/loki:3.4.1
    container_name: loki
    ports:
      - "8080:3100"
    volumes:
      - ./loki:/mnt/config
    command:
      - "-config.file=/mnt/config/loki-config.yaml"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
    restart: unless-stopped

  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"   # HTTP API, Grafana가 trace 조회 시 사용
      - "443:4317"   # OTLP gRPC, 앱 서버가 gRPC로 trace 전송 시 사용
      - "4318:4318"   # OTLP HTTP, 앱 서버가 HTTP로 trace 전송 시 사용
      - "9411:9411"   # Zipkin 호환
    volumes:
      - ./tempo:/etc/tempo
      - tempo-data:/var/tempo
    command:
      - "-config.file=/etc/tempo/tempo.yaml"
    restart: unless-stopped

volumes:
  grafana-data:
  prometheus:
  tempo-data:
```

포트가 굉장히 많이 쓰인다. 다행히 우리가 외부에서 사용해야 할 포트는 OTLP GRPC 포트 하나 뿐이다. 이 포트를 활용해서 앱 서버가 Tempo로 트레이스 로그를 전송하기 때문이다(OTLP GRPC 대신 OTLP HTTP를 활용할 수도 있다). 그래서 기본적으로 4317이었던 OTLP GRPC 포트를 443 포트로 매핑하여 열어준다.

### 2) Tempo 설정 파일 생성

```yaml
server:
  http_listen_port: 3100
  grpc_listen_port: 9095

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: "0.0.0.0:4317"
        http:
          endpoint: "0.0.0.0:4318"

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
```

otlp http는 열어줄 필요가 없긴 한데 그냥 열어주었다.

## 9. Tempo Exporter 설정

### 1) opentelemetry 라이브러리 설치

모니터링 서버에 Tempo를 설치하여 trace를 수집하고 저장할 수 있게 됐다. 그렇다면 이제 어플리케이션이 띄워진 서버에서 trace를 계측하여 Tempo로 전송해 주어야 한다.

trace를 계측하여 전송하는 방식은 아래 두 가지로 나뉜다.

- OpenTelemetry Java Agent: 코드 수정 없이 jar 파일을 서버와 함께 동작시키는 방식
- gradle 라이브러리 활용: gradle 설정을 변경하여 trace 계측 설정을 코드 단에서 동작시키는 방식

우리는 gradle 라이브러리를 활용하는 방법이 비교적 빠르고 쉽게 구현할 수 있다고 판단했다. 데모데이가 얼마 남지 않은 상황에서 시간을 절약하는 것이 좋다고 판단했고, 이를 위해 `build.gradle`에 아래 세 가지 라이브러리를 설치해 주었다.

```java
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
implementation 'net.ttddyy.observation:datasource-micrometer-spring-boot:1.2.0'
```

- `io.micrometer:micrometer-tracing-bridge-otel`: Spring Boot에서 일어나는 이벤트에 대한 trace, span를 생성하여 OpenTelemetry(OTel) 형식으로 변환.
- `net.ttddyy.observation:datasource-micrometer-spring-boot`: 애플리케이션의 JDBC(DB) 호출을 감지하고 trace, span 데이터를 생성.
- `io.opentelemetry:opentelemetry-exporter-otlp`: 변환된 span 데이터를 OTLP 프로토콜을 사용하여 Tempo로 전송.

### 2) application.yml 수정

아래 라인을 추가한다.

```yaml
management:
  tracing:
    enabled: ${TRACING_ENABLE}
    sampling:
      probability: 1.0
  otlp:
    tracing:
      transport: grpc
      endpoint: ${TEMPO_GRPC_LISTENER_URL}
jdbc:
  datasource-proxy:
    enabled: ${DB_TRACING_ENABLE}
    include-parameter-values: ${DB_TRACING_ENABLE}
    query:
      logger-name: my.query-logger
  resultset-operations:
    enabled: ${DB_TRACING_ENABLE}
```

- `${TRACING_ENABLE}`
    - trace를 자동으로 계측할 것인지 여부.
    - dev 서버에는 true로, 로컬이나 prod 서버에는 false로 지정한다.
    → dev 서버의 trace만 Tempo에 기록됨.
- `${TEMPO_GRPC_LISTENER_URL}` : `{모니터링 서버의 TEMPO OTLP GRPC 경로}`
- `${DB_TRACING_ENABLE}`
    - SQL문에 대한 trace도 계측할 것인지 여부.
    - 이게 false가 되면 Tempo에서 SQL문은 보이지 않는다. (단순히 DB를 조회했다는 사실만 알 수 있음)

위 환경변수들을 환경에 맞게 `.env`에 등록한다.

해당 단계에서의 참고 자료는 아래와 같다.

https://jdbc-observations.github.io/datasource-micrometer/docs/current/docs/html/

https://deuk9.github.io/blog/monitoring/spring-kafka-trace-monitoring/

### 3) Grafana와 연결

이제 Trace도 Grafana를 통해 확인할 수 있게 되었다.

<img width="2904" height="1844" alt="4" src="https://github.com/user-attachments/assets/38263f0c-0f93-4ef4-a50d-68f7fcf6dea4" />



## 10. prod 서버에서도 한 번 더 작업

이제 dev 서버 연결은 모두 끝났다.

위 과정에서 4번, 6번을 prod 서버에 한 번 더 해주면 prod 서버 대시보드도 만들 수 있다.

Prometheus 같은 경우는 dev 서버의 메트릭과 prod 서버의 메트릭을 분리해주기 위해, 설정 파일을 아래와 같이 수정해 주었다.

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'dev_os_metrics' # OS, 서버 인스턴스 메트릭 수집 경로 (Node Exporter로부터 가져온다)
    scheme: https
    static_configs:
      - targets: ['{dev 서버 Node Exporter 경로}']
        labels:
          environment: dev
    tls_config:
      insecure_skip_verify: true

  - job_name: 'dev_app_metrics' # 애플리케이션 메트릭 수집 경로 (Spring Actuator을 통해 제공받는다)
    scheme: https
    metrics_path: '/prometheus-provider'
    static_configs:
      - targets: ['{dev 서버 Node Exporter 경로}']
        labels:
          environment: dev
    tls_config:
      insecure_skip_verify: true

  - job_name: 'prod_os_metrics' # OS, 서버 인스턴스 메트릭 수집 경로 (Node Exporter로부터 가져온다)
    scheme: https
    static_configs:
      - targets: ['{prod 서버 Node Exporter 경로}']
        labels:
          environment: prod
    tls_config:
      insecure_skip_verify: true

  - job_name: 'prod_app_metrics' # 애플리케이션 메트릭 수집 경로 (Spring Actuator을 통해 제공받는다)
    scheme: https
    metrics_path: '/prometheus-provider'
    static_configs:
      - targets: ['{prod 서버 Node Exporter 경로}']
        labels:
          environment: prod
    tls_config:
      insecure_skip_verify: true

```

이렇게 `environment`라는 라벨을 달면 메트릭 로드 시 dev와 prod 메트릭을 분리할 수 있다.

# 결론

위 과정을 거쳐 핏토링 팀은 Grafana를 활용한 모니터링 서버를 구축할 수 있었다. 모니터링 서버에서 Grafana와 관련 서비스를 모두 켜고 `http://{모니터링 서버 IP}:80`으로 들어가면 Grafana 페이지로 접근할 수 있다.

이 글을 작성하고 있는 지금은 데모데이를 모두 무사히 끝냈다. 기존 방식과 비교하여 Grafana의 장단점을 비교해보면 아래와 같다.

- 모니터링 서버가 굉장히 안정적이다. 기존 Pinpoint APM 서버는 빈번하게 서버 인스턴스가 꺼졌는데, 현재 모니터링 서버는 약 한 달 간 한 번도 꺼진 적이 없다.
- 기존에 산재되어있던 모니터링 항목들(로그, 메트릭, 트레이스)을 하나의 대시보드에서 모두 관리할 수 있다.
- Kafka와 같은 실시간 이벤트 스트리밍 서비스 없이 애플리케이션 수준 메트릭을 확인할 수 있게 되었다.
- 아쉬운 점은, Pinpoint APM처럼 Trace를 시각화해주지 않고 글로만 보여준다. 그러나 기능 자체는 Pinpoint APM과 동일하기 때문에 약간의 수고로움을 감수하면 크게 번거롭지는 않다.

안정적인 모니터링 환경을 구축한 덕분에, 데모데이 전에 스레드풀 및 커넥션풀을 적절하게 조정할 수 있었고, 메모리 누수 문제도 발견하여 해결할 수 있었다. 그 결과 데모데이를 성공적으로 진행할 수 있었다.

Grafana 역시 설치 및 관리가 쉬운 것은 아니라고 생각한다. 위에서 서술한 모든 과정을 문제 없이 성공해야 하며, 서비스 규모가 커지면서 Grafana 로그 롤링 정책 등 고려해야 할 사항이 많아진다. 그러나 아직까지는 굉장히 만족하며 사용하고 있다. 롤링 정책을 고려할 만큼 아직은 서비스의 규모가 크지 않고, 지금의 수준으로도 충분히 원하는 만큼의 모니터링을 진행할 수 있기 때문이다.

물론 앞으로도 모니터링 환경을 조금씩 개선해 나가며, 더 안정적인 서비스를 운영할 수 있도록 노력할 예정이다. Grafana를 비롯한 다양한 오픈소스 도구들이 주는 인사이트 덕분에, 서비스의 성장을 한층 더 가까이에서 지켜볼 수 있었다. 혹시라도 이 글을 보시는 분들 중 모니터링 환경을 구축하고자 하시는 분들이 계신다면 Grafana를 사용해 보시는 것을 추천드린다.
