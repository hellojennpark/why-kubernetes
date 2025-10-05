---
description: Control Plane의 구성요소인 kube-apiserver에 대해 알아보겠습니다.
---

# kube-apiserver

**kube-apiserver**는 쿠버네티스 클러스터의 **컨트롤 플레인(Control Plane)** 구성 요소 중 하나입니다. 컨트롤 플레인은 클러스터의 전반적인 상태를 결정하고 관리하는 두뇌 역할을 하며, kube-apiserver, etcd, kube-scheduler, kube-controller-manager 등으로 구성됩니다.

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

### kube-apiserver의 역할

모든 HTTP/gRPC 요청이 apiserver를 통하기 때문에 kube-apiserver는 **클러스터의 단일 진입점(Front Door/Front-end)**&#xB77C;고 불립니다. 즉, **RESTful API**를 통해 쿠버네티스 클러스터에 대한 모든 명령 및 통신을 노출하고 처리하는 역할을 한다는 것인데요. 대표적인 역할 네가지를 하나씩 살펴보겠습니다.

* **모든 요청의 단일 진입점:** 사용자(kubectl), 클러스터 내부 컴포넌트(Controller, Scheduler, Kubelet), 외부 시스템 등 **클러스터의 상태를 조회하거나 변경하려는 모든 주체**는 반드시 kube-apiserver를 통해서만 접근해야 합니다.
  * 요청 흐름 제어 (APF, inflight limit, timeout 등)
  * 프록시/터널 (exec/logs/port-forward 등)
* **인증 및 인가(Authentication & Authorization):** 모든 요청에 대해 유효한 사용자인지(인증)와 요청된 작업을 수행할 권한이 있는지(인가)를 검증하여 클러스터의 **보안 게이트** 역할을 합니다.
  * 인증(Authentication), 인가(Authorization) 외에도 어드미션(Admission: Mutating/Validating), 감사(Audit)

{% hint style="info" %}
아래 단계에서 실패하는 경우 보안 문제로 요청 자체가 클러스터에 접근 할 자격 자체가 없다는 의미입니다.

* **인증(Authentication)**&#xC740; 요청을 보낸 주체가 누구인지 신원을 확인합니다.
  * ex. Service Account Token, X.509 인증서, Basic Auth
  * 대표적으로  `401 Unauthorized` 에러와 관련되어있는데 이 경우에는 요청자의 인증 정보(토큰, 인증서)가 유효한지 확인해야 합니다.
* **인가 (Authorization)**&#xB294; 인증된 주체가 요청한 **작업(Verb)**&#xC744; 해당 리소스에 대해 수행할 권한이 있는지 확인합니다. (RBAC 규칙 적용)
  * 대표적으로 `403 Forbidden` 에러 발생 시, 요청자의 RBAC Role 및 RoleBinding 규칙 확인해야 합니다.
  * RBAC는 **Role-Based Access Control (역할 기반 접근 제어)**&#xC758; 약자로, 쿠버네티스 클러스터 리소스에 접근하는 **주체(User, Group, ServiceAccount)**&#xC5D0;게 **역할(Role)**&#xC744; 부여하여 접근 권한을 통제하는 메커니즘입니다.

아래 단계에서 실패하는 경우 구성/정책 문제로 요청된 리소스의 정의(Spec)가 규칙이나 정책을 위반했다는 의미입니다.

* **뮤테이팅 어드미션 (Mutating Admission)**&#xC740; 요청을 변경(Mutate)하는 단계입니다. 기본값 설정, 사이드카 컨테이너 삽입 등 정책 적용됩니다.
  * 예상치 못한 파드의 Spec 변경이 발생한다면 `MutatingWebhookConfiguration`을 확인해야 합니다.
* **유효성 검사 (Validation)**&#xC5D0;서는 요청된 리소스 객체가 쿠버네티스의 스키마에 맞는지, 논리적으로 유효한지 검증합니다.
  * `422 Unprocessable Entity` 에러 발생 시, YAML 파일의 구조나 필드 값 확인해야 합니다.
* **유효성 검증 어드미션 (Validating Admission)**&#xC5D0;서는 요청의 유효성(Validate)을 검사합니다. 커스텀 정책이나 복잡한 비즈니스 로직 적용됩니다.
  * 커스텀 정책 위반 시 에러가 발생합니다.
  * `ValidatingWebhookConfiguration` 설정 및 외부 웹훅 서버 상태를 확인할 수 있습니다.
{% endhint %}

* **유효성 검사(Validation):** 요청된 API 객체(예: Pod, Deployment)의 정의(Spec)가 쿠버네티스 스키마에 맞는지 확인합니다.
  * **리소스 수명주기 관리**
  * 기본값 채우기(Defaulting), 스키마/필드 검증(Validation), 버전 변환(Conversion), 최종화자(Finalizer) 처리
* **상태 저장 중개자:**  클러스터의 **'의도된 상태(Desired State)'**&#xB97C; 저장하는 etcd 데이터베이스에 대한 유일한 직접 접근 경로를 제공합니다.
  * etcd Linearizable Write, Serializable Read, Watch Cache

{% hint style="info" %}
**객체 저장 (Object Persistence)** 단계에서는 모든 검증을 통과한 객체를 etcd에 최종적으로 저장합니다.

* 앞선 보안/검증 과정을 모두 통과한 요청이 `kube-apiserver`를 벗어나 **클러스터의 중앙 데이터베이스(`etcd`)에 실제로 기록되는 경계 지점**이기 때문에, 여기서 발생하는 문제는 `kube-apiserver`의 기능적 문제보다는 인프라/네트워크/etcd 자체의 안정성 문제를 시사합니다.
* API Server가 etcd와 통신 문제를 겪고 있는지, etcd 클러스터 상태 확인해야 합니다.
{% endhint %}

