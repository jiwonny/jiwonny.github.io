---
categories: projects
title: AWS NEPTUNE 적용기 (2) - 설치하기
excerpt: "2020-12-01"
taxonomy: projects
toc: true
toc_sticky: true
toc_label: 목차
tags:
    - AWS
author_profile: true
---
<br>

> Neptune과 그래프 데이터베이스에 대한 포스트 바로가기<br>[AWS NEPTUNE 적용기(1)](/projects/aws-neptune-1)

<br>

# AWS CloudFormation으로 설치하기
*참조*  
[AWS CloudFormation을 사용해 Neptune DB 클러스터 생성](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/get-started-create-cluster.html)
<br>

Neptune DB 인스턴스를 생성하고 사용하기 위해서는 Neptune 외에도 EC2, VPC, IAM과 관련된 설정들이 필요한데, `AWS CloudFormation`을 사용하면 별 다른 작업 없이 이 리소스들을 편리하게 생성할 수 있다. AWS에서 Neptune 설치에 필요한 `AWS CloudFormation` 템플릿을 제공해주고 있기 때문에, CloudFormation 스택을 활용해 설치 작업을 시작할 수 있다.
<br>

Neptune 클러스터를 수동으로 생성하고자 할 경우 [콘솔을 사용하여 Neptune DB 시작하기](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/manage-console-launch.html)를 참고하도록 하자.

<br>

# Neptune DB 클러스터 생성
## 1. 스택 시작하기
위 링크의 **AWS CloudFormation 스택을 사용하여 Neptune DB 클러스터 생성** 부분에서 스택을 시작할 수 있다.
![cloudformation launch stack](/assets/images/aws-neptune/cloudformation1.png)  

Amazon Neptune은 서울 리전도 지원하므로 서울 리전에 해당하는 스택을 생성하면 된다.

<br>

![cloudformation launch stack2](/assets/images/aws-neptune/cloudformation2.png)  

스택을 생성하기 위해서는 템플릿이 필요한데, 이 단계에서는 위의 설정을 그대로 냅둔 상태에서 다음 단계로 넘어가면 된다. 

<br>

## 2. 스택 세부 정보 지정
다음은 **스택 세부 정보**를 지정하는 단계이다.
![cloudformation launch stack3](/assets/images/aws-neptune/cloudformation3.png)  

여기서 유의해야할 부분은 다음과 같다.
- `DBinstanceType` : 요금과 스펙을 확인해서 변경
- `EC2ClientInstanceType` : 요금과 스펙 확인하여 변경
- `EC2SSHKeyPairName` : 이미 생성한 key가 있다면 key 선택

**DB instance Type과 EC2 instance Type**을 결정하는 부분인데 기본 설정은 위 사진과 같이 `db.r5.large`와 `r5.2xlarge`로 되어있다. 만약 저대로 두면 요금 폭탄을 맞을 수 있기 때문에 instance 가격을 확인하여 잘 설정하도록 한다.
<br>

