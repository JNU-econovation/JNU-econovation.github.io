---
layout: post
title: AWS Lambda 와 S3를 활용한 이미지 업로드
subtitle: AWS Lambda 와 S3를 활용한 이미지 업로드 
author: yunseong
categories: TECH
banner:
  image: "https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/094c9d08-fdea-4d20-8b17-75852485f5c0"
tags: TECH
sidebar: []
---

# AWS Lambda 와 S3를 활용한 이미지 업로드

안녕하세요. 웹 프론트엔드 개발자 이윤성입니다. 

오늘은 프로젝트 과정에서 AWS Lambda와 S3를 사용하여 이미지 업로드/조회를 구현한 내용을 공유하려 합니다.

오늘 글을 통해 AWS를 사용하는 팀에서 쉽게 이미지 업로드/조회기능을 구현하셨으면 합니다.

# 개요

유저의 이미지 업로드와 조회기능을 구현하기 위해 여러가지 방법이 있는데요, 오늘은 AWS Lambda, S3, 그리고 Cloudfront를 활용해서 이미지 업로드/조회를 구현한 방법에 알아보겠습니다. 아래는 오늘 다뤄볼 간단한 구성도입니다.

<img src= https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/094c9d08-fdea-4d20-8b17-75852485f5c0/>

두가지 흐름으로 나눌 수 있는데요, 업로드와 조회로 구분해서 살펴보겠습니다.

## 업로드

먼저, AWS Lambda는 S3에 접근해 Presigned URL을 발급하는 역할을 수행합니다. 이 점을 유의하고 단계적으로 살펴보겠습니다.

1. 사용자는 S3에 업로드하기 위한 presigned URL을 요청합니다. 이 때, 람다 함수를 trigger하기 위해 API Endpoint를 Trigger 조건으로 구성합니다. 즉, 사용자는 presigned URL을 발급받으려면 람다함수의 Trigger 조건인 API Endpoint를 호출해야 합니다.
2. 람다 함수가 실행된다면, 람다는 S3에 접근해 presigned URL을 발급받습니다. 그리고, 반환 받은 presigned URL을 응답 값으로 사용자에게 넘겨줍니다.

## 조회

조회는 간단합니다. CloudFront URL을 통해 S3에 있는 객체들에 접근이 가능합니다. 이 때 사용자는 별도의 절차 필요 없이, 이미지를 조회할 수 있습니다.

## Lambda 선택 이유

S3에 업로드를 하기 위해 여러 가지 방법들이 있었지만, Lambda를 선택한 이유는 다음과 같습니다.

1. **저렴합니다.**

    lambda의 비용은 요청 1백만건당 0.2USD로, 굉장히 저렴합니다. 

   <img width="874" alt="lambda선택이유1" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/b64d300a-8880-4dcd-a756-27c18dee38c4">

2. **S3에 보다 손쉽게 접근할 수 있습니다.** 

    Lambda를 사용한다면 다른 AWS 서비스에 접근이 용이합니다. 특히 제가 해결한 방법으로는 S3에 접근할 때 클라이언트 코드에 인가 정보를 주는 것이 아닌 람다에서 접근하고 엔드포인트만 노출하므로 보안적으로도 비교적 좋다고 생각합니다. 
3. **서비스의 백엔드 서버에 의존도를 줄여 줍니다.**

    서비스의 백엔드 서버에 이미지 업로드 로직이 있다면 이미지가 서비스 백엔드 서버를 한 번 거치는데요, 이는 현재 단계(사용자가 별로 없는 단계)에서는 별 의미가 없지만 부하가 많아진다면 서비스에 지장이 갈 수 있다고 생각이 들었습니다. 따라서 이미지 업로드를 처리하는 로직을 람다로 분리해 관리하기로 결정했습니다.

## CloudFront 선택이유

