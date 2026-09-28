---
title: 노드 셧다운
content_type: concept
weight: 10
---

<!-- overview -->

쿠버네티스 클러스터에서 {{< glossary_tooltip text="노드" term_id="node" >}}는
계획된 그레이스풀 방식으로 셧다운되거나 정전 또는 기타 외부 요인으로 인해
예기치 않게 셧다운될 수 있다. 노드가 셧다운되기 전에 드레인되지 않으면 워크로드
장애가 발생할 수 있다. 노드 셧다운은 **그레이스풀** 또는
**논 그레이스풀** 방식으로 수행될 수 있다.

{{< caution >}}
데비안의 `unattended-upgrades` 패키지는 일반적인 설정에서 노드의 그레이스풀 셧다운과
충돌한다.
서버의 셧다운 유예 기간을 사용자 지정하는 `unattended-upgrades`의 기본 설정을 사용하는 경우,
kubelet이 셧다운 이벤트를 제대로 처리하는 데 필요한 잠금을 획득하지 못한다.

이는 `shutdownGracePeriod` 값이 30초보다 큰 경우 발생한다.
이를 방지하려면 `unattended-upgrades` 설정의 일부를 비활성화할 수 있으며,
`/etc/systemd/logind.conf.d/unattended-upgrades-logind-maxdelay.conf`를 심볼릭 링크로 만들어
`/dev/null`을 가리키도록 한다.

