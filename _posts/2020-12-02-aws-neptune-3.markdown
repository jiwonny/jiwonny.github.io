---
categories: projects
title: AWS NEPTUNE 적용기 (3) - Gremlin 사용하기
excerpt: "AWS NEPTUNE 적용기 세번째! Gremlin 콘솔 사용하기"
taxonomy: projects
toc: true
toc_sticky: true
toc_label: 목차
tags:
    - AWS
author_profile: true
---
<br>

> Neptune 설치 방법 포스트 바로가기<br>[AWS NEPTUNE 적용기(2)](/projects/aws-neptune-2)

<br>
Neptune 그래프 데이터베이스에 쿼리를 실행할 수 있는 `Gremlin`을 사용하기 위해 `Gremlin` 콘솔을 EC2에 설치했다. 이 때, EC2는 Neptune 클러스터에 접근하기 위해 CloudFormation 템플릿을 사용해 생성한 것으로, 생성 방법은 이전 포스트에 적어두었다.   
Gremlin을 통해 그래프 데이터베이스에 쿼리를 실행할 수 있는데, 이 콘솔을 EC2에 설치하는 이유는 동일한 VPC 내에서만 Neptune에 접근이 가능하기 때문이다.

*참조*  
[AWS Neptune Gremlin 콘솔 설정 가이드](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/access-graph-gremlin-console.html)
<br>

# Gremlin 콘솔 설치
## 1. EC2 SSH 접속
먼저 EC2에 SSH 접속이 필요하다. pem 키가 저장되어 있는 경로를 사용하여 생성한 EC2에 접속한다. cloudformation으로 EC2 인스턴스를 생성했다면 사용자 이름은 `ec2-user`일 것이다. 정확한 정보는 EC2 console에서 확인할 수 있다.  
콘솔에서 퍼블릭 IPv4 DNS를 확인할 수 있는데, EC2에 Elastic IP를 할당한 경우 DNS 주소 외에도 퍼블릭 IPv4 주소를 사용해서 SSH 접속이 가능하다.

```shell
ssh -i <pem key 저장 경로>/<pem key 이름>.pem ec2-user@<ec2 퍼블릭 IPv4 DNS>
```
<br>

## 2. EC2에 Java 8 설치
Gremlin 콘솔 사용을 위해서는 EC2에 먼저 java8이 설치되어 있어야 한다. 아래 명렁어를 통해 java8을 설치한다.
```shell
sudo yum install java-1.8.0-devel
```
<br>

설치 이후 java8을 EC2의 기본 java로 설정한다. 다음 명령어를 입력하면 아래와 같은 선택창을 볼 수 있다. Java 1.8에 해당하는 번호를 입력하여 설정을 마친다.
```shell
sudo /usr/sbin/alternatives --config java
```
![gremlin-java1](/assets/images/aws-neptune/gremlin-java1.png)  

<br>

## 3. Gremlin 다운로드 및 설치
이제 Gremlin을 다운로드하고 설치할 차례이다. 그 전에, 지원되는 Gremlin 버전을 확인해야 한다. Neptune 엔진 버전을 확인하고 그에 호환되는 Gremlin 버전을 다운로드하면 된다. [Neptune 엔진을 확인하는 페이지](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/engine-releases.html)에서 다음과 같은 목록 확인이 가능하다.  
![neptune-version](/assets/images/aws-neptune/neptune-version.png)  
<br>

Neptune 엔진 버전에 맞는 링크로 들어가면 지원되는 쿼리 언어 버전을 확인할 수 있다.  
![gremlin-version](/assets/images/aws-neptune/gremlin-version.png)  
<br>

