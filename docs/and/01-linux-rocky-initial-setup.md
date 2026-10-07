# Linux 기초 & Rocky Linux 초기 구축 (VMware 기반)

실무에서 CentOS Linux가 종료된 지금 Rocky Linux를 RHEL 대체재로 도입하는 배경과, Rocky Linux 기반 서버를 처음 구축할 때 필요한 커널·SELinux·SSH 초기 설정을 정리한다.

---

## 1. 개요 (Overview)

실무에서 CentOS Linux가 종료된 지금, Rocky Linux는 CentOS Linux의 대안으로 등장한 무료 엔터프라이즈 리눅스 배포판으로, **RHEL과의 높은 호환성**을 목표로 하는 것이 RHEL 대체재로 선택되는 핵심 이유다. 기존 CentOS/RHEL 기반 운영 환경을 비교적 적은 변경으로 이전할 수 있다는 점이 가장 큰 장점이며, Rocky Linux는 CentOS 공동 창립자인 **Gregory Kurtzer**가 개발을 주도했고 현재는 Rocky Enterprise Software Foundation(RESF)을 중심으로 운영된다. Rocky Linux는 RHEL과의 **Bug-for-Bug Compatibility**를 목표로 설명하고 있지만, 실제 도입 시에는 패키지 호환성과 별개로 해당 소프트웨어 벤더가 Rocky Linux를 공식 지원하는지 반드시 확인해야 한다. RHEL용 프로그램이 Rocky Linux에서 기술적으로 동작하더라도, Oracle DB·SAP·백신·백업 솔루션·스토리지 드라이버 등의 **공식 기술 지원 및 HCL 인증이 자동으로 보장되는 것은 아니다.**

Ubuntu와 비교했을 때의 판단 기준을 보면, **RHEL 계열(RHEL/Rocky/Alma)**은 기존 CentOS 또는 RHEL 기반 시스템을 마이그레이션해야 하는 환경, `dnf`·RPM·SELinux·firewalld·systemd 기반 운영 표준을 사용하는 환경, 금융·공공·제조·기업 서버 등 장기간 안정적인 운영이 필요한 환경, 상용 솔루션의 RHEL 계열 지원 여부가 중요한 환경, 최신 기능보다 안정성과 장기 유지보수가 우선인 환경에 적합하다. 반면 **Ubuntu**는 Ubuntu LTS를 표준 운영체제로 사용하는 조직, 클라우드·컨테이너·Kubernetes·DevOps 중심 환경, AI/ML 관련 패키지 및 개발 생태계를 중요하게 보는 환경, `apt` 및 DEB 패키지 생태계에 익숙한 환경, Canonical의 상용 지원이 필요한 환경에 적합하다. **CentOS Stream**은 단순한 "불안정한 베타판"이라기보다 **다음 RHEL 마이너 릴리스에 반영될 변경 사항을 먼저 확인할 수 있는 지속 배포형 배포판**으로, 일반적인 기술 흐름은 다음과 같이 이해할 수 있다.

```text
Fedora → CentOS Stream → RHEL
                         ↓
                    Rocky Linux
                    AlmaLinux
```

CentOS Stream도 개발·검증·CI 환경이나 RHEL의 향후 변경 사항을 미리 확인하는 용도로 사용할 수 있으며, 프로덕션 사용 가능 여부는 조직의 검증 기준과 지원 정책에 따라 판단해야 하고 무조건 "사용 불가"라고 단정하는 것은 적절하지 않다.

서버 구축 후 SELinux를 Disable하는 것에 대해서는, 운영 환경에서는 SELinux 비활성화를 권장하지 않는다. 문제가 발생했다면 먼저 `permissive` 모드에서 로그를 수집하고, 필요한 정책을 추가한 뒤 `enforcing` 상태로 운영하는 것이 표준적인 접근 방법이다. 학습·실습 환경에서는 교육 목적에 따라 SELinux를 비활성화할 수 있지만, 운영 환경에 그대로 적용해서는 안 된다. SELinux의 모드는 `enforcing`(SELinux 정책을 적용하고 위반 동작을 차단), `permissive`(위반 동작을 차단하지 않고 로그만 기록), `disabled`(SELinux 정책을 로드하지 않음) 세 가지로 구분되며, 운영 환경에서 권장하는 절차는 다음과 같다: 1) `enforcing` 상태에서 장애 현상 확인, 2) 일시적으로 `permissive`로 전환, 3) `/var/log/audit/audit.log`에서 거부 로그 확인, 4) 파일 컨텍스트, SELinux Boolean 및 서비스 설정 점검, 5) 필요한 정책만 예외 처리, 6) 다시 `enforcing`으로 전환.

```bash
# 현재 SELinux 모드 확인
getenforce
sestatus

# 일시적으로 permissive 전환
# 재부팅하면 설정 파일의 값으로 돌아간다.
setenforce 0

# 다시 enforcing 전환
setenforce 1

# 최근 SELinux 거부 로그 확인
ausearch -m AVC,USER_AVC -ts recent

# 특정 서비스와 관련된 SELinux Boolean 확인
getsebool -a

# SELinux 파일 컨텍스트 복구
restorecon -Rv /경로
```

`audit2allow`는 로그를 기반으로 정책을 생성할 수 있지만, 원인을 검토하지 않고 결과를 그대로 적용하면 과도한 권한을 허용할 수 있으므로 주의해야 한다. `SELINUX=disabled` 또는 커널 파라미터 `selinux=0`을 사용한 뒤 다시 SELinux를 활성화하면 전체 파일 시스템의 **재레이블링(relabeling)**이 필요할 수 있다.

