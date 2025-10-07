---
description: Control Plane의 구성요소인 kube-apiserver에 대해 알아보겠습니다.
---

# kube-apiserver

**kube-apiserver**는 쿠버네티스 클러스터의 **컨트롤 플레인(Control Plane)** 구성 요소 중 하나입니다. 컨트롤 플레인은 클러스터의 전반적인 상태를 결정하고 관리하는 두뇌 역할을 하며, kube-apiserver, etcd, kube-scheduler, kube-controller-manager 등으로 구성됩니다.

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

## kube-apiserver의 역할

모든 HTTP/gRPC 요청이 apiserver를 통하기 때문에 kube-apiserver는 **클러스터의 단일 진입점(Front Door/Front-end)**&#xB77C;고 불립니다. 즉, **RESTful API**를 통해 쿠버네티스 클러스터에 대한 모든 명령 및 통신을 노출하고 처리하는 역할을 한다는 것인데요. 대표적인 역할 네가지를 하나씩 살펴보겠습니다.

* **모든 요청의 단일 진입점:** 사용자(kubectl), 클러스터 내부 컴포넌트(Controller, Scheduler, Kubelet), 외부 시스템 등 **클러스터의 상태를 조회하거나 변경하려는 모든 주체**는 반드시 kube-apiserver를 통해서만 접근해야 합니다.
  * 요청 흐름 제어 (APF, inflight limit, timeout 등)
  * 프록시/터널 (exec/logs/port-forward 등)
* **인증 및 인가(Authentication & Authorization):** 모든 요청에 대해 유효한 사용자인지(인증)와 요청된 작업을 수행할 권한이 있는지(인가)를 검증하여 클러스터의 **보안 게이트** 역할을 합니다.
  * 인증(Authentication), 인가(Authorization) 외에도 어드미션(Admission: Mutating/Validating), 감사(Audit)

{% hint style="info" %}
**트러블슈팅을 한다면?**

아래 단계에서 실패하는 경우 보안 문제로 요청 자체가 클러스터에 접근 할 자격 자체가 없다는 의미입니다.

* **인증(Authentication)**&#xC740; 요청을 보낸 주체가 누구인지 신원을 확인합니다.
  * ex. Service Account Token, X.509 인증서, Basic Auth
  * &#x20;`401 Unauthorized` 에러와 관련되어있는데 이 경우에는 요청자의 인증 정보(토큰, 인증서)가 유효한지 확인해야 합니다.
* **인가 (Authorization)**&#xB294; 인증된 주체가 요청한 **작업(Verb)**&#xC744; 해당 리소스에 대해 수행할 권한이 있는지 확인합니다. (RBAC 규칙 적용)
  * `403 Forbidden` 에러 발생 시, 요청자의 RBAC Role 및 RoleBinding 규칙 확인해야 합니다.
  * RBAC는 **Role-Based Access Control (역할 기반 접근 제어)**&#xC758; 약자로, 쿠버네티스 클러스터 리소스에 접근하는 **주체(User, Group, ServiceAccount)**&#xC5D0;게 **역할(Role)**&#xC744; 부여하여 접근 권한을 통제하는 메커니즘입니다.

아래 단계에서 실패하는 경우 구성/정책 문제로 요청된 리소스의 정의(Spec)가 규칙이나 정책을 위반했다는 의미입니다.

* **뮤테이팅 어드미션 (Mutating Admission)**&#xC740; 요청을 변경(Mutate)하는 단계입니다. 기본값 설정, 사이드카 컨테이너 삽입 등 정책 적용됩니다.
  * 예상치 못한 파드의 Spec 변경이 발생한다면 `MutatingWebhookConfiguration`을 확인해야 합니다.
* **유효성 검증 어드미션 (Validating Admission)**&#xC5D0;서는 요청의 유효성(Validate)을 검사합니다. 커스텀 정책이나 복잡한 비즈니스 로직 적용됩니다.
  * 커스텀 정책 위반 시 에러가 발생합니다.
  * `ValidatingWebhookConfiguration` 설정 및 외부 웹훅 서버 상태를 확인할 수 있습니다.
{% endhint %}

* **유효성 검사(Validation):** 요청된 API 객체(예: Pod, Deployment)의 정의(Spec)가 쿠버네티스 스키마에 맞는지 확인합니다.
  * **리소스 수명주기 관리**
  * 기본값 채우기(Defaulting), 스키마/필드 검증(Validation), 버전 변환(Conversion), 최종화자(Finalizer) 처리

