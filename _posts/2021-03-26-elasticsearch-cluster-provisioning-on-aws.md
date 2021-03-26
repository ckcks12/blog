---
layout: post
title: AWS 에 Elasticsearch Cluster 띄우기
author: Eunchan Lee
categories: [software,aws,asg,elasticsearch,cluster,cloudinit]
---

새로 클러스터를 개편하기 전에는 Packer 로 Elasticsearch 이미지를 만들어 EC2 에 활용하였습니다.

Nori 형태소 분석기를 사용 중이었고 사전을 교체하려면 모든 EC2 노드에 SSH 접근하여 바꿔주어야 했습니다.

또한 elasticsearch.yml 이나 jvm.options 등의 설정을 바꿔 줄 때에

1. 이미지를 새로 굽거나
2. 모든 노드에 SSH 접근하여 수동으로 바꾸어 주거나

했어야 합니다. 물론 Ansible 을 쓸 수도 있긴 했지만요. ㅎㅎ

따라서 개편을 통해 ASG(Auto Scaling Group) Template 활용하여 관리 포인트 줄이고 scale-out 가능하게 만들려고 했어요.

이와 함께 커널 튜닝과 인덱서 분리 등을 통해서 전체적으로 성능 최적화를 이끌어 내기도 했어요.

# ASG 를 사용한 이유?

ES 는 Kubernetes 에도 띄울 수 있다고 하죠.

공식 ES 홈페이지에서 제공해주는 one yaml 도 봤는데... 절대 쉽지 않아 보였어요.

아무래도 Disk 에 민감한.. 어떻게 보면 Database 이니까요.

Kubernetes 의 언제나 죽을 수 있다는 가정에 좋지 않고.. Persistance Volume 도 그닥..

그래서 EC2 를 직접 Orchestration 해야 했어요.

이때 ASG 의 Template 을 활용하면 정말 꽤나 멋지게 쓸 수 있습니다.

# Cloud Init

EC2 에 user_data 로 shell script 를 넣을 수 있을 뿐 아니라 Cloud Init 을 사용할 수 있습니다.

Cloud Init 은 Cloud Infra 의 Instance 를 초기화할 때 유용하게 쓸 수 있는 툴입니다.