리눅스(Linux)는 엄밀하게 말하면 운영체제 전체가 아니라 **운영체제의 핵심인 커널(Kernel)**을 의미한다. 일반적으로 리눅스 커널과 GNU 도구, 라이브러리, 패키지 관리자, 데스크톱 환경 등을 결합한 운영체제를 리눅스 배포판이라고 하며, 리눅스는 Unix의 설계 철학과 동작 방식을 따르는 **Unix-like 운영체제**이지만 원래 Unix 소스 코드를 그대로 복제한 것은 아니다. 오픈소스로 공개되어 전 세계 개발자가 자유롭게 검토하고 개선할 수 있다.

리눅스의 역사를 보면, 1991년 헬싱키 대학교 학생이었던 **리누스 토르발스(Linus Torvalds)**가 리눅스 커널 개발을 시작했다. Linux 0.01은 초기 개발 버전이었으며, 이후 0.02 버전부터 외부 개발자들이 사용할 수 있는 형태로 공개되기 시작했다. 리누스 토르발스는 리눅스 운영체제 전체가 아니라 커널을 개발했으며, 이후 GNU 프로젝트의 컴파일러, 쉘, 라이브러리 및 각종 도구가 리눅스 커널과 결합되면서 완전한 운영체제 환경이 만들어졌다.

리눅스 배포판은 다음 구성 요소를 조합하여 제공한다.

```text
Linux Kernel
+ GNU 도구
+ 시스템 라이브러리
+ 패키지 관리자
+ 시스템 관리 도구
+ 데스크톱 환경 또는 서버 프로그램
```

대표적인 리눅스 배포판으로는 Red Hat Enterprise Linux, Rocky Linux, AlmaLinux, Fedora, Ubuntu, Debian, SUSE Linux Enterprise, openSUSE가 있다.

GNU 프로젝트의 시작 배경을 보면, GNU는 **"GNU's Not Unix"**의 재귀적 약어이다. GNU 프로젝트는 리처드 스톨만(Richard Stallman)이 1983년에 발표하고 1984년부터 본격적으로 개발을 시작한 자유 소프트웨어 프로젝트이며, 목표는 사용자가 자유롭게 실행·연구·수정·재배포할 수 있는 Unix 호환 운영체제를 만드는 것이었다. FSF(Free Software Foundation)는 1985년 리처드 스톨만이 설립했으며 자유 소프트웨어의 개발과 배포, 사용자 권리 보호를 지원한다. 여기서 말하는 "Free"는 단순히 가격이 무료라는 뜻이 아니라 사용자의 **자유(Freedom)**를 의미한다. 자유 소프트웨어의 핵심 자유는 프로그램을 원하는 목적으로 실행할 자유, 프로그램의 동작 방식을 연구하고 수정할 자유, 프로그램을 복제하여 다른 사람에게 배포할 자유, 수정한 프로그램을 다시 배포할 자유로 구성된다.

GNU 프로젝트의 많은 프로그램은 GPL(General Public License) 라이선스로 배포된다. GPL은 프로그램의 사용·수정·재배포를 허용하며, GPL 프로그램 또는 그 파생물을 배포할 때는 라이선스 조건에 따라 해당 수령자에게 대응하는 소스 코드를 제공해야 한다. 수정한 GPL 프로그램을 재배포할 때도 동일한 GPL 조건을 유지해야 하는데, 이를 **카피레프트(Copyleft)**라고 한다. 프리웨어(Freeware)는 가격이 무료인 소프트웨어이지만 소스 코드가 공개되거나 수정·재배포가 허용된다는 보장은 없는 반면, 자유 소프트웨어(Free Software)는 실행·연구·수정·재배포의 자유를 중요하게 생각하는 소프트웨어로 반드시 가격이 무료여야 하는 것은 아니다. 자유 소프트웨어는 유료로 판매할 수도 있지만, GPL 프로그램을 배포하는 경우에는 GPL 조건에 따라 수령자에게 소스 코드 제공 의무가 발생할 수 있다.

가상머신(Virtual Machine, VM)은 물리적 컴퓨터의 CPU, 메모리, 디스크, 네트워크 등을 가상화하여 독립된 컴퓨터처럼 사용할 수 있게 하는 기술이다. 하나의 물리적 컴퓨터에서 여러 운영체제를 동시에 실행할 수 있으며, 각 가상머신은 독립된 CPU, 메모리, 디스크 및 네트워크 인터페이스를 할당받은 것처럼 동작한다. 테스트, 개발, 서버 실습, 장애 재현, 보안 격리 등에 유용하다. **호스트 OS(Host Operating System)**는 실제 컴퓨터에 설치된 운영체제로, VMware Workstation과 같은 가상화 프로그램이 실행되는 기반 운영체제이며 예로는 Windows 11, Linux가 있다. **게스트 OS(Guest Operating System)**는 가상머신 내부에 설치되는 운영체제로, 예로는 Rocky Linux, Ubuntu, Windows Server가 있고 다른 가상머신 및 호스트와 논리적으로 분리되어 동작한다.

멀티부팅과 가상머신의 차이는 다음 표와 같다.