1. **캐싱의 장점을 용이하게 살릴 수 있습니다.**

    이미지를 고유한 이름으로 저장한다면, 캐싱이 굉장히 유용한 전략이라고 생각합니다. 시간이 지나도 데이터가 변할리 없는 데이터이기 때문입니다. (만약 글을 수정해서 이미지가 바뀐다면 고유한 이미지 자체가 바뀌겠죠.)

    따라서, S3에 직접 접근해 처리하는 방식보다 AWS 엣지 로케이션에서 캐싱하는 방식으로 콘텐츠 전송을 보다 더 빠르게 하기 위해 구성하였습니다.

    <img width="653" alt="클라우드프론트선택이유" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/43c6107f-8826-4348-8616-c99dffff2e3e">

    Response Header에서 CloudFront를 통해 캐싱이 되고 있음을 알 수 있습니다.


2. **압축 지원**

    콘텐츠에 대한 압축을 지원합니다. 용량이 큰 파일은 압축을 해 보냄으로써, 사용자는 보다 더 빠른 이미지를 받을 수 있습니다.

<aside>
⚠️ <b> 은 탄환(Sliver Bullet)은 없습니다. </b> <br>
콘텐츠에 대한 압축은 사용자의 CPU에 부담을 줍니다. 압축의 목적은 네트워크 상으로 전송하는 절대적인 바이트를 줄여서 사용자에게 보다 더 빠르게 제공하려는 것에 목적이 있습니다. <br> 
만약 사용자 환경이 CPU 성능이 그렇게 좋지 못한 디바이스라면, 압축을 검토해보는 것이 좋습니다. 네트워크 속도보다 CPU가 압축된 콘텐츠를 Decoding하는 시간이 더 걸릴수도 있습니다. <br>
</aside>

# 1. S3 구성

버킷은 사용자가 이미지를 업로드(PUT) presigned URL을 발급해주어야 합니다. 따라서 버킷을 생성하고, 그에 맞는 설정을 해보겠습니다. 여기서 가장 중요한 건 S3에 접근하는 AWS 서비스 혹은 특정 서비스에게 어떻게 권한을 주어야 하는가를 주의깊게 살펴보아야 합니다. 따라하시면서 계속 나오겠지만, 아마 에러가 발생한다면 이 권한에서 그럴 확률이 상당히 높습니다.

## 1.1 s3 생성

먼저, S3를 구성하겠습니다. 

<img width="866" alt="s3생성" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/4f20773f-f3e0-4898-9c4f-afec33f56974">

주의할 점은 퍼블릭 액세스 차단을 해제 해주세요. 여기서는 후술할 **버킷 정책** 을 통해 액세스를 제어할 예정입니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/7daabdb7-c6e2-45af-89ca-81d4d3f08279/>

## 1.2 CORS 설정

생성하였다면, 버킷의 권한에 들어가 CORS 설정을 해주어야 합니다. 실제 우리 서비스의 Origin에서 람다로 보내는 요청은 CORS 이기 때문입니다.

따라서 이곳에서는 CORS(교차 출처 리소스 공유)에 대한 지식이 필요합니다.

<img width="1529" alt="cors설정" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/e92ec5d1-6353-4f57-8d47-01fe88d435b7">

### 예시 코드

```jsx
[
    {
        "AllowedHeaders": [ // 허용할 헤더
            "*"
        ],
        "AllowedMethods": [ // 허용할 메서드. 여기서는 PUT을 작성
            "PUT"
        ],
        "AllowedOrigins": [ // 허용할 Origin
            "http://localhost:5173",
            "http://localhost:3000",
            "http://your-service-address.com"
        ],
        "ExposeHeaders": [] // CORS 상에서 기본 헤더를 제외하고 노출시킬 헤더
    }
]
```

- **AllowedMethod**
    - 허용할 메서드를 작성합니다. 이미지 업로드 로직이므로 PUT을 허용하였습니다. 잠시 후 람다 함수 설명 구간을 참고하세요.
