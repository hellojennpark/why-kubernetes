---
description: etcd 클러스터 개념 이해
---

# kube-apiserver 와 etcd 통신 구조

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

쿠버네티스에서 `kubectl`이나 컨트롤러가 요청을 보내면, kube-apiserver가 이를 받아 처리한 뒤 etcd에 저장하게 됩니다.\
이 때 kube-apiserver는 etcd v3 클라이언트 라이브러리를 사용하며, 이 라이브러리가 gRPC 클라이언트 Stub을 통해 etcd 멤버와 통신합니다.&#x20;

이 때 `kube-apiserver`는 etcd v3 클라이언트 라이브러리를 사용하며, 이 라이브러리가 gRPC 클라이언트 Stub을 통해 `etcd` 멤버와 통신합니다.

{% hint style="info" %}
**Stub란?**

gRPC는 IDL(proto 파일)로 서비스 정의를 하고, protoc가 각 언어별로 코드를 자동으로 생성해줍니다. 이때 클라이언트 쪽에 생기는 코드를 Stub(혹은 Proxy)라고 부릅니다.

함수 모양은 로컬 호출처럼 보이지만, 내부적으로는 gRPC 채널을 통해 원격 서버로 직렬화/네트워크 전송을 하게 됩니다.
{% endhint %}

etcd 클라이언트는 여러 멤버 엔드포인트를 설정하면 내부적으로 라운드로빈 방식으로 연결을 시도하고, 장애가 발생하면 자동으로 다른 멤버로 failover 합니다.\
쓰기 요청은 최종적으로 Raft 리더로 라우팅되지만, 클라이언트 입장에서는 'etcd 클러스터' 전체를 대상으로 안정적으로 통신하는 것처럼 보이게 됩니다.

### etcd cluster

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

etcd 클러스터는 여러 개의 etcd 멤버로 구성된 분산 Key-Value 저장소의 집합을 말합니다.

* etcd cluster: 여러 멤버로 구성되며, Raft 합의로 일관성 유지
* etcd 멤버: 클라이언트 요청은 **gRPC 서버 엔드포인트**에서 받고, 멤버 간 동기화는 **Raft peer 엔드포인트**로 처리

#### etcd 클러스터의 멤버의 역할

etcd 클러스터의 각 멤버들은 두 가지 역할을 수행하게 됩니다.

* **gRPC 서버(clientURL, 보통 2379포트)**: 클라이언트(kube-apiserver 등)의 요청을 받음
* **Raft 피어(peerURL, 보통 2380포트)**: 다른 멤버와 합의(리더 선출, 로그 복제)를 수행

클라이언트는 특정 멤버 하나가 아니라, 클러스터 전체를 대상으로 읽기/쓰기를 하는 개념이며, 논리적으로는 하나의 일관된 데이터 저장소로 취급합니다.

{% hint style="info" %}
**읽기/쓰기 요청 처리**

* **읽기 요청 (Read):** `kube-apiserver`가 라운드 로빈으로 요청을 보낸 멤버가 리더든 팔로워든 상관없이, 해당 멤버가 바로 응답합니다. (단, etcd 설정에 따라 `linearizable` Read는 리더 확인이 필요할 수 있습니다.)
* **쓰기 요청 (Write):** 라운드 로빈으로 요청을 보낸 멤버가 팔로워인 경우, 요청은 내부적으로 리더에게 전달되어 Raft 합의 과정을 거쳐 처리됩니다. 즉, `kube-apiserver`는 멤버의 역할을 신경 쓰지 않고 요청을 분산합니다.
{% endhint %}

#### kube-apiserver 의 통신 방식

`kube-apiserver`는 `etcd` 클러스터와의 통신을 위해 별도의 로드 밸런서를 두지 않습니다. 대신 etcd 클라이언트 라이브러리 자체의 내장 기능을 사용하여 통신을 관리합니다.

{% code overflow="wrap" %}
```yaml
etcd-servers: https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379
```
{% endcode %}

`etcd-servers` 플래그에 나열된 모든 멤버에게 지속적인 TCP 연결을 유지합니다. 요청 시 Round Robin 방식으로 멤버를 순회하며 요청을 보냅니다. 특정 멤버가 실패하면 연결을 끊고 다음 멤버에게 요청을 재시도합니다.

* **로드 밸런싱:** 모든 요청 (읽기/쓰기)은 연결된 `etcd` 멤버들에게 라운드 로빈(Round Robin) 방식으로 골고루 분산되어 전송됩니다. 이는 클라이언트 측 로드 밸런싱의 한 형태입니다.
* **읽기 요청 처리:** 요청을 받은 `etcd` 멤버 (리더 또는 팔로워)가 바로 응답하여 부하를 분산시킵니다.
* **쓰기 요청 처리:** 요청이 팔로워에게 가더라도, `etcd` 내부적으로 해당 요청은 자동으로 현재 Raft 리더에게 전달되어 합의 과정을 거쳐 처리됩니다.
* **페일오버:** 멤버 중 하나가 다운되면, `kube-apiserver`는 연결을 끊고 자동으로 남은 멤버들에게 요청을 전달하여 클러스터의 고가용성을 유지합니다.
