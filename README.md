**Table of contents**

- [1. Install OTel Agent](#1-install-otel-agent)
  - [OTel Agent Job Download 하기](#otel-agent-job-download-하기)
  - [Job Definition 수정하기](#job-definition-수정하기)
- [2. Collect Docker container metrics](#2-collect-docker-container-metrics)
  - [Client Server 에서 nomad 설정 수정](#client-server-에서-nomad-설정-수정)
  - [otel-agent.nomad 파일 수정](#otel-agentnomad-파일-수정)
  - [docker 그룹 확인](#docker-그룹-확인)
  - [에이전트 재배포 후 수집 확인](#에이전트-재배포-후-수집-확인)
- [3. Collect Nomad metrics](#3-collect-nomad-metrics)
  - [Client Server 에서 prometheus 메트릭 설정하기](#client-server-에서-prometheus-메트릭-설정하기)
  - [otel-agent.nomad 에서 receiver 설정하기](#otel-agentnomad-에서-receiver-설정하기)
  - [에이전트 재배포 후 수집 확인](#에이전트-재배포-후-수집-확인-1)
- [4. APM Instrumentation](#4-apm-instrumentation)

# 1. Install OTel Agent

## OTel Agent Job Download 하기

아래 Github 주소에서 nomad 용 에이전트 job 파일을 다운로드 받습니다.

[Splunk OpenTelemetry Collector for HashiCorp Nomad](https://github.com/signalfx/splunk-otel-collector/tree/main/deployments/nomad)

```bash
$ wget https://github.com/signalfx/splunk-otel-collector/blob/main/deployments/nomad/otel-agent.nomad
```

## Job Definition 수정하기

다운로드 받은 otel-agent.nomad 파일을 열어 아래 부분을 수정합니다

```bash
job "otel-agent" {
  datacenters = ["dc1"]
  type        = "system"

# Constraint 주석처리
#  constraint {
#    attribute = "${attr.nomad.version}"
#    operator  = "semver"
#    value     = "< 1.8.0"
#  }

    service {
      provider = "nomad" #모든 서비스에 해당 부분 추가
      name = "otel-agent"
      port = "health_check"
      tags = ["health"]


    service {
      provider = "nomad"  #모든 서비스에 해당 부분 추가
      name = "otel-agent"
      port = "otlp"
      tags = ["otlp"]
    }


    task "otel-agent" {


      env {
        SPLUNK_ACCESS_TOKEN = "<o11y_ingest_token>"
        SPLUNK_REALM = "<realm>"
        SPLUNK_MEMORY_TOTAL_MIB = 500
      }
    }

```

# 2. Collect Docker container metrics

일단 위 단계에서 수정 한 파일을 가지고 otel agent job을 배포 해 보면 에이전트가 각 클라이언트 서버에 잘 배포 되는 것을 확인 할 수 있습니다.

```bash
$ nomad job run otel-agent.nomad

==> 2025-03-11T09:43:16+09:00: Monitoring evaluation "0ccecfc9"
    2025-03-11T09:43:16+09:00: Evaluation triggered by job "otel-agent"
    2025-03-11T09:43:18+09:00: Allocation "38bc5e89" created: node "c55ca068", group "otel-agent"
    2025-03-11T09:43:18+09:00: Allocation "e0ace9ab" created: node "85b7e6e9", group "otel-agent"
    2025-03-11T09:43:18+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-03-11T09:43:18+09:00: Evaluation "0ccecfc9" finished with status "complete"
```

그러나 이 상태에서는 에이전트만 배포되고, 컨테이너에서 발생하는 메트릭을 signalfx로 보내지 못하는 상태입니다. docker receiver 를 설정하여 메트릭을 스크래핑 후 o11y cloud로 보내도록 설정 해주어야 합니다

## Client Server 에서 nomad 설정 수정

클라이언트 서버에 콘솔로 접근 후, nomad 설정파일을 엽니다

```bash
$ cd /etc/nomad.d/
$ sudo vi nomad.hcl
```

설정 파일에 아래와 같은 형식으로 볼륨 마운트 설정을 넣어줍니다

```bash
# nomad.hcl

data_dir  = "/opt/nomad"
bind_addr = "0.0.0.0"

client {
  host_volume "dockersock" {
    path      = "/var/run/docker.sock"
    read_only = true
  }
}
```

## otel-agent.nomad 파일 수정

에이전트 설정 파일에 아래와 같이 내용을 추가합니다

```bash
# otel-agent.nomad

job "otel-agent" {


  group "otel-agent" {
    volume "vol" {
      type      = "host"
      read_only = true
      source    = "dockersock"
    }


    task "otel-agent" {
      driver = "docker"

      volume_mount {
        volume      = "vol"
        destination = "/var/run/docker.sock"
        read_only   = true
      }


receivers:
  smartagent/docker-container-stats:
    type: docker-container-stats


service:
  pipelines:
    metrics:
      exporters:
      - debug
      - signalfx
      processors:
      - memory_limiter
      - batch
      - resourcedetection
      receivers:
      - hostmetrics
      - signalfx
      - smartagent/docker-container-stats

```

## docker 그룹 확인

```bash
$ cat /etc/group | grep docker
docker:x:992:ec2-user

그룹정보를 추가합니다. (group_add)

    task "otel-agent" {
      driver = "docker"

      volume_mount {
        volume      = "vol"
        destination = "/var/run/docker.sock"
        read_only   = true
      }

      config {
        image = "quay.io/signalfx/splunk-otel-collector:latest"
        group_add = [
          "992"  # 호스트의 docker 그룹 ID
        ]

        force_pull = true
        entrypoint = [
          "/otelcol",
          "--config=local/config/otel-agent-config.yaml",
          "--metrics-addr=0.0.0.0:8889",
        ]
```

## 에이전트 재배포 후 수집 확인

otel-agent.nomad 파일로 잡을 다시 구동시켜 새로운 설정이 에이전트에 반영 되도록 합니다.

```bash
$ nomad job run otel-agent.nomad

==> 2025-03-11T09:45:30+09:00: Monitoring evaluation "033c1e2e"
    2025-03-11T09:45:30+09:00: Evaluation triggered by job "otel-agent"
    2025-03-11T09:45:30+09:00: Allocation "fedecf6e" created: node "85b7e6e9", group "otel-agent"
    2025-03-11T09:45:30+09:00: Allocation "3f403317" created: node "c55ca068", group "otel-agent"
    2025-03-11T09:45:30+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-03-11T09:45:30+09:00: Evaluation "033c1e2e" finished with status "complete"
```

Nomad UI 페이지로 가서 job이 제대로 구동되고 있는지 확인합니다.
![2-1. Nomad Job Status](./src/images/2-1-nomad-job-status.jpg)

Splunk Observability Cloud 로 가서 컨테이너 메트릭이 유입되고 있는지 확인합니다.
![2-2. O11y Container Dashboard](./src/images/2-2-o11y-container-dashboard.jpg)

# 3. Collect Nomad metrics

이제는 노마드 플랫폼에서 제공하는 노마드에 특화된 메트릭을 추가로 수집해야합니다.
노마드 클러스터를 관리하기 위해서는 리소스를 구분하는 단위가 노마드에서 사용하는 단위 (JOB, Allocation, Server, Client 등) 로 만들어진 메트릭을 가져와야 의미가 있습니다

## Client Server 에서 prometheus 메트릭 설정하기

제일 처음 해야 될 절차는 노마드 컨테이너를 실행시키는 클라이언트 서버에서 Prometheus metrics를 발생(publish) 시키도록 명시적으로 설정을 해 주어야 합니다.

Nomad Server와 Client들의 /etc/nomad.d/nomad.hcml 에서 아래 내용을 추가 합니다.

```bash
telemetry {
  collection_interval        = "5s"
  disable_hostname           = false
  prometheus_metrics         = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

클라이언트 서버로 접속 해 봅시다

```bash
$ cd /etc/nomad.d/
$ sudo vi nomad.hcl
```

설정 파일에 아래와 같은 형식으로 볼륨 마운트 설정을 넣어줍니다

```bash
# nomad.hcl

# plugin "docker" / config 항목에 allow_privileged = true 을 추가 합니다.
plugin "docker" {
  config {
    endpoint = "unix:///var/run/docker.sock"
    allow_privileged = true  # 권한 문제 방지
  }
}

# 아래 텔레메트리 설정을 Client 바깥에 넣습니다
telemetry {
  collection_interval        = "5s"
  disable_hostname           = false
  prometheus_metrics         = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

Nomad Server의 server.hcl

```bash
$ cat /etc/nomad.d/server.hcl

datacenter = "aws"
data_dir = "/opt/nomad"

server {
  enabled          = true
  bootstrap_expect = 1
}

client {
  enabled = false
}

telemetry {
  collection_interval = "5s"
  disable_hostname = false
  prometheus_metrics = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

Nomad Client의 client.hcl

```bash
$ cat /etc/nomad.d/client.hcl
datacenter = "aws"
data_dir = "/opt/nomad"

client {
  enabled = true
  servers = ["172.31.17.214:4647"]

  host_volume "dockersock" {
    path      = "/var/run/docker.sock"
    read_only = false
  }
}

plugin "docker" {
  config {
    endpoint = "unix:///var/run/docker.sock"
    allow_privileged = true  # 권한 문제 방지
  }
}

telemetry {
  collection_interval = "5s"
  disable_hostname = false
  prometheus_metrics = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

Nomad Server / Client 들을 각각 재기동합니다.

```bash
$ sudo systemctl restart nomad
$ sudo systemctl status nomad
```

Nomad Server에 환경변수를 설정합니다.

```bash
$ export NOMAD_VAR_host_node_addr=$HOSTNAME
$ echo $NOMAD_VAR_host_node_addr
ip-172-31-17-214.ec2.internal
```

저는 .bash_profile에 등록하였습니다.

```bash
$ cat .bash_profile
...
export PATH
export NOMAD_VAR_host_node_addr=$HOSTNAME
```

## otel-agent.nomad 에서 receiver 설정하기

에이전트 파일을 열어서 receivers 아래에 있는 설정에 다음과 같이 추가합니다

```bash
variable "host_node_addr" {
  type = string
}

job "otel-agent" {
  datacenters = ["dc1"]
  type        = "system"
....

    task "otel-agent" {
      driver = "docker"
.....
      env {
        SPLUNK_ACCESS_TOKEN = "xxxx"
        SPLUNK_REALM = "lab0"
        SPLUNK_MEMORY_TOTAL_MIB = 500
        HOST_NODE_ADDR = var.host_node_addr
      }
....

receivers:
  prometheus/nomad:
    config:
      scrape_configs:
      - job_name: nomad
        scrape_interval: 10s
        metrics_path: /v1/metrics
        params:
          format: ['prometheus']
        static_configs:
        - targets:
          - ${HOST_NODE_ADDR}:4646

processors:
  resourcedetection/os:
    detectors:
      - system
    system:
      hostname_sources:
        - os

service:
  pipelines:
    metrics:
      exporters:
      - debug
      - signalfx
      processors:
      - memory_limiter
      - batch
      - resourcedetection
      - resourcedetection/os
      receivers:
      - hostmetrics
      - signalfx
      - smartagent/docker-container-stats
      - prometheus/nomad
```

## 에이전트 재배포 후 수집 확인

otel-agent.nomad 파일로 잡을 다시 구동시켜 새로운 설정이 에이전트에 반영 되도록 합니다.

```bash
$ nomad job run otel-agent.nomad

==> 2025-03-11T09:45:30+09:00: Monitoring evaluation "033c1e2e"
    2025-03-11T09:45:30+09:00: Evaluation triggered by job "otel-agent"
    2025-03-11T09:45:30+09:00: Allocation "fedecf6e" created: node "85b7e6e9", group "otel-agent"
    2025-03-11T09:45:30+09:00: Allocation "3f403317" created: node "c55ca068", group "otel-agent"
    2025-03-11T09:45:30+09:00: Evaluation status changed: "pending" -> "complete"
==> 2025-03-11T09:45:30+09:00: Evaluation "033c1e2e" finished with status "complete"
```

Running 상태의 Allication ID 확인하고, 아래 명령으로 Node server host가 출력되는지 확인합니다.

```bash
$ node job status otel-agent

$ node alloc exec <alloc_id> sh

$ echo $HOST_NODE_ADDR
```

Nomad UI 페이지로 가서 job이 제대로 구동되고 있는지 확인합니다.
![2-1. Nomad Job Status](./src/images/2-1-nomad-job-status.jpg)

Splunk Observability Cloud 로 가서 노마드 메트릭이 유입되고 있는지 확인합니다.
![2-2. O11y Container Dashboard](./src/images/3-1-o11y-nomad-metrics.jpg)

# 4. APM Instrumentation

작성 예정