- **AllowedOrigins**
    - 허용할 Origin들을 작성합니다. 팀에서 [localhost](http://localhost) 에서 테스트 하기위해 localhost의 주소를 넣어주었습니다.

## 1.3 버킷 정책 생성

S3에 대해 다른 서비스들이 어떤 작업을 할수 있는지 설정해주는 정책 구간입니다. 정책 생성기를 이용하면 편리합니다. 간단히 설명하고 넘어가겠습니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/d50b826f-e58a-4701-9c60-117cc2aa0893/>

```jsx
{
  "Id": "Policy1718086112979",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1718086111552",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::econovation-test-bucket/*",
      "Principal": "*"
    }
  ]
}
```

중요한 부분만 살펴보겠습니다.

Principal: 이 정책이 적용되는 주체를 의미합니다. 여기서는 제 서비스의 모든 AWS주체를 의미합니다.

**Action:** 허용할 액션을 정의합니다. CloudFront에서는 GetObject를, Lambda에서는 PutObject권한이 필요하니 각각 설정해주었습니다.

Resource: 정책이 적용되는 리소스를 의미합니다. 여기서든 모든 리소스에 접근합니다.

# 2. CloudFront 구성

먼저 쉬운 Cloudfront를 구성해보겠습니다. 그러기 전에 CDN에 대해 간단히 알아보겠습니다. 

## CDN이란?

CDN은 Content Delivery Network의 약자로, 콘텐츠를 효율적으로 전달하기 위해 여러 노드(지점)를 가진 네트워크에 데이터를 저장해 제공하는 방식입니다.

쉽게 말해 여러 캐시 서버를 물리적으로 가깝게(혹은 짧은 홉의) 여러 개를 두어 사용자로 하여금 보다 빠르고 효율적으로 콘텐츠를 제공받게 도와주는 것이라고 생각하면 되겠습니다.

여기서는 S3라는 이미지 원본 저장소의 콘텐츠를 AWS의 각 노드(지점)마다 있는 CDN에 복사해 가지고 있다가 콘텐츠를 필요로 하는 사용자에게 제공하는 역할이라고 보시면 되겠습니다.

## 2.1 cloudfront 생성

먼저, CloudFront를 생성합니다. 하나씩 차근차근 살펴보겠습니다. 먼저, 여기서 말하는 원본은 콘텐츠를 제공하는 “원본 저장소”를 의미합니다. CloudFront는 “원본 서버”에서 각 콘텐츠를 복사해온다는 점을 잊지 마세요.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/f0df1830-833d-44b6-addf-7564e42e05e2/>

**Origin domain**

- 원본을 지정합니다. 여기서는 S3입니다.

**Origin Path**

- 여기서는 생략합니다. root에서 바로 접근할 수 있도록 간단하게 구성할 것이기 때문입니다.

**이름**

- 원본의 이름을 입력합니다. 원하는 이름으로 구성하시면 됩니다.

**원본 액세스**

- 원본의 객체를 CloudFront를 통해서만 조회할 수 있도록 구성합니다.

### 기본 캐시 동작

앞에서 CloudFront는 원본 저장소에서 콘텐츠를 “복사”해서 가지고 있는다고 하였습니다. 이를 “캐시”라고 합니다. (엄밀한 정의는 아닙니다.) 따라서 “복사한 콘텐츠를 얼마나 가지고 있어야 하나?” 에 대한 설정이라고 보시면 되겠습니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/6f8e1a03-d93f-4eed-8508-11372f8f6e1b/>

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/c4075cc4-2d47-4881-ac6f-539fa0fa9c35/>

**객체 압축**

- 앞서 말했듯 Gzip, brotli를 통해 객체를 압축해 전송할 수 있습니다. 보내는 콘텐츠의 용량을 줄여서 보낼 수 있게 지원합니다.

**허용된 HTTP 방법**

- CloudFront를 통해 조회만 하므로 GET, HEAD로만 지정해줍니다.

**캐시 정책**

- HTTP 캐싱에 관련된 TTL, 캐시 키등을 설정합니다. 중점은 “얼마나 캐시 설정을 할 것이냐?” 라고 보시면 됩니다.

**가격 분류**

서비스 지역을 한국만 잡고 계시다면, 아래 설정을 권장합니다. 가격 차이가 어느정도 있으니까요.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/749a7c23-d5e2-4cb1-b1d3-6c51a26f7bf5/>

나머지 설정은 프로젝트 내 요구사항에 맞게 설정하시면 됩니다. 구체적인 설정들은 생략하도록 하겠습니다.

## 2.2 배포확인

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/6d889fc5-13c2-4183-81b4-5efacf15fe2e/>

배포 확인을 하면 배포 도메인을 확인할 수 있습니다. S3에 이미지를 수동으로 업로드 해보고, 아래 주소로 확인해보세요.

`https://<배포된 CloudFront 도메인 이름>/<S3에 업로드한 파일명(확장자 포함)>`

`ex) [https://dmxnsdl234.cloudfront.net/myimage.png](https://dmxnsdl234.cloudfront.net/myimage.png)`

# 3. AWS Lambda

이제 이미지 조회까지 완료했습니다! 이제 이미지 업로드를 구현해보겠습니다. 앞서 말했듯 Lambda를 사용합니다. Lambda는 쉽게 말해서 한 줄로 정리가 가능합니다.

**“사용자로부터 요청이 들어오면, S3에 이미지 업로드를 할 수 있는 URL을 발급받아, 사용자에게 응답으로 알려주는 초간단 서버”**

지금부터 한 번 만들어 볼까요?

## 3.1 람다 함수 생성

다음과 같이 구성해주세요.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/cbcbf899-1939-49b7-b463-29db0ec4735c/>

**함수 이름**

- 편하게 작성해주세요.

**런타임**

- putObject를 수행할 런타임을 골라주세요. 자기가 구현할 언어로 선택하시면됩니다. 저는 설명을 위해 범용적인 Python을 채택했습니다.

**아키텍처**

- 현재 예제에서는 아키텍처는 큰 구애를 받지 않습니다. 우리가 구현하려는 기능과, 파이썬 자체가 명령어 집합에 구애받는 언어는 아니기 때문이죠. 성능까지 고려하기에는 그 가치가 미미하기 때문에, 저렴한 arm64를 선택해봅시다.

> 사설: arm64가 저렴한 이유는 뭘까요? 
*사설은 제 개인적인 생각입니다.*
이는 아키텍처 설계와 관련있어 보입니다. arm 기반 아키텍처는 상대적으로 성능에 최적화 하기보다 전력 소비와 열 발생 등을 줄이도록 설계했기 때문이라고 생각합니다.
> 

여기까지 왔다면 성공입니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/6135f66c-0acb-4bf0-9219-32ec280e7edd/>

## 3.2 업로드 기능 구현

업로드 기능은 AWS SDK 에 접근할 수 있는 라이브러리인 boto3 를 사용합니다. boto3에 대한 설명은 따로 적지 않겠습니다.

lambda_function에 아래와 같이 작성해주세요. 코드에 주석으로 설명이 첨부되어 있습니다!

제가 이렇게 작성했지만, 저보다 더 서버로직을 잘 작성하시는 여러분은 본인 언어에 맞게 구현하시면 됩니다 :)