| 구분 | 멀티부팅 | 가상머신 |
|---|---|---|
| 운영체제 실행 | 한 번에 하나만 실행 | 여러 운영체제 동시 실행 가능 |
| 운영체제 전환 | 재부팅 필요 | 창 전환으로 가능 |
| 설치 위치 | 별도 디스크 또는 파티션 | 가상 디스크 파일 |
| 하드웨어 접근 | 물리 장치에 직접 접근 | 가상화된 장치 사용 |
| 주요 용도 | 성능이 중요한 실제 사용 | 개발·테스트·교육·실습 |

하이퍼바이저(Hypervisor)는 가상머신을 생성하고 실행·관리하는 소프트웨어 계층으로, 물리 시스템의 CPU, RAM, 디스크, 네트워크 자원을 가상머신에 배분한다. **Type 1 하이퍼바이저**는 물리 하드웨어에서 직접 실행되며 예로는 VMware ESXi, Microsoft Hyper-V Server 계열, Xen이 있다. **Type 2 하이퍼바이저**는 호스트 운영체제 위에서 프로그램 형태로 실행되며 예로는 VMware Workstation Pro, Oracle VirtualBox가 있다.

VMware의 주요 특징은 한 대의 물리 컴퓨터에서 여러 서버 및 클라이언트 환경 구성, NAT·브리지·호스트 전용 네트워크 구성, CPU·메모리·디스크 및 네트워크 어댑터 변경, 다중 NIC·LVM·RAID 및 라우팅 실습, 스냅샷을 이용한 특정 시점 저장 및 복원, Suspend를 이용한 실행 상태 일시 중단 및 재개, Clone을 이용한 가상머신 복제, 호스트 시스템과 분리된 실습 환경 제공이다.

> **참고:** 현재 VMware Workstation Pro는 개인·교육·상업적 용도로 무료 제공된다. 기존 VMware Workstation Player는 독립 제품으로 중단되었으므로, 과거의 Pro/Player 기능 및 라이선스 비교표를 현재 기준으로 그대로 적용하면 안 된다.

VMware Workstation Pro의 주요 기능은 다음과 같다.

| 항목 | VMware Workstation Pro |
|---|---|
| 지원 호스트 OS | Windows 및 Linux |
| 지원 게스트 OS | Windows, Linux 등 |
| 라이선스 | 현재 무료 제공 |
| 가상머신 생성 | 지원 |
| 스냅샷 | 지원 |
| 가상 네트워크 편집 | 지원 |
| Clone | 지원 |
| Suspend | 지원 |

실습용 가상머신 구성 예시는 다음과 같다.

| 항목 | Server-A | Server-B | Client-L | WinClient |
|---|---|---|---|---|
| 주 용도 | 서버 전용 | 테스트 서버 | Linux 클라이언트 | Windows 클라이언트 |
| 게스트 OS 종류 | RHEL 9 계열 64-bit | RHEL 9 계열 64-bit | RHEL 9 계열 64-bit | Windows 10/11 64-bit |
| 설치 ISO | Rocky Linux 9 | Rocky Linux 9 | Rocky Linux 9 | Windows ISO |
| 가상머신 이름 | Server-A | Server-B | Client-L | WinClient |
| 저장 폴더 | `C:\Rocky9\Server-A` | `C:\Rocky9\Server-B` | `C:\Rocky9\Client-L` | `C:\Rocky9\WinClient` |
| 디스크 용량 | 80GB | 40GB | 40GB | 60GB |
| 디스크 타입 | SCSI | SCSI | SCSI 또는 NVMe | SCSI 또는 NVMe |
| 메모리 할당 | 2GB 이상 | 2GB 이상 | 2GB 이상 | 4GB 이상 권장 |
| 네트워크 타입 | NAT | NAT | NAT | NAT |
| CD/DVD | 사용 | 사용 | 사용 | 사용 |
| Audio | 제거 가능 | 제거 가능 | 선택 | 선택 |
| USB | 선택 | 선택 | 선택 | 선택 |
| Printer | 제거 가능 | 제거 가능 | 제거 가능 | 선택 |

커널(Kernel)은 운영체제의 핵심으로, 하드웨어와 사용자 프로그램 사이에서 자원을 관리한다. CPU, 메모리, 디스크, 네트워크 및 입출력 장치를 관리하며, 사용자 프로그램이 하드웨어에 직접 접근하는 대신 시스템 호출을 통해 커널의 기능을 사용하도록 한다. 커널의 주요 기능은 다음과 같다. **프로세스 관리**는 프로그램 실행 및 종료, CPU 스케줄링, 프로세스 간 통신을 포함하고, **메모리 관리**는 물리 메모리 할당 및 회수, 가상 메모리 관리, 프로세스별 메모리 공간 보호를 포함한다. **파일 시스템 관리**는 파일 읽기 및 쓰기, 디렉터리와 권한 관리, 다양한 파일 시스템 지원을 포함하며, **장치 관리**는 디스크·키보드·마우스·그래픽카드·네트워크 장치 관리 및 장치 드라이버 제공을 포함한다. **네트워크 관리**는 TCP/IP 네트워크 스택, 패킷 송수신, 라우팅 및 네트워크 인터페이스 관리를 포함한다.

커널 버전 체계를 보면, 예를 들어 업스트림 커널 버전이 `5.15.83`이라면 다음과 같이 구분할 수 있다.

```text
5  : 주 버전
15 : 부 버전
83 : 패치 버전
```

현재의 Linux 버전 번호는 반드시 변경 규모나 호환성 파괴 정도만을 의미하지는 않으며, 배포판 커널에는 업스트림 버전 외에 배포판 릴리스 번호가 추가된다.