{% hint style="info" %}
**트러블슈팅을 한다면?**

위의 경우와 마찬가지로 리소스의 정의(Spec)가 규칙이나 정책을 위반했다는 의미입니다.

**유효성 검사 (Validation)**&#xC5D0;서는 요청된 리소스 객체가 쿠버네티스의 스키마에 맞는지, 논리적으로 유효한지 검증합니다.

* `422 Unprocessable Entity` 에러 발생 시, YAML 파일의 구조나 필드 값 확인해야 합니다.
{% endhint %}

* **상태 저장 중개자:**  클러스터의 **'의도된 상태(Desired State)'**&#xB97C; 저장하는 etcd 데이터베이스에 대한 유일한 직접 접근 경로를 제공합니다.
  * etcd Linearizable Write, Serializable Read, Watch Cache

{% hint style="info" %}
**트러블슈팅을 한다면?**

이 경우에는 앞 선 검증 단계를 통과했기 때문에 API Server는 정상이지만, 백엔드 etcd와의 통신 또는 etcd 자체의 문제일 가능성이 높습니다.

**객체 저장 (Object Persistence)** 단계에서는 모든 검증을 통과한 객체를 etcd에 최종적으로 저장합니다.

* 앞선 보안/검증 과정을 모두 통과한 요청이 `kube-apiserver`를 벗어나 **클러스터의 중앙 데이터베이스(`etcd`)에 실제로 기록되는 경계 지점**이기 때문에, 여기서 발생하는 문제는 `kube-apiserver`의 기능적 문제보다는 인프라/네트워크/etcd 자체의 안정성 문제를 시사합니다.
* API Server가 etcd와 통신 문제를 겪고 있는지, etcd 클러스터 상태 확인해야 합니다.
{% endhint %}

### Flow Control(흐름 제어)

API Server는 클러스터의 핵심 게이트웨이로서, 갑작스러운 요청 폭주(Traffic Surge)로부터 스스로를 보호하고 중요한 트래픽을 보장하기 위한 흐름 제어(Flow Control) 메커니즘을 사용합니다. 이것이 바로 Inflight Limit과 APF입니다.

| 구분    | Inflight Limit (Legacy)   | APF (API Priority and Fairness) |
| ----- | ------------------------- | ------------------------------- |
| 제한 기준 | 오직 요청 종류 (읽기 vs. 쓰기)      | 우선순위 및 요청 주체                    |
| 부하 대응 | 즉시 거부 (모든 요청에 대해 공평하게 거부) | 우선순위에 따라 처리 후, 대기열에 넣음          |
| 운영 목표 | API Server 다운 방지 (전체 안정성) | 핵심 트래픽 보장 (서비스 연속성)             |

#### Inflight Limit (동시 처리 요청 제한)

Inflight Limit은 kube-apiserver가 특정 시점에 동시에 처리할 수 있는 최대 요청 수를 제한하는 가장 기본적인 보호 메커니즘입니다.

Inflight"는 "진행 중인"이라는 뜻으로, 클라이언트로부터 요청을 받아 **처리 중이지만 응답을 완료하지 않은 상태**의 요청 수를 의미합니다. 요청 폭주가 발생했을 때 API Server의 리소스(CPU, 메모리)가 고갈되어 서버 자체가 다운되는 것을 방지합니다. 이 제한을 초과하는 요청은 즉시 거부되며 클라이언트에게 **HTTP 429 (Too Many Requests)** 응답 코드를 반환합니다.

{% hint style="warning" %}
Inflight Limit은 요청을 오직 "읽기"와 "쓰기" 두 가지로만 구분했습니다. 이 때문에, **사소하거나 덜 중요한 요청**(예: 로깅, Pod 의 List 요청)이 제한을 모두 소진하면, 클러스터 운영에 필수적인 핵심 요청(e. Scheduler의 Pod 바인딩 요청)까지 함께 거부되는 문제가 발생했습니다.
{% endhint %}

#### APF (API Priority and Fairness)

APF (API Priority and Fairness)는 Inflight Limit의 한계를 극복하고 API Server의 안정성을 높이기 위해 쿠버네티스 $$1.20$$ 버전 이후부터 기본으로 활성화된 고급 흐름 제어 메커니즘입니다.