코드를 읽어보며 사용자가 어떤 요청을 보내야 하는지, 어떤 응답을 받아야하는지 주의하며 읽어주세요.

```python
import json
import boto3
import uuid

# Configure S3 client 
s3 = boto3.client('s3')

def lambda_handler(event, context):
 
 
 # 파일 확장자를 알기 위해 커스텀 HTTP를 추가하였습니다.
 fileExtension = event.get('headers').get('x-extension');
 
 # 모니터링의 로그에 남습니다. 생략하셔도 됩니다.
 print(event);
 print(fileExtension);
 
 # S3 버킷명을 작성해주세요.
 bucket_name = 'econovation-test-bucket'
 
 # 파일명을 유니크한 이름으로 짓기 위해 uuid를 사용합니다.
 key = f'{uuid.uuid4()}.{fileExtension}' 

 # S3로부터 pre-signed URL을 발급 받는 과정입니다.
 
 presigned_url = s3.generate_presigned_url(
 ClientMethod='put_object', # 여기서 putObject를 사용합니다. 아마 안된다면 정책 설정에 문제가 있을 확률이 높습니다.
 Params={
   'Bucket': bucket_name, 
   'Key': key,
   'ContentType': f'application/{fileExtension}', # Set the Content-Type for MIME type
 },
 
 ExpiresIn=3600, # URL 만료 시간을 설정합니다. 저는 1시간으로 설정했습니다!
 HttpMethod='PUT'
 );
 

 # 클라이언트에게 보내주는 응답 값 입니다.
 # 예제에서는 에러 처리에 대한 내용은 다루지 않겠습니다 :(
 response = {
   'statusCode': 200,
   'headers': {
   'Access-Control-Allow-Origin': '*'
   },
   'body': json.dumps({
   'presignedUrl': presigned_url,
   'key': key
   })
 }

 return response
```

