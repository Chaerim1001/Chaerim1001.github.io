---
title: private 리소스에 접근하는 방법 (AWS로 시작하는 인프라 구축의 정석)
date: 2023-07-24
categories: [Study, AWS로 시작하는 인프라 구축의 정석]

tags: [AWS]
---

> 본 포스트는 [AWS로 시작하는 인프라 구축의 정석](https://www.yes24.com/Product/Goods/109747932) 책을 활용한 스터디 기록입니다. <br> (CHAPTER 5 점프 서버 준비하기)
{: .prompt-info}

## 점프 서버
### (1) 점프 서버란?
점프 서버(Bastion Host)란 우리가 생성한 모든 리소스에 접근할 수 있는 입구를 의미합니다.

![](https://velog.velcdn.com/images/chaerim1001/post/b3b69882-b57d-42bf-a7c2-95a2bd93d605/image.png)

점프 서버는 **EC2**를 이용해 구축할 수 있습니다. 점프 서버는 리소스로의 통로 이외의 용도는 없기 때문에 성능이 낮아도 괜찮습니다!

### (2) 점프 서버가 필요한 이유
우리는 우리만의 가상 네트워크를 만들고 그 안에 여러 리소스들을 생성하여 사용하게 되는데요. 그런데 모든 리소스들을 public subnet에 두고 IP를 연결해 접근할 수 있게 만들면 안되겠죠.

그런데 그렇게 하면 private subnet에 생성한 리소스에는 어떻게 접근해야 할까요? ß이럴 때 우리는 점프 서버를 사용할 수 있습니다.

## Session Manager
> 아래 Session Manager에 대해 정리한 내용은 책에 포함되어 있지 않습니다.

![](https://velog.velcdn.com/images/chaerim1001/post/fead8f0b-e3be-4191-9397-18df4ea39ac9/image.png)

EC2 연결 탭에 들어가면 ssh를 사용한 접속 외에 **Session Manager** 라는 것이 존재합니다.
Session Mananger를 사용하면 점프 서버를 이용하는 것보다 조금 더 간결하게 아키텍처를 구성할 수 있습니다. 

### (1) Session Mananger의 장점

![](https://velog.velcdn.com/images/chaerim1001/post/9c79e613-d46e-4eb3-a8d2-118cd88063f2/image.png)

1. 비용 최적화
앞서 이야기한 것처럼 점프 서버 또한 EC2를 이용하며, 이는 당연히 비용이 발생할 수 있는 부분입니다! Session Manager를 사용하면 점프 서버 구축 비용을 절감할 수 있습니다.

2. 운영 효율
점프 서버 또한 우리가 직접 생성하여 사용하는 것이기 때문에 하나의 관리 포인트가 될 수 있습니다. Session Manager를 사용하면 점프 서버에 대한 운영관리가 사라집니다.

3. 보안 컴플라이언스 준수
Session Manager를 이용하면 Session에서 발생하는 로그를 모두 S3로 적재할 수 있기에 운영 중에 발생하는 이슈, 장애에 대한 트래킹에 매우 유용합니다.

### (2) Session Manager 사용하기
단순히 EC2 인스턴스를 생성한다고 해서 Session Manager를 바로 사용할 수 있는 것은 아닙니다.

Session Manager 사용을 위해서는 **AmazonSSMManagedInstanceCore** 이라는 IAM Policy가 적용된 Role을 생성하여 EC2에 적용시켜줘야 합니다!

<br>

> 참고 <br> 
https://www.lgcns.com/blog/cns-tech/aws-ambassador/40899/ <br>
https://catalog.workshops.aws/general-immersionday/ko-KR/basic-modules/10-ec2/ec2-linux/3-ec2-1