```bash
uname -r
```

예시:

```text
5.14.0-xxx.el9_6.x86_64
```

Rocky Linux 9 계열은 기본적으로 RHEL 9과 같은 계열의 `5.14` 기반 커널을 사용한다. 버전 번호가 낮아 보여도 보안 패치와 기능이 백포트되므로, 단순히 업스트림 버전 번호만으로 오래된 커널이라고 판단하면 안 되며, 기업 환경에서는 호환성과 벤더 지원을 위해 배포판이 제공하는 기본 커널을 유지하는 것이 일반적이다.

쉘(Shell)은 사용자가 명령어를 입력하고 프로그램을 실행할 수 있도록 제공되는 인터페이스이다. 사용자가 입력한 명령을 분석하고 필요한 프로그램을 실행하며, 결과를 화면에 표시한다.

```text
사용자 ↔ 쉘 ↔ 프로그램·시스템 호출 ↔ 커널 ↔ 하드웨어
```

쉘의 역할은 명령어 해석, 외부 프로그램 실행, 환경변수 관리, 입출력 리다이렉션, 파이프 처리, 쉘 스크립트를 통한 작업 자동화, 명령어 기록 및 작업 제어이다. 대표적인 CLI 쉘로는 `bash`, `sh`, `zsh`, `ksh`가 있으며, 그래픽 쉘 및 데스크톱 환경으로는 GNOME Shell, KDE Plasma가 있다.

쉘 동작 예시로, 사용자가 다음 명령을 입력한다고 가정한다.

```bash
ls -l
```

처리 과정은 다음과 같다: 1) 쉘이 명령어와 옵션을 분석한다, 2) 쉘이 실행 경로에서 `ls` 프로그램을 찾는다, 3) `ls` 프로그램을 새 프로세스로 실행한다, 4) `ls`가 시스템 호출을 통해 디렉터리 정보를 요청한다, 5) 커널이 파일 시스템에서 정보를 읽어 프로그램에 전달한다, 6) 프로그램의 출력 결과가 터미널에 표시된다.

Red Hat 계열 리눅스를 살펴보면, Red Hat Linux는 Red Hat이 과거에 제공했던 리눅스 배포판이다. 2003년 이후 기업용 제품은 **Red Hat Enterprise Linux(RHEL)** 중심으로 재편되었으며, RHEL은 유료 서브스크립션 기반으로 기술 지원, 보안 업데이트, 장기 유지보수 및 인증 생태계를 제공한다. Red Hat 계열의 특징은 RPM 패키지 형식, `dnf` 패키지 관리자, systemd 서비스 관리, SELinux 보안 체계, firewalld 방화벽 관리이며, 기업 서버 및 클라우드 환경에 적합하고 장기 지원과 안정성을 중심으로 패키지가 구성된다.

CentOS Linux는 RHEL 공개 소스를 기반으로 만들어졌던 무료 재빌드 배포판이다. RHEL의 상표와 전용 요소를 제거하고 패키지를 재빌드하는 방식으로 제공되었으며, RHEL과 높은 호환성을 제공했지만 Red Hat의 상용 기술 지원은 포함되지 않았다. CentOS Linux 8은 2021년 12월 31일 지원이 종료되었고, CentOS Linux 7은 2024년 6월 30일 지원이 종료되었으며, 이후 CentOS 프로젝트의 중심은 CentOS Stream으로 이동했다.

Rocky Linux는 CentOS Linux 종료 이후 이를 대체하기 위해 등장한 무료 엔터프라이즈 리눅스 배포판이다. CentOS 공동 창립자인 Gregory Kurtzer가 개발을 주도했으며, RHEL과의 높은 호환성 및 재현성을 목표로 한다. 기존 CentOS/RHEL 환경을 이전하기 쉬워 서버 및 교육 환경에서 많이 사용되지만, 기술적 호환성과 특정 상용 제품의 공식 인증은 별개의 문제이므로 도입 전 벤더 지원 정책을 확인해야 한다.

AlmaLinux는 Rocky Linux와 함께 CentOS Linux 종료 이후 등장한 무료 RHEL 호환 배포판이다. AlmaLinux OS Foundation이 관리하며, 현재는 RHEL과의 1:1 동일성보다 **애플리케이션 바이너리 인터페이스(ABI) 호환성**을 중심으로 개발 방향을 설명한다.

Fedora는 Red Hat이 후원하는 커뮤니티 기반 배포판이다. 최신 커널, 컴파일러, 라이브러리 및 시스템 기술을 빠르게 도입하며, 새로운 기술이 Fedora에서 먼저 적용되고 이후 CentOS Stream과 RHEL 개발 과정에 반영될 수 있다. 서버뿐 아니라 데스크톱 및 개발 환경에서도 널리 사용된다.

배포판을 비교하면 다음과 같다.

| 배포판 | 성격 | 특징 | 주요 위치 |
|---|---|---|---|
| Fedora | 커뮤니티 최신 기술 배포판 | 최신 기능을 빠르게 반영 | RHEL 생태계의 주요 업스트림 |
| CentOS Stream | 지속 배포형 엔터프라이즈 배포판 | 다음 RHEL 변경 사항을 미리 반영 | Fedora와 RHEL 사이 |
| RHEL | 상용 엔터프라이즈 배포판 | 장기 지원, 인증, 기술 지원 | 기업 표준 운영체제 |
| CentOS Linux | 과거의 무료 RHEL 재빌드 | RHEL과 높은 호환성 제공 | 지원 종료 |
| Rocky Linux | 무료 RHEL 호환 배포판 | RHEL과의 높은 호환성 목표 | CentOS Linux 대안 |
| AlmaLinux | 무료 RHEL 호환 배포판 | ABI 호환성 중심 | CentOS Linux 대안 |
| Ubuntu LTS | Debian 계열 엔터프라이즈 배포판 | 클라우드·개발 생태계 및 장기 지원 | 서버·클라우드·개발 환경 |