다 작성하셨나요? 그렇다면 “Deploy”를 눌러주세요. 그렇다면 정상적으로 업로드 되었음을 확인할 수 있습니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/f86dd668-3b2c-4fc8-9c38-45f9eec06cbf/>

<aside>
💡 Test를 통해 실제 사용자가 요청을 보내는 것처럼 테스트를 해볼 수 있습니다! 관심있으신 분들은 AWS의 [Testing Lambda function](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/testing-functions.html) 을 참고해보세요!

</aside>

### 실행 역할 설정

구성 → 실행역할→ 역할이름을 클릭합니다. (역할 이름이 없다면 하나 생성하시기 바랍니다.)

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/8be39df9-a1b9-4ae8-877e-a2b858236fe2/>

권한추가 → 인라인 정책 생성을 클릭합니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/6c4bff10-0b58-4ae2-9b18-e16e858b6630/>

### 권한 지정
S3와 PutObject를 선택하고 다음을 누릅니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/dd6b8861-e127-4bf7-a050-9668da37bb4b />

알맞게 정책이름을 작성하고 생성합니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/0ee5acdb-2a00-4a2f-b8fc-8789c13f71d3 />

# 4. API Gateway Trigger

클라이언트에게 람다 함수를 호출하기 위한 API를 제공해야합니다. 이를 API Gateway를 통해 람다함수를 Trigger할 수 있는데요,

람다함수 → 구성 → 트리거 혹은 다이어그램의 트리거 추가 버튼을 누릅니다.

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/5c7ca720-4fe3-4889-ad50-852fec1ae787 />

다음과 같이 설정해시면됩니다. 이 부분은 쉽기 때문에 따로 설명은 생략하도록 하겠습니다. 도메인이 다르다면 당연하게도 CORS 설정을 (예)로 해주시길 바랍니다!

<img src=https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/c15cf243-be27-4bb7-8502-c48983de1287/>

그렇다면 이렇게 api 엔드포인트가 나오게 됩니다. 이제 프론트엔드에서 업로드 서비스를 구현해볼까요?

## CORS 설정

아까 lambda의 코드에서 custom 헤더가 있었음을 기억하시나요? 확장자를 파악하기 위해서 있는 건데요, 그래서 CORS상에서도 이 헤더를 추가해주어야 합니다.

API Gateway → 좌측 CORS에 들어가 허용할 Header를 삽입해주세요.

<img width="1529" alt="cors설정" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/944da943-5835-44e8-ad0c-b9edbebec2fe">

# 5. 프론트엔드 구현

이 문서에서는 React + Vite로 간단히 시연해보겠습니다.

<aside>
💡 이 문서에서는 유지보수를 위한 코드, 안전한 코드보다 이해를 돕기위해 절차적으로 작성해보겠습니다. 수없는 매직넘버들이 난무할 것이며…(_ _) 실제 프로덕션에 담는 코드는 이 코드 그대로 작성하기 보다 개선해보시기 바랍니다. 특히 도메인 자체를 깃 호스팅 사이트에 올리는 건 위험할 수 있습니다.
따라서.. 어떻게 돌아가는지에 집중해서 읽어주시기 바랍니다.

</aside>

## presigned URL 발급 및 업로드 로직

```tsx
// api.ts
import axios, { RawAxiosRequestHeaders } from 'axios';

export const imgUploadAxiosInstance = axios.create({
  baseURL: 'https://your-lambda-execute-url.com',
});

// MIME Type을 지정해줍니다.
const generateHeaderForPresignedUrl = (
  key: keyof RawAxiosRequestHeaders,
  extension: string
): RawAxiosRequestHeaders => {
  return {
    [key]: `application/${extension}`,
  };
};

// lambda를 호출해 presigned URL을 발급받습니다.
export const getPresignedUrl = async (extension: string) => {
  const response = await imgUploadAxiosInstance.get(
    `/your-lambdda-trigger-endpoint`,
    {
      headers: {
        ...generateHeaderForPresignedUrl('Content-Type', extension),
        'x-extension': extension,
      },
    }
  );
  return response.data as { presignedUrl: string };
};

// 발급받은 Presigned URL을 통해 PutObject를 수행합니다.
export const putImageToS3 = async (
  presignedUrl: string,
  file: File,
  extension: string
) => {
  const instance = axios.create();

  return await instance.put(presignedUrl, file, {
    headers: generateHeaderForPresignedUrl('Content-Type', extension),
  });
};

```

