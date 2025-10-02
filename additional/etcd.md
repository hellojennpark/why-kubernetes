---
description: etcd 에 대해 조금 더 깊게 알아봅시다.
---

# etcd

## kube-apiserver 와 etcd 통신 구조

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

쿠버네티스에서 `kubectl`이나 컨트롤러가 요청을 보내면, kube-apiserver가 이를 받아 처리한 뒤 etcd 에 저장하게 됩니다. 이 때에 kube-apiserver는 gRPC 클라이언트 stub을 이용해 etcd 멤버 중 하나의 gRPC 서버 엔드포인트에 연결합니다.&#x20;

{% hint style="info" %}
**Stub란?**

gRPC는 IDL(proto 파일)로 서비스 정의를 하고, protoc가 각 언어별로 코드를 자동으로 생성해줍니다. 이때 클라이언트 쪽에 생기는 코드를 Stub(혹은 Proxy)라고 부릅니다.

함수 모양은 로컬 호출처럼 보이지만, 내부적으로는 gRPC 채널을 통해 원격 서버로 직렬화/네트워크 전송을 하게 됩니다.
{% endhint %}



### etcd cluster

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

etcd 클러스터는 여러 개의 etcd 멤버로 구성된 분산 Key-Value 저장소의 집합을 말합니다.

* etcd cluster: 여러 멤버로 구성되며, Raft 합의로 일관성 유지
* etcd 멤버: 클라이언트 요청은 **gRPC 서버 엔드포인트**에서 받고, 멤버 간 동기화는 **Raft peer 엔드포인트**로 처리

#### etcd 클러스터의 멤버의 역할

etcd 클러스터의 각 멤버들은 두 가지 역할을 수행하게 됩니다.

* **gRPC 서버(clientURL, 보통 2379포트)**: 클라이언트(kube-apiserver 등)의 요청을 받음
* **Raft 피어(peerURL, 보통 2380포트)**: 다른 멤버와 합의(리더 선출, 로그 복제)를 수행

즉, kube-apiserver는 etcd 클러스터라고 부르는 논리적 집합을 대상으로 하되, 실제 연결은 멤버들의 clientURL 리스트 중 하나에 직접 하게 됩니다.

클러스터는 Raft 합의 알고리즘으로 동작하여, 멤버 중 한 명이 리더가 되고 나머지는 팔로워로서 로그를 복제합니다.

#### etcd 클러스터 라고 부르는 이유

그렇다면 왜 그냥 '**etcd**'가 아니라 '**etcd 클러스터**'라고 부르는 걸까요?

클라이언트(예. kube-apiserver) 입장에서는 특정 멤버 하나가 아니라, **클러스터 전체**를 대상으로 읽기/쓰기를 하는 개념이기 때문입니다.&#x20;

실제로는 멤버 개별 주소(client URL)에 붙지만, 논리적으로는 하나의 일관된 데이터 저장소(etcd cluster) 취급합니다.

#### kube-apiserver 입장에서는?

{% code overflow="wrap" %}
```yaml
etcd-servers: https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379
```
{% endcode %}

`--etcd-servers` 플래그에 멤버 clientURL들을 쉼표로 나열하고, apiserver는 리스트 중 접속 가능한 멤버를 찾아 연결하게 됩니다.&#x20;

특정 멤버가 죽어도 다른 멤버로 자동으로 페일오버 됩니다. 즉, 엔드포인트를 “클러스터”라고 부르는 건 이런 멀티 엔드포인트 묶음을 의미하는 거고, 실제로는 멤버 중 하나에 붙게 됩니다.



{% hint style="info" %}
**로드밸런서가 있나요?**

기본적으로는 없습니다. etcd 클러스터는 Raft 기반이기 때문에 '리더/팔로워' 관계가 있고, 클라이언트는 어느 멤버로 붙어도 됩니다. 팔로워에 붙었더라도 쓰기 요청은 결국 Raft 리더로 라우팅 됩니다.
{% endhint %}