---

## 2. 표준 설정 템플릿 (Configuration)

> **적용 환경:** RHEL 계열(RHEL·Rocky Linux·AlmaLinux) 및 대부분의 Linux 배포판 공통.

### Step 1. 네트워크 고정 IP 설정(NetworkManager keyfile 방식, RHEL 9 표준)

RHEL 9 계열에서는 NetworkManager를 사용한다. 기존 `ifcfg-*` 형식은 더 이상 기본 방식이 아니며, NetworkManager keyfile 또는 `nmcli` 사용이 권장된다.

먼저 실제 네트워크 인터페이스와 연결 프로파일 이름을 확인한다.

```bash
ip link show
nmcli device status
nmcli connection show
```

다음은 `ens160` 인터페이스의 keyfile을 직접 수정하는 예시이다.

```bash
vi /etc/NetworkManager/system-connections/ens160.nmconnection
```

```properties
[connection]
id=ens160
uuid=5989e9d3-fb91-3508-90e8-2ecc49b5b5d3
type=ethernet
interface-name=ens160

[ethernet]

[ipv4]
method=manual
address1=192.168.10.100/24,192.168.10.2
dns=192.168.10.2;

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
```

- `interface-name`은 `ip link show`로 확인한 실제 NIC 이름을 입력한다.
- `address1` 형식은 `IP 주소/Prefix,Gateway`이다.
- 예제의 UUID를 그대로 복사하지 말고 기존 프로파일의 UUID를 유지해야 한다.
- VMware NAT의 게이트웨이와 DNS 주소는 환경마다 다를 수 있으므로 Virtual Network Editor 또는 실제 네트워크 설정을 확인한다.

keyfile의 소유권과 권한을 설정한다.

```bash
chown root:root /etc/NetworkManager/system-connections/ens160.nmconnection
chmod 600 /etc/NetworkManager/system-connections/ens160.nmconnection
```

설정을 적용한다.

```bash
nmcli connection reload
nmcli connection down ens160
nmcli connection up ens160
```

> **주의:** SSH로 접속한 상태에서 IP 주소를 변경하거나 연결을 내리면 현재 SSH 세션이 끊어진다. 가능하면 VMware 콘솔에서 작업한다.

`nmcli`만 사용하여 고정 IP를 설정할 수도 있다.

```bash
nmcli connection modify ens160 \
  ipv4.method manual \
  ipv4.addresses 192.168.10.100/24 \
  ipv4.gateway 192.168.10.2 \
  ipv4.dns 192.168.10.2

nmcli connection up ens160
```

IPv6를 사용하지 않는 실습 환경이라면 다음과 같이 비활성화할 수 있다.

```bash
nmcli connection modify ens160 ipv6.method disabled
nmcli connection up ens160
```

---

### Step 2. SELinux 비활성화(실습 환경 전용)

> **주의:** 운영 환경에서는 SELinux를 `enforcing`으로 유지하는 것을 권장한다. 다음 설정은 SELinux 동작을 제외한 기초 실습이 필요한 경우에만 사용한다.

#### 현재 상태 확인

```bash
sestatus
getenforce
```

#### 일시적으로 Permissive 모드 전환

재부팅 전까지만 적용된다.

```bash
setenforce 0
getenforce
```

#### 방법 1. 커널 파라미터로 SELinux 완전 비활성화

```bash
grubby --update-kernel ALL --args="selinux=0"
reboot
```

설정 확인:

```bash
getenforce
sestatus
cat /proc/cmdline
```

#### 방법 2. SELinux 설정 파일 변경

```bash
vi /etc/selinux/config
```

```properties
SELINUX=disabled
SELINUXTYPE=targeted
```

변경 후 재부팅한다.

```bash
reboot
```

> **참고:** Rocky Linux 9 계열에서 SELinux를 완전히 비활성화하려면 커널 파라미터 `selinux=0` 방식이 명확하다.

#### SELinux 다시 활성화

먼저 커널 파라미터에서 `selinux=0`을 제거한다.

```bash
grubby --update-kernel ALL --remove-args="selinux=0"
```

설정 파일을 수정한다.

```bash
vi /etc/selinux/config
```

```properties
SELINUX=enforcing
SELINUXTYPE=targeted
```

파일 시스템 재레이블링을 예약한다.

```bash
touch /.autorelabel
reboot
```

> **참고:** 파일 수가 많으면 재레이블링에 오랜 시간이 걸릴 수 있다.

---

### Step 3. SSH Root 원격 접속 허용(격리된 실습 환경 전용)

> **보안 경고:** 운영 환경에서는 Root 계정의 직접 SSH 로그인과 비밀번호 인증을 허용하지 않는 것이 원칙이다. 일반 사용자로 접속한 뒤 `sudo` 또는 `su`를 사용하고, SSH 공개키 인증을 적용하는 것을 권장한다.

실습 환경에서 Root SSH 접속을 허용하려면 다음 파일을 수정한다.