### hooks

```tsx
// useImageUpload.ts
import { ChangeEvent, Dispatch, SetStateAction, useState } from 'react';
import { getPresignedUrl, putImageToS3 } from './api';

// 파일의 확장자를 가져옵니다. (매우 러프하게...)
const getFileExtension = (file: File) => {
  const { name } = file;
  return name.split('.')[1];
};

// 업로드 하는 비동기 로직을 구현합니다.)
const handleUpload = async (
  file: File,
  setFileNameList: Dispatch<SetStateAction<string[]>>
) => {
  // 1. extension을 찾아옵니다.
  const extension = getFileExtension(file);
  // 2. presigned URL을 발급받습니다.
  const presignedBody = await getPresignedUrl(extension);
  try {
    // 3. 받아온 presigned URL을 통해 S3에 업로드합니다.
    await putImageToS3(presignedBody.presignedUrl, file, extension);
    const url = new URL(presignedBody.presignedUrl);
    // 업로드하고 파일명으로 이미지 조회가 되는지 확인하기 위한 로직
    const fileName = url.pathname.slice(1);
    setFileNameList((prev) => [...prev, fileName]);
  } catch (e) {
    alert('이미지 업로드에 실패하였습니다. 잠시 후 다시 시도 해보세요.');
  }
};

const useImageUpload = () => {
  const [fileNameList, setFileNameList] = useState<string[]>([]);

  const onUploadFile = (event: ChangeEvent<HTMLInputElement>) => {
    if (!event.target.files) return;
    const file = event.target.files[0];
    handleUpload(file, setFileNameList);
  };

  return { onUploadFile, fileNameList };
};

export default useImageUpload;

```

### component

```tsx
import useImageUpload from './useImageUpload';

const CLOUDFRONT_URL = 'https://your.cloudfront.net';
function App() {
  const { onUploadFile, fileNameList } = useImageUpload();

  return (
    <>
      <div>
        <input
          onChange={onUploadFile}
          type="file"
          accept="image/png, image/jpeg, image/jpg"
        />

        <div>
          { /* CloudFront URL과 fileName의 조합으로 찾아옵니다. */}
          {fileNameList.map((fileName) => (
            <img key={fileName} src={`${CLOUDFRONT_URL}/${fileName}`} alt="" />
          ))}
        </div>
      </div>
    </>
  );
}

export default App;

```

완성입니다! 실제로 이미지가 잘 올라오고, 또 CloudFront를 통해 잘 조회되는지 살펴보세요.

람다호출, 이미지 업로드 잘 되는 걸 확인할 수 있네요 😄

<img width="1349" alt="에코노" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/03e0c5af-6483-4ea6-ad18-fefba6186528">


이미지 조회 또한 잘 되는 것을 확인할 수 있습니다 😎

<img width="831" alt="이미지조회" src="https://github.com/JNU-econovation/JNU-econovation.github.io/assets/125952146/7e840f37-f96e-406b-a4b2-062b3705b80c">

# 결론

오늘은 이렇게 Lamdba, S3, CloudFront를 활용해 이미지 업로드/조회 하는 기능을 구현해보았습니다. 이 글을 통해 여러분들 프로젝트에 도움이 되셨기를 바라며, AWS를 더 잘 다루시기 바랍니다!

글에 대한 오류나 오타가 있다면 언제든지 연락주세요~!

## 참고문서

[https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

https://medium.com/@brianhulela/upload-files-to-aws-s3-bucket-from-react-using-pre-signed-urls-543cca728ab8

[https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.htm](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html)

https://redhat.com/ko/topics/linux/what-is-arm-processor

- 아마존 서비스 공식문서
- boto3 document
