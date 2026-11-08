# DevOps 로그 확인 명령어 가이드

이 문서는 운영체제, SSH, Docker, Kubernetes 로그를 확인할 때 자주 쓰는 읽기 전용 명령어를 정리합니다. 로그 위치와 서비스 이름은 배포판, 버전, 런타임, 설정에 따라 다르므로 하나의 경로가 모든 시스템에 적용된다고 가정하지 말고 해당 호스트의 설정을 먼저 확인하세요.

`<service>`, `<container>`, `<namespace>`, `<pod>`, `<container-name>`, `<label>` 같은 자리표시자는 실제 값으로 바꾸고 꺾쇠괄호는 입력하지 않습니다. 아래 명령어는 로그를 조회하거나 새 로그를 따라가며, 로그를 삭제하거나 수정하지 않습니다. 일부 로그는 관리자/root 권한 또는 로그 조회 그룹 권한이 필요합니다.

## 목차

- [기본 장애 분석 순서](#기본-장애-분석-순서)
- [Linux](#linux)
- [Unix, BSD, macOS](#unix-bsd-macos)
- [Windows](#windows)
- [SSH](#ssh)
- [Docker](#docker)
- [kubectl을 이용한 Kubernetes 로그 확인](#kubectl을-이용한-kubernetes-로그-확인)
- [검색, 필터링, 안전한 취급](#검색-필터링-안전한-취급)
- [공식 문서](#공식-문서)

## 기본 장애 분석 순서

1. 대상 호스트, 서비스 또는 워크로드, 네임스페이스, 대략적인 발생 시간을 확인합니다. 타임스탬프가 맞지 않으면 시스템 시계와 시간대를 확인합니다.
2. 시스템 서비스 저널/이벤트 로그, 컨테이너의 stdout/stderr, Kubernetes Pod 및 컨테이너 로그처럼 가장 관련성이 높은 로그부터 확인합니다.
3. 컨테이너가 재시작되거나 비정상 종료됐다면 현재 로그와 이전 컨테이너 로그를 모두 확인합니다. 애플리케이션 오류를 서비스 재시작, 노드 문제, Kubernetes 이벤트와 비교합니다.
4. 필요한 경우에만 조회 시간 범위를 넓히거나 관련 구성 요소를 확인합니다. 조사 내용을 공유할 때는 원래 타임스탬프와 맥락을 보존합니다.

## Linux

### systemd 저널

`journalctl`은 많은 Linux 배포판에서 사용하는 systemd 저널 로그를 읽는 명령어입니다. systemd를 사용하는 시스템에서만 쓸 수 있으며, 모든 Unix/macOS/Windows에서 통용되는 명령어는 아닙니다. 특정 서비스의 로그, 최근 시간 범위의 로그, 지정한 심각도 수준의 메시지를 조회할 때 사용합니다.

systemd 기반 Linux 호스트에서 다음 읽기 전용 명령어를 사용할 수 있습니다.

```bash
sudo journalctl -u <service> --since "1 hour ago"
sudo journalctl -u <service> -n 200 --no-pager
sudo journalctl -u <service> -f
sudo journalctl -p err --since "1 hour ago"
sudo journalctl -k --since today
```

`<service>`는 systemd 유닛 이름으로 바꿉니다. 예를 들어 SSH 유닛은 Debian/Ubuntu에서 `ssh`, 많은 RHEL 계열에서 `sshd`일 수 있습니다. `-u`는 서비스 선택, `-n`은 최근 출력 개수 제한, `--no-pager`는 결과를 터미널에 바로 출력, `-f`는 중단할 때까지 새 메시지 추적, `--since`는 조회 시작 시간, `-p err`는 오류 우선순위 메시지 선택, `-k`는 커널 메시지 선택 옵션입니다. 보관된 부팅 목록은 `journalctl --list-boots`로 확인하고, 이전 부팅 로그가 보관되어 있다면 `journalctl -b -1`로 조회할 수 있습니다.

systemd를 사용하지 않거나 저널 저장이 설정되지 않았거나 보관되지 않아 저널을 볼 수 없다면 `/var/log/` 아래의 배포판/서비스별 파일과 서비스에 설정된 로그 대상을 확인하세요. 아래 경로는 흔한 예시일 뿐 모든 시스템에 있는 것은 아닙니다.

### 텍스트 기반 시스템 로그

전통적인 syslog 파일은 systemd를 사용하지 않는 시스템에서 흔하며, systemd 저널과 함께 존재할 수도 있습니다. 일반적인 위치는 다음과 같지만 배포판에 따라 다릅니다.

| 로그 목적 | 일반적인 위치(배포판별 차이 있음) |
|---|---|
| 일반 시스템 메시지 | Debian/Ubuntu의 `/var/log/syslog`, 많은 RHEL 계열의 `/var/log/messages` |
| 인증 및 SSH | Debian/Ubuntu의 `/var/log/auth.log`, 많은 RHEL 계열의 `/var/log/secure` |
| 커널 메시지 | 일부 Debian 계열의 `/var/log/kern.log`; 그 외에는 저널 또는 syslog 설정 확인 |
| 서비스별 로그 | `/var/log/` 아래 설정된 파일 또는 systemd 저널 |

먼저 실제 존재하는 파일을 확인합니다.

```bash
sudo ls -l /var/log
sudo tail -n 200 /var/log/syslog
sudo tail -f /var/log/auth.log
sudo grep -i "error" /var/log/syslog
```

호스트에 존재하는 경로만 사용하세요. 순환된 로그에는 `.1` 또는 날짜 접미사가 붙을 수 있고, 압축 로그는 보통 `.gz`로 끝납니다. `zless`, `zgrep` 등으로 확인할 수 있습니다.

## Unix, BSD, macOS

Unix 계열에 공통으로 적용되는 단일 로그 경로나 서비스 관리자는 없습니다. 먼저 플랫폼과 syslog 설정의 로그 대상을 확인하세요.

### BSD 및 기타 Unix 계열

FreeBSD에서는 시스템 메시지에 `/var/log/messages`, 인증에 `/var/log/auth.log`를 사용하는 경우가 많지만 실제 파일은 `syslog.conf`와 버전에 따라 다릅니다. 다른 Unix 계열은 로그 기능, 파일명, 중앙 로그 수집 방식이 다를 수 있습니다.

```sh
uname -a
ls -l /var/log
tail -n 200 /var/log/messages
grep -i "error" /var/log/messages
```

실제로 존재하는 파일만 사용하세요. 예를 들어 OpenBSD는 인증 관련 메시지를 `/var/log/authlog`에 기록하는 경우가 많습니다. 해당 호스트의 `syslog.conf`와 매뉴얼 페이지를 확인하세요.

### macOS

최근 macOS는 모든 시스템 메시지를 전통적인 텍스트 파일에 기록하는 대신 통합 로깅을 사용합니다. `log` 명령어로 최근 로그를 조회할 수 있습니다.

```sh
log show --last 1h --style compact
log show --last 1h --style compact --predicate 'process == "sshd"'
log stream --style compact --predicate 'process == "sshd"'
```

`log show`는 보관된 로그를 조회하고, `log stream`은 종료할 때까지 새 로그를 추적합니다. 일부 로그는 비공개 정보로 처리되거나 권한이 없어 표시되지 않을 수 있습니다. 서드파티 서비스는 애플리케이션에 설정된 로그 대상도 확인하세요.

## Windows

Windows 시스템 및 서비스 이벤트는 주로 이벤트 뷰어와 PowerShell의 `Get-WinEvent`에서 확인합니다. 일반적인 채널은 **System**, **Application**, **Security** 및 서비스별 채널입니다. Security 로그는 접근이 제한됩니다.

### 이벤트 뷰어 GUI

**이벤트 뷰어**(`eventvwr.msc`)를 열고 **Windows 로그**에서 **System**, **Application**, **Security**를 선택합니다. 시간, 원본, 이벤트 수준, 이벤트 ID로 필터링하세요. 제품별 채널은 **Applications and Services Logs**에서 확인합니다.

### PowerShell

```powershell
Get-WinEvent -LogName System -MaxEvents 100
Get-WinEvent -LogName Application -MaxEvents 100 |
  Select-Object TimeCreated, ProviderName, Id, LevelDisplayName, Message
Get-WinEvent -FilterHashtable @{ LogName = 'System'; StartTime = (Get-Date).AddHours(-1) }
Get-WinEvent -ListLog * |
  Select-Object LogName, RecordCount, IsEnabled
```

알고 있는 이벤트 채널을 조회할 때는 `-LogName`의 이름을 바꾸세요. 채널이 비활성화되어 있거나 접근이 거부될 수 있으며, 관리자 권한 PowerShell이 필요한 경우도 있습니다.

### Windows 서비스 및 파일

**서비스**(`services.msc`)에서 서비스 이름과 상태를 확인한 후 System/Application 이벤트 채널 또는 제품 자체의 로그 디렉터리를 확인합니다. 일반적인 파일 경로는 `%ProgramData%\<vendor>\<product>\Logs` 또는 애플리케이션 설치/데이터 디렉터리이지만, 모든 Windows 프로그램에 공통인 경로는 아닙니다.

Windows OpenSSH는 이벤트 뷰어의 **Applications and Services Logs > OpenSSH > Operational**에서 확인합니다. PowerShell 로그 이름은 보통 `OpenSSH/Operational`입니다.

```powershell
Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 100
```

## SSH

SSH 문제를 확인할 때는 클라이언트 연결 진단과 서버 인증/서비스 로그를 구분합니다.

### 클라이언트 연결 진단

클라이언트에서 verbose 모드로 SSH를 실행하면 이름 확인, 키 교환, 인증 방식, 연결 실패 지점을 확인할 수 있습니다.

```sh
ssh -v <user>@<host>
ssh -vvv -i <private-key-path> <user>@<host>
```

필요한 정보를 얻을 수 있는 가장 낮은 verbosity를 사용하세요. 출력에 사용자명, 호스트명, 파일 경로, 인증 메타데이터가 포함될 수 있으므로 공유 전에 검토해야 합니다. 개인 키를 로그나 티켓에 보내거나 내용을 복사하지 마세요.

### 서버 로그

Linux에서는 우선 SSH 서비스 저널을 확인합니다.

```bash
sudo journalctl -u sshd --since "1 hour ago"
sudo journalctl -u ssh --since "1 hour ago"
```

해당 호스트에 존재하는 유닛 이름을 사용하세요. 인증 이벤트를 파일에 기록하는 경우 Debian/Ubuntu는 `/var/log/auth.log`, 많은 RHEL 계열은 `/var/log/secure`를 사용하는 일이 많습니다. BSD와 macOS는 자체 syslog 또는 통합 로깅 설정을 사용하므로 위 플랫폼별 항목을 참고하세요.

Windows OpenSSH Server는 이벤트 뷰어 또는 `Get-WinEvent`에서 `OpenSSH/Operational` 채널을 확인합니다. 서버 로그 위치와 상세 수준은 별도로 설정할 수 있습니다. 클라이언트 출력만으로 서버에 어떤 로그가 남았는지 알 수는 없습니다.

## Docker

컨테이너 로그 저장 위치와 조회 방법은 Docker 로깅 드라이버에 따라 다릅니다. 기본 `json-file` 드라이버의 경우 내부 파일을 직접 읽기보다 Docker CLI를 사용하세요.

```bash
docker ps
docker logs --tail 200 --timestamps <container>
docker logs --since 1h --tail 200 --timestamps --follow <container>
docker inspect --format '{{.HostConfig.LogConfig.Type}}' <container>
docker info --format '{{.LoggingDriver}}'
```

`docker logs`는 일반적으로 컨테이너의 stdout/stderr를 보여줍니다. 애플리케이션이 컨테이너 내부 파일에만 로그를 기록하면 유용한 출력이 없을 수 있습니다. 기본값이 아닌 로깅 드라이버는 보관 방식이 다르거나 일부 로그 조회 옵션을 지원하지 않을 수 있으므로 해당 드라이버 문서를 확인하세요.

Docker Compose를 사용할 때는 다음과 같이 조회합니다.

```bash
docker compose ps
docker compose logs --tail=200 <service>
docker compose logs --since=1h --tail=200 --follow <service>
```

Docker 내부 저장 경로를 기준으로 작업하지 마세요. 플랫폼, 드라이버, 데몬 설정에 따라 다릅니다. 디스크 공간을 확보하려고 로그 파일을 삭제하거나 비우지 말고 설정된 순환/보관 정책을 따르세요.

## kubectl을 이용한 Kubernetes 로그 확인

먼저 현재 클러스터 컨텍스트를 확인하고 대상 네임스페이스, Pod, 컨테이너를 찾습니다. 명령이 운영 환경 데이터에 접근할 수 있으므로 의도한 컨텍스트인지 확인하세요.

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> -l <label>
kubectl describe pod <pod> -n <namespace>
```

### Pod 로그 조회 및 추적

```bash
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -c <container-name>
kubectl logs <pod> -n <namespace> --all-containers=true --since=1h --tail=200 --timestamps
kubectl logs <pod> -n <namespace> -c <container-name> --follow
```

Pod에 컨테이너가 여러 개면 `-c`로 하나를 선택하거나 `--all-containers=true`를 사용합니다. Deployment가 관리하는 Pod의 경우 먼저 Deployment의 레이블 셀렉터 등을 이용해 Pod를 나열한 뒤 대상 Pod를 조회하세요. 그러면 어떤 복제본의 출력인지 분명하게 확인할 수 있습니다.

### 이전 컨테이너 인스턴스 및 이벤트

컨테이너가 재시작된 경우 런타임이 이전 인스턴스를 보관하고 있으면 다음처럼 확인합니다.

```bash
kubectl logs <pod> -n <namespace> -c <container-name> --previous
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod> -n <namespace>
```

`--previous`는 Pod 내에서 직전에 종료된 컨테이너 로그를 확인하는 데 유용하지만, 과거 모든 인스턴스의 로그를 보관하는 기능은 아닙니다. 이벤트는 수명이 짧은 클러스터 객체이며 보관 기간은 클러스터 설정에 따라 다릅니다. 장기간 로그가 필요하면 클러스터에 구성된 로그 백엔드를 사용하세요.

Kubernetes는 일반적으로 컨테이너 stdout/stderr를 노드의 컨테이너 런타임을 통해 수집하지만, 노드 파일 경로는 구현 세부사항이며 순환되거나 접근할 수 없을 수 있습니다. 노드 경로를 직접 읽기보다 `kubectl logs`와 클러스터에서 지원하는 로그 시스템을 우선 사용하세요. 현재 컨텍스트, RBAC 권한, 워크로드 상태에 따라 조회 가능 범위가 달라집니다.

## 검색, 필터링, 안전한 취급

- 큰 로그를 검색하기 전에 `tail`, `-n` 또는 해당 도구가 지원하는 시간 범위 옵션으로 출력 범위를 제한합니다.
- 별도 지정이 없으면 검색은 대소문자를 구분할 수 있습니다. 일치한 행 주변 맥락을 확인한 뒤 오류인지 판단하세요.
- 시스템 간 타임스탬프를 비교할 때 시간대와 시계를 확인합니다. 로그 일부를 공유할 때는 시간대 정보를 보존하세요.
- `sudo`, 관리자 PowerShell, `systemd-journal` 또는 `adm` 그룹 권한이 필요할 수 있습니다. 필요한 최소 권한만 요청하세요.
- 로그에는 토큰, 세션 ID, 개인정보, 내부 호스트명, IP 주소, 요청 본문이 포함될 수 있습니다. 공유 전에 민감한 값을 가리고 조직의 접근, 보존, 보안 사고 대응 정책을 따르세요.
- 일반적인 장애 분석 중 로그를 비우거나 자르거나 삭제하지 마세요. 증거를 훼손하거나 보존 요구사항을 위반할 수 있습니다. 승인된 순환, 보존 및 수집 절차를 따르세요.

## 공식 문서

- [systemd journalctl 매뉴얼](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft: Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [OpenSSH 매뉴얼](https://www.openssh.com/manual.html)
- [Docker: 컨테이너 로그 보기](https://docs.docker.com/engine/logging/)
- [Kubernetes: 로깅 아키텍처](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Kubernetes: kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