자세한 내용은
[`logind.conf` 문서](https://www.freedesktop.org/software/systemd/man/latest/logind.conf.html)를 참고한다.
{{< /caution >}}

<!-- body -->

## 그레이스풀 노드 셧다운 {#graceful-node-shutdown}

kubelet은 노드 시스템 셧다운을 감지하고 노드에서 실행 중인 파드를 종료하려고 시도한다.

kubelet은 노드 셧다운 중에 파드가 일반적인
[파드 종료 프로세스](/ko/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)를
따르도록 한다. 노드 셧다운 중에는 kubelet이 새로운
파드를 수락하지 않는다(해당 파드가 이미 노드에 바인딩되어 있더라도 마찬가지이다).

### 그레이스풀 노드 셧다운 활성화

{{< tabs name="graceful_shutdown_os" >}}
{{% tab name="Linux" %}}
{{< feature-state feature_gate_name="GracefulNodeShutdown" >}}

Linux에서 그레이스풀 노드 셧다운 기능은 `GracefulNodeShutdown`
[기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)로 제어되며
1.21부터 기본적으로 활성화되어 있다.

{{< note >}}
그레이스풀 노드 셧다운 기능은 systemd에 의존하며, 이는
[systemd inhibitor locks](https://www.freedesktop.org/wiki/Software/systemd/inhibit/)를 활용하여
주어진 시간 동안 노드 셧다운을 지연시키기 때문이다.
{{</ note >}}
{{% /tab %}}

{{% tab name="Windows" %}}
{{< feature-state feature_gate_name="WindowsGracefulNodeShutdown" >}}

Windows에서 그레이스풀 노드 셧다운 기능은 `WindowsGracefulNodeShutdown`
[기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)로
제어되며, 1.32에서 알파 기능으로 도입되었다. 쿠버네티스 1.34에서 이 기능은 베타가 되었고
기본적으로 활성화되어 있다.

{{< note >}}
Windows 그레이스풀 노드 셧다운 기능은 kubelet이 Windows 서비스로 실행되는 것에 의존하며,
이 경우 [서비스 제어 핸들러](https://learn.microsoft.com/en-us/windows/win32/services/service-control-handler-function)가 등록되어
주어진 시간 동안 preshutdown 이벤트를 지연시킨다.
{{</ note >}}

Windows 그레이스풀 노드 셧다운은 취소할 수 없다.

kubelet이 Windows 서비스로 실행되지 않는 경우
[Preshutdown](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/ns-winsvc-service_preshutdown_info) 이벤트를 설정하고 모니터링할 수 없으며,
노드는 위에서 언급한 [논 그레이스풀 노드 셧다운](#non-graceful-node-shutdown) 절차를 거쳐야 한다.

Windows 그레이스풀 노드 셧다운 기능이 활성화되어 있지만 kubelet이
Windows 서비스로 실행되지 않는 경우, kubelet은 실패하지 않고 계속 실행된다. 그러나
Windows 서비스로 실행해야 한다는 오류를 로그에 기록한다.
{{% /tab %}}

{{< /tabs >}}

### 그레이스풀 노드 셧다운 구성

기본적으로 아래에서 설명하는 두 구성 옵션인
`shutdownGracePeriod`와 `shutdownGracePeriodCriticalPods`는 0으로 설정되어 있으며,
따라서 그레이스풀 노드 셧다운 기능이 활성화되지 않는다.
이 기능을 활성화하려면 두 옵션을 적절하게 구성하고
0이 아닌 값으로 설정해야 한다.

kubelet이 노드 셧다운 알림을 받으면 노드에 `NotReady` 컨디션을 설정하고
`reason`을 `"node is shutting down"`으로 설정한다. kube-scheduler는 이 컨디션을 따르며
영향을 받는 노드에 파드를 스케줄링하지 않는다. 다른 서드파티 스케줄러도
동일한 로직을 따를 것으로 예상된다. 이는 새로운 파드가 해당 노드에 스케줄링되지
않으며 따라서 어떤 파드도 시작되지 않음을 의미한다.

kubelet은 진행 중인 노드 셧다운을 감지한 경우 `PodAdmission` 단계에서도
파드를 **거부**하므로,
`node.kubernetes.io/not-ready:NoSchedule`에 대한
{{< glossary_tooltip text="톨러레이션" term_id="toleration" >}}이 있는 파드도 해당 노드에서 시작되지 않는다.

kubelet이 API를 통해 노드에 해당 컨디션을 설정할 때,
로컬에서 실행 중인 모든 파드도 종료하기 시작한다.

그레이스풀 셧다운 중에 kubelet은 파드를 두 단계로 종료한다.

1. 노드에서 실행 중인 일반 파드를 종료한다.
1. 노드에서 실행 중인 [중요 파드](/ko/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#파드를-중요-critical-로-표시하기)를
   종료한다.

그레이스풀 노드 셧다운 기능은 두 개의
[`KubeletConfiguration`](/docs/tasks/administer-cluster/kubelet-config-file/) 옵션으로 구성된다.

- `shutdownGracePeriod`:

  노드가 셧다운을 지연해야 하는 총 시간을 지정한다. 이는 일반 파드와
  [중요 파드](/ko/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#파드를-중요-critical-로-표시하기)의
  종료에 사용되는 전체 유예 기간이다.

- `shutdownGracePeriodCriticalPods`:

  노드 셧다운 중
  [중요 파드](/ko/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#파드를-중요-critical-로-표시하기)를
  종료하는 데 사용되는 시간을 지정한다. 이 값은 `shutdownGracePeriod`보다 작아야 한다.

{{< note >}}

시스템에 의해(또는 관리자가 수동으로) 노드 종료가 취소되는 경우가
있다. 이러한 경우 노드는 `Ready` 상태로 돌아간다.
그러나 이미 종료 프로세스를 시작한 파드는 kubelet에 의해 복원되지
않으며 다시 스케줄링되어야 한다.

{{< /note >}}

예를 들어 `shutdownGracePeriod=30s`이고
`shutdownGracePeriodCriticalPods=10s`이면 kubelet은 노드 셧다운을
30초 동안 지연시킨다. 셧다운 중 처음 20(30-10)초는
일반 파드를 그레이스풀하게 종료하는 데 사용되고 마지막 10초는
[중요 파드](/ko/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/#파드를-중요-critical-로-표시하기)를 종료하는 데 사용된다.

{{< note >}}
그레이스풀 노드 셧다운 중 축출된 파드는 셧다운된 것으로 표시된다.
`kubectl get pods`를 실행하면 축출된 파드의 상태가 `Terminated`로 표시된다.
또한 `kubectl describe pod`는 노드 셧다운으로 인해 파드가 축출되었음을 나타낸다.

```
Reason:         Terminated
Message:        Pod was terminated in response to imminent node shutdown.
```

{{< /note >}}

### 파드 우선순위 기반 그레이스풀 노드 셧다운 {#pod-priority-graceful-node-shutdown}

{{< feature-state feature_gate_name="GracefulNodeShutdownBasedOnPodPriority" >}}

그레이스풀 노드 셧다운 중 파드의 종료 순서에 더 많은 유연성을
제공하기 위해, 클러스터에서 이 기능을 활성화한 경우 그레이스풀 노드 셧다운은
파드의 PriorityClass를 고려한다. 이 기능을 통해 클러스터 관리자는
그레이스풀 노드 셧다운 중 파드의 종료 순서를
[프라이어리티 클래스](/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/#프라이어리티클래스)에 따라 명시적으로 정의할 
수 있다.

위에서 설명한 [그레이스풀 노드 셧다운](#graceful-node-shutdown) 기능은
파드를 두 단계로 종료한다. 중요하지 않은 파드를 먼저 종료하고 그다음 중요
파드를 종료한다. 셧다운 중 파드의 종료 순서를 보다 세분화하여
명시적으로 정의해야 하는 경우 파드 우선순위 기반 그레이스풀
셧다운을 사용할 수 있다.

그레이스풀 노드 셧다운에서 파드 우선순위를 고려하면
여러 단계로 그레이스풀 노드 셧다운을 수행할 수 있으며, 각 단계에서
특정 프라이어리티 클래스의 파드를 종료한다. kubelet에는 정확한
단계와 각 단계의 셧다운 시간을 구성할 수 있다.

클러스터에 다음과 같은 사용자 정의 파드
[프라이어리티 클래스](/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/#프라이어리티클래스)가
있다고 가정한다.

| 파드 프라이어리티 클래스 이름 | 파드 프라이어리티 클래스 값 |
| ----------------------- | ------------------------ |
| `custom-class-a`        | 100000                   |
| `custom-class-b`        | 10000                    |
| `custom-class-c`        | 1000                     |
| `regular/unset`         | 0                        |

[kubelet 구성](/docs/reference/config-api/kubelet-config.v1beta1/)에서
`shutdownGracePeriodByPodPriority` 설정은 다음과 같을 수 있다.

| 파드 프라이어리티 클래스 값 | 셧다운 기간 |
| ------------------------ | --------------- |
| 100000                   | 10초      |
| 10000                    | 180초     |
| 1000                     | 120초     |
| 0                        | 60초      |

이에 해당하는 kubelet 구성 YAML은 다음과 같다.

```yaml
shutdownGracePeriodByPodPriority:
  - priority: 100000
    shutdownGracePeriodSeconds: 10
  - priority: 10000
    shutdownGracePeriodSeconds: 180
  - priority: 1000
    shutdownGracePeriodSeconds: 120
  - priority: 0
    shutdownGracePeriodSeconds: 60
```

위 표는 `priority` 값이 100000 이상인 파드는
셧다운에 10초만 주어지고, 값이 10000 이상 100000 미만인 파드는 180
초, 값이 1000 이상 10000 미만인 파드는 120초의 셧다운 시간이 주어짐을 의미한다.
마지막으로 그 밖의 모든 파드에는 60초의 셧다운 시간이 주어진다.

모든 클래스에 해당하는 값을 지정할 필요는 없다. 예를
들어 다음 설정을 대신 사용할 수 있다.

| 파드 프라이어리티 클래스 값 | 셧다운 기간 |
| ------------------------ | --------------- |
| 100000                   | 300초     |
| 1000                     | 120초     |
| 0                        | 60초      |

위의 경우 `custom-class-b`를 사용하는 파드는 셧다운 시
`custom-class-c`와 동일한 버킷에 들어간다.

특정 범위에 파드가 없으면 kubelet은
해당 우선순위 범위의 파드를 기다리지 않는다. 대신 kubelet은 즉시
다음 프라이어리티 클래스 값 범위로 넘어간다.

이 기능이 활성화되어 있고 구성이 제공되지 않으면 순서 지정
동작은 수행되지 않는다.

이 기능을 사용하려면 `GracefulNodeShutdownBasedOnPodPriority`
[기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)를 활성화하고,
[kubelet 구성](/docs/reference/config-api/kubelet-config.v1beta1/)의
`ShutdownGracePeriodByPodPriority`를 원하는 구성으로 설정해야 하며,
여기에는 파드 프라이어리티 클래스 값과
각각의 셧다운 기간이 포함된다.

{{< note >}}
그레이스풀 노드 셧다운 중 파드 우선순위를 고려하는 기능은
쿠버네티스 v1.23에서 알파 기능으로 도입되었다. 쿠버네티스 {{< skew currentVersion >}}에서
이 기능은 베타이며 기본적으로 활성화되어 있다.
{{< /note >}}

노드 셧다운을 모니터링하기 위해 `graceful_shutdown_start_time_seconds`와 `graceful_shutdown_end_time_seconds`
메트릭이 kubelet 서브시스템에서 방출된다.

## 논 그레이스풀 노드 셧다운 처리 {#non-graceful-node-shutdown}

{{< feature-state feature_gate_name="NodeOutOfServiceVolumeDetach" >}}

노드 셧다운 동작은 kubelet의 노드 셧다운 매니저에서 감지되지 않을 수 있다.
이는 명령이 kubelet에서 사용하는 inhibitor lock 메커니즘을 트리거하지
않거나 사용자 오류, 즉 ShutdownGracePeriod와
ShutdownGracePeriodCriticalPods가 올바르게 구성되지 않았기 때문일 수 있다. 자세한 내용은 위의
[그레이스풀 노드 셧다운](#graceful-node-shutdown) 섹션을 참고한다.

노드가 셧다운되었지만 kubelet의 노드 셧다운 매니저에서 감지하지 못하면,
{{< glossary_tooltip text="스테이트풀셋" term_id="statefulset" >}}에 속하는 파드는
셧다운된 노드에서 종료 중 상태에 고착되어 새로운 실행 중인 노드로 이동할 수 없다.
이는 셧다운된 노드의 kubelet이 파드를 삭제할 수 없어
스테이트풀셋이 동일한 이름의 새 파드를 생성할 수 없기 때문이다. 파드에서 사용하는 볼륨이 있는 경우,
VolumeAttachment가 원래 셧다운된 노드에서 삭제되지 않으므로 해당 파드가
사용하는 볼륨을 새로 실행 중인 노드에 연결할 수 없다. 그 결과
스테이트풀셋에서 실행되는 애플리케이션이 제대로 작동할 수 없다. 원래
셧다운된 노드가 다시 실행되면 kubelet이 파드를 삭제하고 새로운 파드가
다른 실행 중인 노드에 생성된다. 원래 셧다운된 노드가 다시 실행되지 않으면
이 파드는 셧다운된 노드에서 계속 종료 중 상태로 남는다.

위 상황을 완화하기 위해 사용자는 `node.kubernetes.io/out-of-service` 테인트를
`NoExecute` 또는 `NoSchedule` 효과와 함께 노드에 수동으로 추가하여 서비스 불가 상태로 표시할 수 있다.
노드가 이 테인트로 서비스 불가 상태로 표시되면 노드의 파드에
일치하는 톨러레이션이 없는 경우 강제로 삭제되며, 노드에서 종료 중인 파드의 볼륨
분리 작업도 즉시 수행된다. 이를 통해 서비스 불가 상태인 노드의 파드가
다른 노드에서 빠르게 복구될 수 있다.

논 그레이스풀 셧다운 중 파드는 두 단계로 종료된다.

1. 일치하는 `out-of-service` 톨러레이션이 없는 파드를 강제로 삭제한다.
1. 해당 파드에 대한 볼륨 분리 작업을 즉시 수행한다.

{{< note >}}

- `node.kubernetes.io/out-of-service` 테인트를 추가하기 전에 노드가 이미
  셧다운 또는 전원 꺼짐 상태인지(재시작 중인 상태가 아닌지) 확인해야 한다.
- 사용자는 파드가 새 노드로 이동하고
  셧다운되었던 노드가 복구되었는지 확인한 후
  자신이 처음 추가했던 서비스 불가 상태 테인트를 수동으로 제거해야 한다.

{{< /note >}}

### 타임아웃 시 강제 스토리지 분리 {#storage-force-detach-on-timeout}

어떤 상황에서든 파드 삭제가 6분 동안 성공하지 못한 경우, 쿠버네티스는
그 시점에 노드가 비정상 상태라면 마운트 해제 중인 볼륨을 강제로 분리한다.
강제로 분리된 볼륨을 사용하는 워크로드가 노드에서 여전히 실행 중이라면
다음
[CSI 사양](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerunpublishvolume)을
위반하게 된다. 이 사양에서는 `ControllerUnpublishVolume`이 볼륨의 모든
`NodeUnstageVolume` 및 `NodeUnpublishVolume` 호출이 성공한 후 "**반드시** 호출되어야 한다"고 명시한다.
이러한 상황에서는 해당 노드의 볼륨에서 데이터 손상이 발생할 수 있다.

강제 스토리지 분리 동작은 선택 사항이며, 사용자는 대신 "논 그레이스풀
노드 셧다운" 기능을 사용할 수 있다.

타임아웃 시 강제 스토리지 분리는 `kube-controller-manager`의
`disable-force-detach-on-timeout` 구성 필드를 설정하여 비활성화할 수 있다. 타임아웃 시 강제 분리를 비활성화하면
6분 이상 비정상 상태인 노드에서 호스팅되는 볼륨은
관련
[VolumeAttachment](/docs/reference/kubernetes-api/config-and-storage-resources/volume-attachment-v1/)가
삭제되지 않는다.

이 설정을 적용한 후에도 볼륨에 연결된 비정상 파드는
위에서 언급한 [논 그레이스풀 노드 셧다운](#non-graceful-node-shutdown) 절차를 통해 복구해야 한다.

{{< note >}}

- [논 그레이스풀 노드 셧다운](#non-graceful-node-shutdown) 절차를 사용할 때는 주의해야 한다.
- 위에 문서화된 단계에서 벗어나면 데이터 손상이 발생할 수 있다.

{{< /note >}}

## {{% heading "whatsnext" %}}

다음 내용을 더 살펴본다.

- 블로그: [논 그레이스풀 노드 셧다운](/blog/2023/08/16/kubernetes-1-28-non-graceful-node-shutdown-ga/).
- 클러스터 아키텍처: [노드](/ko/docs/concepts/architecture/nodes/).
  