이 쿼리 언어 버전에 맞게 Gremlin을 다운로드한다. [Download Apache TinkerPop link](https://tinkerpop.apache.org/downloads.html)에 있는 Gremlin console을 아래 명령어를 통해 설치할 수 있다. 만약 3.4.2 버전을 다운로드한다면, 다음 명령어를 입력하면 된다.

```shell
wget https://archive.apache.org/dist/tinkerpop/3.4.3/apache-tinkerpop-gremlin-console-3.4.3-bin.zip
```  
<br>

다운로드한 Gremlin 콘솔 zip 파일의 압축을 해제하고 해당 디렉토리로 이동한다.
```shell
unzip apache-tinkerpop-gremlin-console-3.4.3-bin.zip  
cd apache-tinkerpop-gremlin-console-3.4.3
```  

<br>

## 4. CA 인증서 설치
Gremlin콘솔에는 인증서가 필요하기 때문에 아래 명령어를 통해 인증서를 다운로드한다. 이 단계는 공식 문서를 그대로 따르면 된다. 
```shell
wget https://www.amazontrust.com/repository/SFSRootCAG2.cer
```
```shell
mkdir /tmp/certs/
```
<br>

아래의 jre_path는 `/usr/lib/jvm/{선택한 java version}` 으로 타고 들어가면 jre directory가 있을 것이므로 이를 사용하면 된다. 
```shell
cp {@jre_path}/lib/security/cacerts /tmp/certs/cacerts
```
<br>

위와 같이 인증서를 새 디렉터리에 복사한 이후 리포지토리에 Amazon 인증서를 추가한다.  
```shell
sudo keytool -import \
             -alias neptune-tests-ca \
             -keystore /tmp/certs/cacerts \
             -file /home/ec2-user/SFSRootCAG2.cer \
             -noprompt \
             -storepass changeit
```

<br>

## 5. Neptune 클러스터 Endpoint 연결하기 
AWS Neptune 콘솔에서 Neptune 클러스터 엔드포인트를 복사해와야한다. Neptune 콘솔에 두 DB identifier 중 Role이 `cluster`인 것을 누르면 endpoint를 볼 수 있다. 이 endpoint를 사용하여 `neptune-remote.yaml`을 생성한다.
<br>

현재 디렉토리가 `apache-tinkerpop-gremlin-console-3.4.1/`인지 확인하고 하위 폴더 `conf/`에 아래의 텍스트로 `neptune-remote.yaml` 파일을 생성하여 다음 텍스트를 넣는다.

```shell
hosts: [your-neptune-endpoint]
port: 8182
connectionPool: { enableSsl: true,  trustStore: /tmp/certs/cacerts }
serializer: { className: org.apache.tinkerpop.gremlin.driver.ser.GryoMessageSerializerV3d0, config: { serializeResultToString: true }}
```  
<br><br>


# Gremlin 콘솔 사용하기
## 1. Gremlin 콘솔 실행
위의 단계를 다 마쳤다면, 다시 `apache-tinkerpop-gremlin-console-3.4.1/` 디렉토리로 이동하여 콘솔을 실행한다. 
```shell
bin/gremlin.sh
```

이렇게 콘솔을 실행하면, 귀여운 친구가 출력되어 나온다..
![gremlin-console1](/assets/images/aws-neptune/gremlin-console1.png)  
<br>

콘솔을 실행한 이후, 쿼리를 실행하기 위해서 Neptune DB 인스턴스에 연결하여 원격 모드로 전환해야 한다. `conf/neptune-remote.yaml`에 Neptune 엔드포인트를 기입해두었으므로, 아래와 같이 입력하면 된다. 
```shell
:remote connect tinkerpop.server conf/neptune-remote.yaml
:remote console
```
<br>

## 2. 쿼리 실행하기
쿼리 실행과 관련한 문법은 [Gremlin Tinkerpop Recipe](https://tinkerpop.apache.org/docs/current/recipes/)와 [Gremlin을 사용하여 그래프에 엑세스](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/get-started-graph-gremlin.html)에서 확인할 수 있다. 이 단계에서는 테스트 용도로 간단한 쿼리를 실행해보고자 한다.  
<br>

아래 쿼리는 DB에 존재하는 모든 vertex(node)의 개수를 출력한다. 만약 Neptune DB에 별도의 데이터를 로드하거나 추가하지 않았다면 0이 출력된다.
```shell
g.V.count()
```
<br>

### Vertex 추가하기
새로운 vertex를 추가하기 위해서는 `label`과 `property`를 지정해준다. 만약 `id`를 별도로 지정해주지 않을 경우, 자동적으로 `string`값으로 된 id가 지정된다.  
`id`를 지정할 경우의 쿼리는 다음과 같다.  
```shell
g.addV('video_review').property(id, 'vr_1').property('name', '영상1')
```
`id`를 별도로 지정할 경우 위와 같이 property로 지정하는데, 이 때 property의 키워드는 따옴표로 묶지 않는다.  
<br>

`id`를 지정하지 않는 경우는 아래와 같다.  
```shell
g.addV('video_review').property('name', '영상1')
```

이렇게 vertex를 추가한 뒤 vertex의 개수를 확인하는 쿼리를 실행하면, 결과가 **1**로 나옴을 확인할 수 있다.  
<br>

### Edge 추가하기
`Edge`는 존재하는 Vertex 사이에 추가하여 vertex끼리 연결되도록 할 수 있다. 이를 위해 다른 vertex를 더 추가한다. 
```shell
g.addV('keyword').property(id, 'k_1').property('name', '파스타').property('category', '양식')
```
<br>

`video_review` vertex가 '파스타'라는 키워드를 갖고 있다는 관계를 표현하고자 위에서 `keyword` vertex를 추가했다. `vr_1`이 `k_1`사이의 관계 label을 `has`로 지정한다면, 다음과 같이 추가할 수 있다. 
```shell
g.V('vr_1').addE('has').to(g.V('k_1'))
```
<br>

만약 이 edge에 관계의 질/가중치를 표현하기 위해 `weight`와 같은 `property`를 추가한다면,
```shell
g.V('vr_1').addE('has').to(g.V('k_1')).property('weight', 0.5)
```
<br>

### 그래프 순회
`label`값을 기준으로 vertex를 얻고자 한다면 다음 쿼리를 실행한다. `video_review` label을 가진 모든 vertex를 반환한다. 
```shell
g.V().hasLabel('video_review')
```
<br>

edge의 label을 기준으로 특정 vertex와 연결된 vertex를 얻고자 한다면 `out()`을 사용할 수 있다. `property` 중 'name'이 '영상1'인 'video_review' vertex에 'has'로 연결된 vertex를 찾고자 한다면, 다음 쿼리를 실행한다.
```shell
g.V().has('video_review', 'name', 'vr_1').out('has')
```

위 쿼리를 실행했을 경우, **v['k_1']** 결과를 얻을 수 있다.
<br>

## 3. 콘솔 종료하기
EC2에서 실행하던 콘솔은 아래 명령어로 종료할 수 있다. 
```shell
:exit
```
<br>

# 마치며
위에서와 같이 쿼리를 통해 데이터를 삽입할 수도 있지만, [기존 그래프를 Amazon Neptune으로 마이그레이션](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/migrating.html)이나 [대량 로드](https://docs.aws.amazon.com/ko_kr/neptune/latest/userguide/bulk-load.html)를 통해 데이터를 삽입할 수 있다. 

진행했던 프로젝트에서는 기존 그래프 데이터베이스가 따로 없고 관계형 데이터베이스에서 필요한 데이터만 추출하여 새로 그래프 데이터를 생성해야 했으므로, 쿼리를 직접 실행하여 데이터를 삽입했다. 콘솔창에서 하나씩 쿼리를 실행하는 방식으로는 RDB에 있는 모든 데이터를 다 삽입하기 어려워서 `Python`을 사용하여 `programmatically` 삽입하는 방식을 선택했다.<br>

`Python`으로 그래프 데이터베이스에 연결하여 노드/엣지 데이터를 넣었던 과정은 다음 포스트에서 다뤄보도록 하겠다.
<br>