위에서 Inflight Limit은 사소하거나 덜 중요한 요청에 의해 핵심 요청이 밀려나는 경우가 있었다고 했는데요, APF는 Inflight Limit이 제공하는 총 동시 처리 용량을 여러 우선순위 레벨(Priority Levels)과 흐름(Flows)으로 나누어 할당함으로써, **중요한 요청이 덜 중요한 요청에 의해 밀려나지 않도록 보장**합니다.

| 용어                         | 설명                                                                                                                          |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| PriorityLevelConfiguration | 요청의 우선순위와 이에 할당될 API Server 총 용량(Concurrency)의 비율을 정의합니다. (ex. 시스템 컨트롤러는 High 우선순위, 일반 사용자는 Low 우선순위)                       |
| FlowSchema                 | 들어오는 요청을 특정 Priority Level로 분류하기 위한 규칙을 정의합니다. (ex. 특정 사용자, ServiceAccount, API Group 등에 따라 분류)                             |
| Seat                       | API Server에서 동시에 요청을 처리할 수 있는 단위 용량을 비유적으로 표현하는 용어. APF가 활성화되면 기존 max-inflight의 합이 APF의 총 Seat가 되어 각 Priority Level에 배분됩니다. |

<p align="center"><sup>[표] APF 용어</sup></p>

APF는 어떻게 동작할까요?

{% stepper %}
{% step %}
**분류 (Classification)**

들어오는 모든 요청은 FlowSchema를 통해 **하나의 Priority Level**에 할당됩니다.
{% endstep %}

{% step %}
**공정성 (Fairness)**

각 Priority Level은 전체 Seat 중 할당된 비율만큼 용량을 가집니다. 같은 Priority Level 내에서도 Flow 단위로 세분화하여 공정하게 자원을 배분합니다.
{% endstep %}

{% step %}
**대기열 (Queueing)**

만약 특정 Priority Level의 Seat가 모두 사용되었다면, 요청은 즉시 거부되지 않고 제한된 크기의 대기열(Queue)에 들어갑니다. 이는 단기적인 요청 폭주(Burst)를 흡수하여 429 에러 발생을 줄여줍니다.
{% endstep %}

{% step %}
**거부 (Rejection)**

대기열마저 가득 차면, 그 때서야 요청은 429 코드로 거부됩니다.
{% endstep %}
{% endstepper %}

### RBAC

RBAC(**Role-Based Access Contro**l, 역할 기반 접근 제어)는 **인가(Authorization)** 단계를 이해하는데 중요한 개념으로 쿠버네티스 클러스터 리소스에 접근하는 **주체(User, Group, ServiceAccount)에게 역할(Role)을 부여**해서 접근 권한을 통제하는 메커니즘입니다.

| 리소스                | 범위                   | 설명                                                                                                                          |
| ------------------ | -------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Role               | 네임스페이스(namespace) 한정 | 특정 네임스페이스 내의 리소스(ex. `dev` 네임스페이스의 `Pod`, `Service`)에 대한 권한을 정의                                                             |
| RoleBinding        | 네임스페이스(namespace) 한정 | 특정 사용자(ex. `ServiceAccount` 등)에게 Role을 부여                                                                                   |
| ClusterRole        | 클러스터 전체 범위           | <p>네임스페이스에 관계없이 클러스터 전체 리소스(ex. <code>Node</code>, <code>PersistentVolume</code>) 또는 모든 네임스페이스에 걸친 리소스에 대한 </p><p>권한 정의</p> |
| ClusterRoleBinding | 클러스터 전체 범위           | 사용자/그룹/ServiceAccount에 ClusterRole을 연결                                                                                      |

<p align="center"><sup>[표] RBAC 핵심 구성요소 네가지</sup></p>

쿠버네티스는 모든 API 요청에 대해 Authorization(인가) 단계를 RBAC로 수행합니다.&#x20;

{% stepper %}
{% step %}
**인증(Authentication)**

유효한 사용자/서비스인가?

* TLS(client cert), Bearer(Token/OIDC), ServiceAccount, etc.
* 실패 시 `401 Unauthorized`
{% endstep %}

{% step %}
**인가(Authorization)**

RBAC에서 허용된 권한인가?

* RBAC/ABAC/Webhook 등 조합, Impersonation 처리
* 실패 시 `403 Forbidden`&#x20;
{% endstep %}

{% step %}
**어드미션(Admission)**

정책(웹훅 등)에 의해 최종 허가되는가?

* Mutating Webhooks (필요 시 패치/기본값 채움)
* Validating Webhooks (정책 검증)
* 실패 시 `400/422`
{% endstep %}
{% endstepper %}