```bash
vi /etc/ssh/sshd_config
```

```properties
PermitRootLogin yes
PasswordAuthentication yes
PubkeyAuthentication yes
Subsystem sftp /usr/libexec/openssh/sftp-server
```

설정 파일 안에 다음과 같은 설명용 문자를 직접 입력하면 안 된다.

```text
PermitRootLogin yes <--- Root 로그인 허용
```

`<--- Root 로그인 허용` 부분은 SSH 설정 문법이 아니므로 설정 파일에는 다음과 같이 작성해야 한다.

```properties
# Root SSH 로그인 허용 - 실습 환경 전용
PermitRootLogin yes
```

RHEL 9 계열은 다음 디렉터리의 설정 파일도 함께 읽는다.

```text
/etc/ssh/sshd_config.d/*.conf
```

중복 설정을 확인한다.

```bash
grep -RniE '^[[:space:]]*(PermitRootLogin|PasswordAuthentication)' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

실제로 적용될 최종 설정값을 확인한다.

```bash
sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

설정 문법을 검사한다.

```bash
sshd -t
```

오류가 없으면 설정을 다시 불러온다.

```bash
systemctl reload sshd
systemctl enable --now sshd
```

방화벽에서 SSH 서비스를 허용한다.

```bash
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
firewall-cmd --list-services
```

Root 계정에 비밀번호가 설정되어 있는지도 확인한다.

```bash
passwd -S root
```

필요한 경우 안전한 비밀번호를 설정한다.

```bash
passwd root
```

> **참고:** 문서나 쉘 히스토리에 실제 비밀번호를 기록하지 않는다. `admin1234`처럼 예측 가능한 비밀번호는 실습 환경에서도 사용하지 않는 것이 좋다.

#### 운영 환경 권장 설정

```properties
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

- SSH 포트를 변경하는 것은 자동화된 접속 시도의 양을 줄일 수 있지만, 본질적인 보안 대책은 아니다.
- 공개키 인증, Root 로그인 차단, 방화벽 접근 제어, VPN 또는 Bastion Host, 로그인 로그 모니터링을 함께 적용해야 한다.

---

### Step 4. 시스템 종료/재부팅 표준 명령어

#### 서버 즉시 종료

```bash
shutdown now
```

또는 다음 명령을 사용할 수 있다.

```bash
shutdown -P now
systemctl poweroff
halt -p
```

#### 10분 뒤 서버 종료

```bash
shutdown -P +10
```

접속 중인 사용자에게 다음과 같은 메시지가 전송된다.

```text
Broadcast message from root@localhost:

The system will power off at the scheduled time.
```

#### 예약 종료 취소

```bash
shutdown -c
```

취소 시 접속 중인 사용자에게 종료 예약이 취소되었다는 메시지가 전송된다.

#### 실제 종료 없이 경고 메시지만 전송

```bash
shutdown -k +30
```

- `-k` 옵션은 실제 종료를 수행하지 않고 사용자에게 경고 메시지만 전송한다.
- 단순 공지 목적이라면 `wall` 명령을 사용할 수도 있다.

```bash
echo "30분 후 서버 점검이 예정되어 있습니다." | wall
```

#### 지정된 시간에 서버 재부팅

```bash
shutdown -r 23:00
```

- `-r`은 종료가 아니라 **재부팅(Reboot)** 옵션이다.
- 예약된 재부팅은 다음 명령으로 취소한다.

```bash
shutdown -c
```

#### 주요 명령 정리

| 명령어 | 설명 |
|---|---|
| `shutdown now` | 시스템 즉시 종료 |
| `shutdown -P +10` | 10분 뒤 전원 종료 |
| `shutdown -r 23:00` | 23시에 시스템 재부팅 |
| `shutdown -k +30` | 실제 종료 없이 경고만 전송 |
| `shutdown -c` | 예약된 종료 또는 재부팅 취소 |
| `systemctl poweroff` | 시스템 전원 종료 |
| `systemctl reboot` | 시스템 재부팅 |
| `halt -p` | 시스템을 정지하고 전원 종료 |

---

### Step 5. 초기 구축 후 필수 패키지 및 업데이트

전체 패키지를 업데이트한다.

```bash
dnf -y update
```

기초 관리 도구를 설치한다.

```bash
dnf -y install bind-utils net-tools vim wget curl bash-completion
```

- `bind-utils`
  - `dig`
  - `nslookup`
  - `host`

- `net-tools`
  - `ifconfig`
  - `netstat`
  - `route`
  - 기존 명령어 실습용으로 사용할 수 있지만 신규 운영 절차에서는 `ip`, `ss` 명령을 우선 사용한다.

- `vim`
  - 텍스트 편집기

- `wget`, `curl`
  - HTTP/HTTPS 파일 요청 및 다운로드

업데이트 후 재부팅 필요 여부를 확인한다.

```bash
dnf -y install dnf-utils
needs-restarting -r
```

커널 또는 주요 시스템 구성 요소가 업데이트되었다면 재부팅한다.

```bash
reboot
```

---

### Step 6. Server-B 초기 설정

Server-B 콘솔에서 Root 계정으로 로그인한다.

```text
localhost login: root
Password: <ROOT_PASSWORD>
```

필수 패키지를 설치한다.

```bash
dnf -y install bind-utils net-tools vim wget curl
```

네트워크 정보를 확인한다.

```bash
ip addr
ip route
```

SELinux 상태를 확인한다.

```bash
sestatus
getenforce
```

실습 환경에서만 SELinux를 비활성화한다.

```bash
grubby --update-kernel ALL --args="selinux=0"
```

```bash
vi /etc/selinux/config
```

```properties
SELINUX=disabled
SELINUXTYPE=targeted
```

패키지를 업데이트한 뒤 재부팅한다.

```bash
dnf -y update
reboot
```

재부팅 후 상태를 확인한다.

```bash
getenforce
```

예상 결과:

```text
Disabled
```

---

### Step 7. Client-L 초기 설정

현재 SELinux 상태를 확인한다.

```bash
sestatus
getenforce
```

실습 환경에서만 SELinux를 비활성화한다.

```bash
grubby --update-kernel ALL --args="selinux=0"
```

```bash
vi /etc/selinux/config
```

```properties
SELINUX=disabled
SELINUXTYPE=targeted
```

패키지를 업데이트한다.

```bash
dnf -y update
```

재부팅한다.

```bash
reboot
```

재부팅 후 SELinux 상태를 확인한다.

```bash
getenforce
```

예상 결과:

```text
Disabled
```

---

### Step 8. 가상 콘솔 접속

- Rocky Linux를 포함한 대부분의 리눅스는 여러 개의 가상 콘솔(TTY)을 제공한다.
- GUI에 문제가 발생했거나 텍스트 모드에서 시스템을 점검해야 할 때 사용할 수 있다.

#### 가상 콘솔 전환

```text
Ctrl + Alt + F1
Ctrl + Alt + F2
Ctrl + Alt + F3
Ctrl + Alt + F4
Ctrl + Alt + F5
Ctrl + Alt + F6
```

- GUI가 실행되는 TTY 번호는 배포판 및 현재 로그인 상태에 따라 달라질 수 있다.
- Rocky Linux 9의 GNOME 환경에서는 GUI가 `tty1` 또는 `tty2`에서 실행될 수 있으므로 "GUI는 항상 F1"이라고 단정하면 안 된다.
- 텍스트 콘솔로 이동하면 로그인 프롬프트가 출력된다.
- 현재 콘솔 번호는 다음 명령으로 확인할 수 있다.

```bash
tty
```

예시:

```text
/dev/tty3
```

현재 기본 부팅 모드를 확인한다.

```bash
systemctl get-default
```

GUI 모드:

```text
graphical.target
```

텍스트 모드:

```text
multi-user.target
```

---

## 3. 검증 및 트러블슈팅 (Verification & Troubleshooting)

### 3-1. 필수 검증 명령어

#### 네트워크 검증

```bash
# 전체 인터페이스 확인
ip addr

