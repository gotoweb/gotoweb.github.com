---
layout: post
title:  "EC2에서 돌아가는 웹 서버의 로그를 CloudWatch로 보내기"
date:   2021-01-28 00:03:27 +0900
categories: AWS
---

참고: [Quick Start: Install and Configure the CloudWatch Logs Agent on an EC2 Linux Instance at Launch](https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html)

EC2에서 CloudWatch로 로그를 보내기 위해서는, 먼저 에이전트를 EC2 내에 설치해야 합니다. 이후 에이전트를 서비스로 등록해 시작해주면, 에이전트 설정에 따른 로그(일단, 파일에 차례대로 기록하는 것을 기본으로 합니다)를 CloudWatch의 특정 로그 그룹에 누적시킬 수 있습니다.

### 용어
- CloudWatch
  - 기본적으로 모니터링 서비스이며, 로그를 수집할 수 있게 도와줍니다.

## 1. IAM 설정

가장 먼저 해야 할 일은 IAM을 설정해주어야 합니다. IAM을 통해 EC2 인스턴스가 CloudWatch에 로그를 쌓을 수 있도록 권한을 부여할 수 있습니다.

다음과 같은 JSON을 이용해 정책을 만들어야 합니다. 콘솔에서 직접 만들어도 좋고, CLI로 만들어도 됩니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
    ],
      "Resource": [
        "*"
    ]
  }
 ]
}
```

허용하는 액션을 보면, 로그 그룹 및 로그 스트림을 만들고, 로그 이벤트를 생성합니다.

정책은 EC2-CloudWatch-logs 라는 이름으로 만들어 보겠습니다. 정책을 만들었으면, 이제 역할을 만들어야 합니다.

EC2가 사용할 역할이므로, 사용 사례는 EC2를 선택한 후, 연결할 정책은 방금 만든 `EC2-CloudWatch-logs`를 연결해주고, 역할 이름은 `EC2-CloudWatchRole` 이라고 만들어 보겠습니다.

### 용어
- IAM
  - AWS에서 어떤 일을 하기 위해 역할 및 정책을 설정하는 곳이다.
  - 정책
    - AWS 서비스와 관련된 특정 액션을 허용하거나, 거부할 수 있는 정책을 세울 수 있습니다.
  - 역할
    - 여러 가지 정책들을 조합해 하나의 역할을 만들어냅니다.
- CloudWatch
  - 로그 그룹/로그 스트림
    - 로그 스트림은, 실제로 로그가 남아있는 공간입니다.
    - 로그 그룹은 로그 스트림의 모음입니다.

## 2. EC2 인스턴스에 역할 부여하기

EC2 인스턴스 생성을 위해 이미지는 Amazon Linux를 사용하였고, 값들은 기본값으로 설정해주었습니다. 인스턴스를 만들고 난 후, 역할을 부여해야 합니다. 콘솔에서 다음 메뉴를 따라가세요.

인스턴스 선택 -> 작업 -> 보안 -> IAM 역할 수정

메뉴를 누른 후 등장하는 화면에서 아까 만들었던 EC2-CloudWatchRole 을 선택하고 저장합니다. 이제 EC2는 주어진 역할에 따라 CloudWatch에 로그를 기록할 수 있습니다.

## 3. EC2 접속 후 에이전트 설치


### 3-1. 패키지 업데이트
EC2에 접속한 후, 레드햇 기반의 운영체제이므로, yum을 이용해서 각종 패키지를 설치해주게 됩니다. 먼저 패키지 repository를 다음 명령어로 업데이트해줍니다.

```
sudo yum update -y
```


### 3-2. 에이전트 설치
그 후, awslogs 패키지를 설치해야 하는데, 이것이 바로 CloudWatch에 로그를 기록하게 하는 에이전트 프로그램입니다.

```
sudo yum install -y awslogs
```

### 3-3. 환경설정
이제 몇가지 설정이 필요합니다. 먼저 이 글을 보고 있을 대부분의 사용자들이 서울 리전을 사용하고 있을 것이므로, `/etc/awslogs/awscli.conf` 파일을 열어 리전을 ap-northeast-2 로 바꿔줍시다.

`/etc/awslogs/awslogs.conf` 파일은, 어떤 로그를 보낼 것인지 설정하는 파일입니다.

저의 목적은 특정 node 앱을 실행시키고, `console.log`를 이용해 stdout으로 출력되는 것들을 로그로 보내는 것입니다. 리눅스의 작동 원리상 모든 io를 파일로 취급하는 까닭에, `awslogs.conf` 역시 파일을 지정해서 로그를 보내게 되어 있습니다.

따라서, 제가 내릴 수 있는 전략은 node 앱을 실행시키고, stdout을 적당한 파일에 쓰는 방식을 이용할 것입니다.

일단 `/var/log/test.log` 라는 파일을 만들고, 여기 기록되는 모든 line을 CloudWatch 로그로 푸시한다고 가정해봅시다.

다음 내용을 `/etc/awslogs/awslogs.conf`에 추가했습니다.

```
[test.log]
# 섹션 이름을 지정해줬습니다