더 자세한 내용은 [https://cloud-init.io/](https://cloud-init.io/) 에서 확인해 보세요.

회사에서 ES Instance 에 사용한 `cloudinit.yaml` 을 보여드리겠습니다.

```yaml
#cloud-config
timezone: Asia/Seoul

runcmd:
  # kernel tuning
  - |
    echo "kernel tuning" >> /var/run/cloud-init/log
    swapoff -a
    sysctl -w vm.max_map_count=262144
    ulimit -u 65535
  # update instance/node name by private ip
  - |
    echo "update instance/node name by private ip" >> /var/run/cloud-init/log
    export INSTANCE_ID=$(cat /run/cloud-init/instance-data.json | jq -rM '.ds["meta-data"]["instance-id"]')
    export IP=$(cat /run/cloud-init/instance-data.json | jq -rM '.ds["meta-data"]["local-ipv4"]')
    aws ec2 create-tags --resource $INSTANCE_ID --tags "Key=Name,Value=${node_name}-$IP"
    sed -i "s/${node_name}/${node_name}-$IP/g" /services/config/elasticsearch/elasticsearch.yml
  # docker-compose up
  - |
    echo "docker-compose up" >> /var/run/cloud-init/log
    docker-compose -f /services/app/docker-compose.yml up -d

write_files:
  # kernel tuning limit
  - path: /etc/security/limits.conf
    content: |
      *       soft    nofile  65536
      *       hard    nofile  65536
      root    soft    nofile  65536
      root    hard    nofile  65536
      elasticsearch  -  nofile  65535

      *       soft    nproc  65536
      *       hard    nproc  65536

      *       soft    memlock unlimited
      *       hard    memlock unlimited
    append: true
  # elasticsearch.yaml
  - path: /services/config/elasticsearch/elasticsearch.yml
    content: |
      ${elasticsearch_yml}
  # jvm.options
  - path: /services/config/elasticsearch/jvm.options
    content: |
      ${jvm_options}
  # log4j2.properties
  - path: /services/config/elasticsearch/log4j2.properties
    content: |
      ${log4j2_properties}
  # datadog.yaml
  - path: /services/config/datadog-agent/datadog.yaml
    content: |
      ${datadog_yaml}
  # metricbeat
  - path: /services/config/metricbeat/metricbeat.yml
    content: |
      ${metricbeat_yml}
  # docker-compose.yml
  - path: /services/app/docker-compose.yml
    content: |
      ${docker_compose_yml}
```

`timezone` 을 서울로 맞춥니다.

`write_files` 로 필요한 파일들을 만들어 냅니다. 이때 `append` 로 이어쓸 수 있습니다.

`runcmd` 필요한 명령어들을 실행시킬 수 있습니다.

정말 기본이 되는 커널 튜닝들을 진행했습니다.

`/run/cloud-init/instance-data.json` 에는 현재 instance 와 관련된 정보들이 들어있습니다.

Cloud Vendor 마다 다르지만 일단은 AWS 의 경우 VPC 부터 Security Group, Subnet, Instance Type, IPv4 등을 확인할 수 있습니다.

저는 클러스터의 관리 용이성을 위해 이 곳에서 IP를 가져와 ES 의 `node_name` 에 사용했습니다.

이렇게 되면 해당 `cloudinit.yaml` 을 갖는 ASG 들은 create/boot 될 때 알아서 필요한 ES Node 로서의 정보를 갖고 세팅되게 됩니다.

또한 설정 값을 바꾸거나 사전을 교체할 시에도 `cloudinit.yaml` 을 적절히 수정하고 ASG 내의 새로운 Instance 를 띄우면 수정 내용이 반영된 상태로 세팅 됩니다.

# ASG Instance Refresh

ASG Template 을 수정한 뒤에 해당 내용으로 모든 Instance 가 교체 되길 원할 때 유용하게 사용할 수 있는 것이 ASG 의 Instance Refresh 입니다.

AWS 문서 : [https://aws.amazon.com/blogs/compute/introducing-instance-refresh-for-ec2-auto-scaling/#:~:text=You can trigger an Instance,ASG terminates and launches instances](https://aws.amazon.com/blogs/compute/introducing-instance-refresh-for-ec2-auto-scaling/#:~:text=You%20can%20trigger%20an%20Instance,ASG%20terminates%20and%20launches%20instances).

`minimum health percentage` 와 `warm up seconds` 를 통해 안전하게 ASG 의 Instance 들을 교체할 수 있습니다.

ES 의 경우 Node 하나가 없어지면 알아서 Replica 를 Primary 로 승격시킨 뒤 Replication 을 추가로 진행하는 등 굉장히 안정적인 클러스터 관리가 가능합니다.

따라서 Node 하나씩 Instance 교체를 한다면 데이터 손실 없이 클러스터 내의 모든 Node 에 최신 변경 사항을 반영시킬 수 있습니다.

저는 아래처럼 aws cli 를 활용한 Shell Script 를 만들었습니다.

```bash
#!/bin/bash

region=""
asg_name=""
min_health=""
warmup_seconds=""

usage() {
  echo "
Description: AutoScalingGroups Instance Reresh
Usage: $(basename $0)
  -r region (ap-northeast-2)
  -n autoscaling group name (search-blue-alpha-kr-data)
  -m min health percentage (100)
  -s warmup seconds (300)
  [-h help]
"
exit 1;
}

while getopts 'r:n:m:s:h' optname; do
  case "${optname}" in
    h) usage;;
    r) region=${OPTARG};;
    n) asg_name=${OPTARG};;
    m) min_health=${OPTARG};;
    s) warmup_seconds=${OPTARG};;
    *) usage;;
  esac
done

[ -z "${region}" ] && >&2 echo "Error: -r region required" && usage
[ -z "${asg_name}" ] && >&2 echo "Error: -n asg_name required" && usage
[ -z "${min_health}" ] && >&2 echo "Error: -m min_health required" && usage
[ -z "${warmup_seconds}" ] && >&2 echo "Error: -s warmup_seconds required" && usage

res=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$asg_name" --region "${region}" --output json)
asg=$(echo "$res" | jq -rM '.AutoScalingGroups[0]')
cnt=$(echo "$asg" | jq -rM '.DesiredCapacity')

echo "$asg_name(cnt: $cnt)"

res=$(aws autoscaling start-instance-refresh \
  --region "${region}" \
  --auto-scaling-group-name "$asg_name" \
  --preferences MinHealthyPercentage="$min_health",InstanceWarmup="$warmup_seconds")
id=$(echo "$res" | jq -rM '.InstanceRefreshId')

watch aws autoscaling describe-instance-refreshes \
  --region "${region}" \
  --auto-scaling-group-name "$asg_name" \
  --instance-refresh-ids "$id"

echo "done"
```

`MinHealthyPercentage` 의 경우 99 또는 100 을 주어서 최소한의 Instance 종료로 교체를 할 수 있습니다.

`WarmupSeconds` 는 보통 10분 이상을 줍니다. Shard Replication/Assign 되는 데에 사용되는 시간입니다.

위 Shell Script 는 아래와 같이 사용할 수 있습니다.

```bash
./asg-refresh.sh -r ap-northeast-2 -n search-blue-alpha-kr-data -m 100 -s 900
```

# Terraform 으로 Provisioning 하기

위에서 ASG 와 Cloud Init 을 이야기 하느라 조금 새긴 했는데..

이제 위의 것들을 어떻게 Terraform 으로 Provisioning 하는가?

일단 `aws_instance` 나 `aws_autoscaling_group` 등 기초 리소스에 대한 설명은 빼고 핵심만 보여드리겠습니다.

우선 `aws_launch_template` 에 `template_cloudinit_config` 을 활용하여 이렇게 넣어줍니다.

```
data "template_cloudinit_config" "cloudinit" {
  gzip = true
  part {
    content_type = "text/cloud-config"
    content = <<EOF
cloudinit.yaml 내용을 이 곳에
EOF
  }
}

resource "aws_launch_template" "lt" {
	user_data = data.template_cloudinit_config.cloudinit.rendered
}
```

그리고 Cloud Init 을 잘 활용해서 `elasticsearch.yml` 등을 잘 집어 넣으시면 Terraform 내에서 아래와 같이 사용하실 수 있어요.

![](https://i.imgur.com/6YfyJGE.png)

![](https://i.imgur.com/SrY5yUj.png)

사실 이 글에서 Terraform 소스와 구조를 모두 보여드리기 어렵겠더라구요.

만약 원하시는 분이 계시다면 라이브러리화 해서 Github 에 올려볼게요!