# ens160 인터페이스 확인
ip addr show ens160

# 링크 상태 확인
ip link show ens160

# 라우팅 테이블 및 기본 게이트웨이 확인
ip route show

# NetworkManager 장치 상태 확인
nmcli device status

# NetworkManager 연결 프로파일 확인
nmcli connection show

# ens160 프로파일 상세 확인
nmcli connection show ens160

# 게이트웨이 통신 확인
ping -c 3 192.168.10.2

# 외부 IP 통신 확인
ping -c 3 8.8.8.8

# DNS 이름 해석 확인
dig naver.com
host naver.com
```

#### SELinux 검증

```bash
sestatus
getenforce
cat /proc/cmdline
```

#### SSH 검증

```bash
# SSH 서비스 상태 확인
systemctl status sshd

# 부팅 시 자동 시작 여부 확인
systemctl is-enabled sshd

# 설정 문법 검사
sshd -t

# 최종 적용 설정 확인
sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'

# 22번 포트 Listen 확인
ss -tlnp | grep ':22'
```

#### 방화벽 검증

```bash
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --list-services
```

#### 시스템 정보 확인

```bash
# 커널 버전
uname -r

# 운영체제 정보
cat /etc/os-release

# 호스트 이름
hostnamectl

# CPU 정보
lscpu

# 메모리 정보
free -h

# 디스크 사용량
df -h