file = /var/log/test.log
# 푸시할 로그 파일입니다

log_group_name = test.log
# CloudWatch 로그 그룹을 지정합니다. 없으면 만듭니다.

log_stream_name = {hostname}
# CloudWatch 로그 스트림을 지정합니다. 없으면 만듭니다. hostname은 일종의 변수입니다. # 진짜 hostname으로 채워집니다. instance id나 ip address로 만들 수도 있습니다.

datetime_format = %Y-%m-%d %H:%M:%S
```

자 이제, 보낼 준비가 끝났습니다.

## 4. 에이전트 실행

EC2 인스턴스를 만들던 시점에서는 Amazon Linux는 최신 버전인 2 버전이 자동으로 선택되었습니다. 모든 명령어는 Amazon Linux 2 기준입니다.

서비스는 이렇게 시작합니다.

```
sudo systemctl start awslogsd
```

당연히 start 대신 `stop`, `status`가 존재하니 바꿔서 명령을 내리면 되겠습니다. 환경설정을 바꾸고 난 후에는 재시작이 필요합니다.

시스템이 부팅할 때마다, 서비스가 자동으로 실행되길 원한다면, 다음 명령을 내립니다.

```
sudo systemctl enable awslogsd.service
```

## 5. 로그가 잘 기록되는지 확인하기

이제 CloudWatch의 로그 그룹 및 로그 스트림에 로그가 잘 기록되는지 확인해볼 차례입니다.

### 5-1. 테스트
다음 명령어로 테스트해봅시다.

```
echo "hello world" > /var/log/test.log
```

### 5-2. 확인
이 명령어를 실행시키고 난 후, CloudWatch -> 로그 그룹 -> test.log 선택 -> (hostname으로 이름 지어진) 로그 스트림 선택 화면으로 들어가 보면, 로그가 잘 쌓여있을 것입니다.

### 5-3. 적용
이제, 원래의 목적인 node app의 stdout을 test.log로 보내봅시다.

리눅스에서 `>`는 출력을 파일로 덮어씌운다는 의미이며, `>>`는 파일 끝(end of line)에 append한다는 의미입니다.

저는 `&>` 를 사용하여, stdout 및 stderr를 전부 로그로 보내볼 예정입니다. `>`가 하나이므로 기존 로그는 지워질 겁니다.

이 예시에서는 간단한 웹 서버 node 앱을 실행시킬 것인데, 이 앱은 모든 요청에 대한 정보를 stdout에 출력한다고 가정해봅시다. 대략 다음과 같은 로그가 나옵니다.

```
➜ npm start

> simple-express@1.0.0 start
> node server.js

Example app listening at http://localhost:8080
GET /hello
POST /world
```

node 앱의 stdout을 이제 `test.log`에 기록하겠습니다. 백그라운드로 실행시키도록 명령 마지막에 `&`도 붙여주었습니다. (필요에 따라 pid를 찾아 kill해야 합니다.)

```
npm start &> /var/log/test.log &
```

이처럼 앱을 백그라운드로 실행시켰습니다. 필요하다면, 실제로 잘 기록이 되는지 tail 명령어와 -f 옵션을 이용해 실시간으로 파일 끝에 추가되는 로그를 눈으로 볼 수 있습니다.

```
tail -f /var/log/test.log
```

CloudWatch에 웹 서버 로그 보내기가 이렇게 마무리 되었습니다!