나는 처음에 default 값으로 해뒀다가 이틀 사이에 요금 폭탄을 맞고 서둘러 바꿨다. 대규모 프로젝트는 아니어서 `db.t3.medium`으로 충분할 것 같아 가장 싼 타입으로 변경했다. EC2 인스턴스는 Neptune 클라이언트로도 쓰지만 웹서버로도 쓰고 있어서 `t2.micro`(프리티어 사용 가능한 타입)이 아닌 `t3.micro`를 선택했다. 인스턴스 타입별 스펙과 가격은 [EC2 온디맨드 요금](https://aws.amazon.com/ko/ec2/pricing/on-demand/)에서 확인할 수 있다.
<br>

다음은 **SSH Key Pair**를 선택하는 부분이다. 이전에 EC2 인스턴스를 생성한 적이 있다면, EC2 instance를 원격에서 접속하기 위한 `EC2 SSH key pair`를 발급받았을 것이다. Neptune을 설치하면서 생성하는 EC2에서도 마찬가지로 같은 key를 사용하여 SSH 접속이 가능하다. 드롭다운 목록에서 생성한 key를 선택하고 다음 단계로 넘어가면 된다.

<br>

### EC2가 왜 필요하지?
>**EC2가 Neptune 생성에 필요한 이유**<br><br>
위 사진에서도 볼 수 있듯이, Neptune 스택 설정을 하다 보면 EC2 인스턴스와 관련된 설정도 필요하다. EC2는 Neptune 을 사용하기 위한 Client로써 필요한데, 이는 Neptune 데이터 보호를 위한 보안 때문이다.<br>Neptune DB 클러스터는 VPC에서만 생성할 수 있는데, 이 DB instance는 Private Subnet에 위치하기 때문에 같은 VPC 내에 있는 리소스만 Neptune에 접근할 수 있다. 따라서, EC2 인스턴스를 생성하는 것은 **같은 VPC 내에 Neptune에 접근할 수 있는 리소스**를 생성하는 과정이다.<br><br>![aws-vpc-ec2](/assets/images/aws-neptune/vpc-ec2-diagram.png)*출처 https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/security-vpc.html*<br><br>위의 그림처럼 VPC 내에 있는 EC2 instance가 Neptune에 접근할 수 있는 **중간 다리 역할**을 한다.


<br>

## 3. 스택 옵션 구성
다음은 **스택 옵션 구성** 단계이다.  
![cloudformation launch stack4](/assets/images/aws-neptune/cloudformation4.png)  

`IAM` 역할을 선택할 수 있는데, 다른 설정을 하지 않아도 CloudFormation이 알아서 IAM 역할을 생성해주기 때문에 바로 다음 단계로 넘어가도 괜찮다.

<br>

## 4. 검토하기
검토 페이지에서 지금까지 설정한 것들을 확인하고 맨 밑으로 내리면 다음과 같은 selectbox를 만나게 될 것이다. 
![cloudformation launch stack5](/assets/images/aws-neptune/cloudformation5.png) 

첫 번째 확인란은 IAM 리소스를 생성하는 것을 승인하는 부분인데, 이를 통해 Neptune에 접근을 제어할 수 있다. 두 번째 확인란으로 `CAPABILITY_AUTO_EXPAND`를 승인한다.
<br>

다음으로 **생성**버튼을 누르면 Neptune 클러스터를 포함해 필요한 리소스들(IAM, EC2, VPC)이 생성된다. 

<br>

# 생성된 리소스 확인하기
## Neptune Console
CloudFormation으로 Neptune DB와 관련 리소스 설정을 마쳤다면 `Neptune Console`에서 클러스터가 잘 생성되었는지 확인해보자. 
![neptune console](/assets/images/aws-neptune/neptune-console.png) 

두개의 DB가 생성된 것을 확인할 수 있을 것이다. 하나는 `cluster`, 나머지 하나는 `writer`로 생성되어있다. 
<br>

## VPC Console
### VPC
위에서 언급했듯, Neptune instance와 EC2 instance는 VPC내에 생성된다. 이 VPC는 미리 프로비저닝된 상태로 새로 생성되는데, VPC 콘솔에서 확인할 수 있다. 이전에 따로 생성한 것이 없다면 총 두 개의 VPC를 볼 수 있을 것이다. 

![vpc console](/assets/images/aws-neptune/vpc-console.png) 

이름이 별도로 설정되어 있지 않은 VPC는 default VPC이고 `Neptune-test`와 같은 이름으로 설정된 VPC는 cloudformation 템플릿에 의해 생성된 VPC이다. 알아보기 편한 이름으로 변경하면 된다.
<br>

생성된 VPC에서 태그 창을 클릭하면, 해당 VPC가 cloudformation을 통해 생성되었음을 확인할 수 있다.
<br><br>

### Subnet
VPC 콘솔 왼쪽 메뉴에서 서브넷 메뉴를 확인할 수 있을 것이다. 서브넷 콘솔에서 여러 서브넷이 생성되어 있는 것을 볼 수 있다.
![subnet console](/assets/images/aws-neptune/subnet-console.png) 

VPC에 따라 정렬하면, 새로 생성된 VPC에 해당하는 서브넷이 여러개 있음을 볼 수 있다. 위 사진의 빨간 박스 위에 있는 서브넷은 필자가 별도로 생성한 것이고, 박스 안에 있는 서브넷 네개가 cloudformation을 통해 생성된 서브넷이다. 각각 가용 영역마다 `private subnet`이 생성되어 있고, 하나의 `public subnet`이 생성되었음을 볼 수 있다.

원래는 별도의 이름 없이 생성되지만 나는 편리한 구분을 위해 가용 영역에 맞게 이름을 설정해뒀다. (`private_2c`는 2c 가용영역에 있는 private subnet을 의미한다.)  

각 가용영역에 있는 private subnet은 Neptune cluster와 관련이 있다. Neptune Console에서 `Writer instance`를 누르면 세 개의 서브넷을 볼 수 있을 것이다. 세 개의 서브넷 모두 private subnet이다. 즉, **Neptune 인스턴스는 VPC 내에서 private subnet에 위치**하는 것이다. 

한편, public subnet은 이후 살펴볼 EC2 instance랑 관련이 있다. cloudFormation을 통해 생성했던 **EC2 client instance는 Neptune instance와 같은 VPC 안에서 public subnet에 위치**한다.  

Neptune 인스턴스가 private subnet에 위치해있어도 EC2가 같은 VPC 내에 있기 때문에 Neptune에 접근할 수 있다. 또한 이 EC2 instance가 public subnet에 위치해있어 로컬과 같이 VPC 바깥 환경에서도 EC2를 통해 Neptune에 엑세스가 가능한 것이다.

> **Private Subnet인지 Public Subnet인지 어떻게 알지?**<br><br>각 서브넷마다 `Routing Table`이 연결되어 있는데, 대상이 `igw-xxx`인 행이 있다면 `public subnet`이고, 대상이 `nat-xxx`인 행이 있다면 `private subnet`이다.<br><br>
![routing table](/assets/images/aws-neptune/routing-table.png)<br><br>위 사진처럼 igw- 가 있는 라우팅 테이블과 연결되어있다면 `public subnet`이다. *VPC와 subnet 관련해서 더 자세한 부분은 나중에 포스팅해볼 생각이다.*

<br>

## EC2 Console
### EC2
EC2 console에서도 cloudFormation 템플릿을 통해 생성된 EC2 instance를 확인할 수 있다.

![ec2 console](/assets/images/aws-neptune/ec2-console.png)
<br>

`VPC ID`를 통해 해당 인스턴스가 Neptune과 같은 VPC 내에 있음을 볼 수 있다. 또한, 위에 언급했던 대로 해당 인스턴스가 VPC의 `public subnet`에 위치해있다. 

또한, CloudFormation으로 EC2 instance를 생성했다면, 인스턴스 플랫폼은 아마도 Amazon Linux로 되어있을 것이다. 
<br>

### Security Group
해당 EC2 instance의 보안 탭을 누르면, 인스턴스에 적용된 보안 규칙 확인이 가능하다. `Security group`의 `inbound rules`를 확인하면 되는데, 사용자 지정 TCP 타입에서 port `8182`가 허용된 행이 있다. 
![security group](/assets/images/aws-neptune/security-group.png)  
<br>

PORT 22는 EC2 인스턴스에 SSH으로 접속하기 위한 인바운드 규칙이고, TCP 8182 인바운드 규칙을 통해 Neptune에 접속할 수 있는 포트를 열어준 것이다.

<br><br>

# 마치며
AWS cloudFormation으로 복잡한 설정 없이 간단하게 Neptune과 필요한 리소스들을 생성할 수 있었다. 다음 포스트에서는 쿼리 언어인 `Gremlin`에 대해 다뤄보고자 한다.

<br><br>