# 블록 장치 정보
lsblk
```

---

### 3-2. 대표 장애 시나리오 및 트러블슈팅

#### 시나리오 1. `.nmconnection` 파일을 수정했는데 IP가 반영되지 않는다

**증상:**  
IP 주소를 `192.168.10.100`으로 변경했지만 `ip addr`에서 기존 DHCP 주소인 `192.168.10.128`이 계속 출력된다.

##### 1단계. 실제 인터페이스와 프로파일 이름 확인

```bash
nmcli device status
nmcli connection show
ip link show
```

인터페이스 이름과 연결 프로파일 이름은 서로 다를 수 있다.

##### 2단계. keyfile 권한 확인

```bash
ls -l /etc/NetworkManager/system-connections/ens160.nmconnection
```

필요한 경우 권한을 수정한다.

```bash
chown root:root /etc/NetworkManager/system-connections/ens160.nmconnection
chmod 600 /etc/NetworkManager/system-connections/ens160.nmconnection
```

##### 3단계. keyfile 문법 확인

```bash
cat /etc/NetworkManager/system-connections/ens160.nmconnection
```

다음 항목을 확인한다.

```properties
[ipv4]
method=manual
address1=192.168.10.100/24,192.168.10.2
dns=192.168.10.2;
```

- IP 주소와 Prefix가 올바른지 확인
- 게이트웨이 주소가 VMware NAT 환경과 일치하는지 확인
- 프로파일의 `interface-name`이 실제 NIC 이름과 일치하는지 확인
- 기존 UUID를 임의로 변경하지 않았는지 확인

##### 4단계. NetworkManager 설정 다시 로드

```bash
nmcli connection reload
nmcli connection down ens160
nmcli connection up ens160
```

SSH 세션에서는 연결이 끊어질 수 있으므로 VMware 콘솔에서 실행하는 것이 안전하다.

##### 5단계. 적용 결과 확인

```bash
ip -4 addr show ens160
ip route show
nmcli device show ens160
```

##### 6단계. NetworkManager 로그 확인

```bash
journalctl -u NetworkManager -n 100 --no-pager
```

실시간 확인:

```bash
journalctl -fu NetworkManager
```

##### 7단계. 중복 프로파일 확인

```bash
nmcli connection show
ls -l /etc/NetworkManager/system-connections/
```

같은 NIC를 사용하는 프로파일이 여러 개라면 잘못된 프로파일이 활성화될 수 있다.

##### 8단계. 최후의 확인 방법

```bash
reboot
```

재부팅 후 다음 명령으로 확인한다.

```bash
ip -4 addr show ens160
```

---

#### 시나리오 2. `PermitRootLogin yes`를 설정했는데도 Root SSH 접속이 거부된다

**증상:**

```text
Permission denied
```

##### 1단계. 최종 적용 설정 확인

```bash
sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication'
```

예상 값:

```text
permitrootlogin yes
passwordauthentication yes
```

##### 2단계. 중복 설정 확인

```bash
grep -RniE '^[[:space:]]*(PermitRootLogin|PasswordAuthentication)' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

RHEL 9 계열은 `/etc/ssh/sshd_config.d/*.conf` 파일을 함께 읽으므로 메인 설정 파일만 확인해서는 안 된다.

##### 3단계. 설정 문법 검증

```bash
sshd -t
```

오류 메시지가 없다면 문법상 정상이다.

##### 4단계. SSH 서비스 설정 다시 로드

```bash
systemctl reload sshd
systemctl status sshd
```

##### 5단계. Root 계정 상태 확인

```bash
passwd -S root
```

계정이 잠겨 있다면 상태를 확인하고 실습 목적에 한해서 비밀번호를 다시 설정한다.

```bash
passwd root
```

##### 6단계. 방화벽 확인

```bash
firewall-cmd --list-all
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

##### 7단계. SSH Listen 상태 확인

```bash
ss -tlnp | grep ':22'
```

##### 8단계. 인증 로그 확인

```bash
tail -f /var/log/secure
```

또는 다음 명령을 사용할 수 있다.

```bash
journalctl -fu sshd
```

##### 9단계. 클라이언트에서 상세 로그 확인

```bash
ssh -vvv root@192.168.10.100
```

> **주의:** 운영 환경에서는 문제 해결 후 `PermitRootLogin no`, `PasswordAuthentication no`로 복구하고 공개키 기반 로그인을 사용하는 것이 좋다.

---

#### 시나리오 3. 고정 IP 설정 후 외부 통신이 되지 않는다

##### 1단계. IP 주소 확인

```bash
ip -4 addr show ens160
```

##### 2단계. 기본 게이트웨이 확인

```bash
ip route show
```

정상 예시:

```text
default via 192.168.10.2 dev ens160
192.168.10.0/24 dev ens160 proto kernel scope link src 192.168.10.100
```

##### 3단계. 게이트웨이 통신 확인

```bash
ping -c 3 192.168.10.2
```

##### 4단계. 외부 IP 통신 확인

```bash
ping -c 3 8.8.8.8
```

##### 5단계. DNS 확인

```bash
cat /etc/resolv.conf
nmcli device show ens160 | grep -i dns
dig naver.com
```

- 게이트웨이 통신부터 실패하면 IP, Prefix, VMware NAT 설정을 확인한다.
- 외부 IP는 통신되지만 도메인만 실패하면 DNS 설정을 확인한다.

---

#### 시나리오 4. SELinux를 비활성화했는데 `getenforce`가 계속 Enforcing으로 출력된다

##### 1단계. 설정 파일 확인

```bash
grep -E '^SELINUX=' /etc/selinux/config
```

##### 2단계. 커널 파라미터 확인

```bash
cat /proc/cmdline
```

##### 3단계. grubby 설정 확인

```bash
grubby --info=ALL | grep args
```

##### 4단계. 커널 파라미터 추가

```bash
grubby --update-kernel ALL --args="selinux=0"
```

##### 5단계. 재부팅

```bash
reboot
```

##### 6단계. 상태 확인

```bash
getenforce
sestatus
```

예상 결과:

```text
Disabled
```

---

> **참고:**  **아키텍트 Tip:** 초기 구축과 검증이 완료되면 VMware Workstation Pro에서 스냅샷을 생성한다.

권장 스냅샷 이름:

```text
Initial-Setup-Done
```

스냅샷 생성 전에는 다음 사항을 확인한다.

- 고정 IP 적용 여부
- SSH 접속 여부
- SELinux 실습 상태
- 방화벽 상태
- 패키지 업데이트 완료 여부
- Server-A, Server-B, Client-L 간 통신 여부
- 실제 비밀번호나 인증정보가 문서 또는 스크립트에 저장되지 않았는지 여부

스냅샷은 백업을 완전히 대체하지 않는다. 중요한 가상머신은 스냅샷뿐 아니라 가상머신 파일 또는 데이터를 별도로 백업해야 한다.
