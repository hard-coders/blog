---
title: KVM & 쿠버네티스 시작하기
tags: 'kvm, vm, 가상머신, 쿠버네티스, kubernetes'
date: 2019-07-25 14:22:38
---

이 글은 [Get started with KVM & Kubernets](https://blog.alexellis.io/kvm-kubernetes-primer/)를 번역한 포스트입니다.

---

이 포스트는 리눅스의 가상화 방식인 KVM을 소개합니다. 이 세상에는 상용급(production-grade) 실험실, 데이터 센터, 사설 클라우드를 구축하는데 사용할 수있는 다양한 도구가 있습니다. 툴링, 서비스, 지원은 제품이 엔터프라이즈, 소규모 비즈니스 또는 오픈 소스를 대상으로 하는지 여부에 따라 달라지죠.

물론 이외에도 다른 하이퍼바이저나 가상머신 소프트웨어가 있습니다. 마이크로소프트의 HyperV, VMWare의 VSphere/ESXi, bhyve(FreeNAS/BSD 적용) 그리고 방금 말한 bhyve 기반으로 하는 도커의 hyperkit이 있습니다.

이 포스트는 학습 및 탐험을 고무코자 KVM을 사용하여 집에서 가상머신(이하 VM) 클러스트러를 만드는 방법에 촛점을 둡니다. 우리는 우분투 리눅스를 이용해 KVM 가상머신 2개를 세팅하고, 쿠버네티스를 설치한뒤 OpenFaaS를 쿠버네티스 꼭대기에서 실행할 겁니다.

## 소개

KVM은 [리눅스를 위한 하이퍼바이저](https://ko.wikipedia.org/wiki/%EC%BB%A4%EB%84%90_%EA%B8%B0%EB%B0%98_%EA%B0%80%EC%83%81_%EB%A8%B8%EC%8B%A0)입니다. 이름이 같다고 [keyboard, video and mouse](https://ko.wikipedia.org/wiki/KVM_%EC%8A%A4%EC%9C%84%EC%B9%98) 멀티플렉서와 혼동하지 마세요. 도커의 대장(Captain)이며 오픈 소스 개발자로써, 매일 매일을 컨테이너 작업에 많은 시간을 할애했습니다. 가끔은 UI에서 클릭 몇 번으로 VM을 "클라우드"에 프로비저닝하거나 원격 API를 호출하지만, 어떤 하이퍼바이저가 사용되는지와 같은 세밀한 부분에 대해서는 거의 관심이 없습니다.

![kvm](https://upload.wikimedia.org/wikipedia/commons/7/70/Kvmbanner-logo2_1.png)

우리는 이 포스트에서 우리만의 VM을 만들거고, 필요한 것들은 다음과 같습니다.

- 가상화 가능한 호스트
- 우분투 리눅스 16.04 LTS
- 이더넷 연결
- 8~16 GB 메모리
- 100~500 GB의 여유 공간(추천사항)

> 우분투 17.10 역시 KVM을 실행할 수 있지만 네트워크 설정이 바뀌었고 [`netplan`](https://wiki.ubuntu.com/Netplan)을 사용합니다. 가이드를 따르는 가장 쉬운 방법은 호스트에 16.04를 설치하거나 사용하는 겁니다.

## 튜토리얼

이 튜토리얼은 네 가지 파트로 나누어져 있습니다.

1. KVM을 위한 호스트 설정
1. 신 2개 생성
1. 쿠버네티스 설치 및 애플리케이션 배포
1. 간단한 리뷰

### 1.0 `kvm` CLI 컴포넌트 설치하기

```bash
$ sudo apt-get install -qy \
    qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```

`kvm-ok` 명령으로 `kvm`이 작동하는지 확인하세요. 이 명령은 `/dev/kvm`에 있는 새 장치를 확인합니다.

```bash
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

### 1.1 호스트 네트워킹

기본적으로 KVM은 호스트의 네트워킹에 NAT를 사용합니다. 이는 인터넷에 접속할 수는 있지만 다른 네트워크로 쉽게 액세스 할 수 없음을 뜻합니다. 때문에 우리는 브릿지를 통해 네트워킹을 변경하는데 필요한 서비스에 엑세스할 수 있는 클러스터를 만들어야 합니다.

> 만약 VirtualBox 또는 VMWare Fusion을 사용 한적이 있는 경우 VM 네트워킹을 위한 브릿지나 NAT라는 용어에 친숙할 겁니다. 브릿지 네트워킹은 VM이 네트워크에 직접 연결되어 있는 것처럼 작동하며 라우터에서 DHCP IP 주소를 가져옵니다.

역주) [Bridging](https://en.wikipedia.org/wiki/Bridging_(networking)) 참고

이제부터 사용자급(consumer-grade) 라우터가 있는 홈 네트워크를 사용하고 `192.168.0.0/24` 서브넷으로 작업한다고 가정하겠습니다. 필요한 경우 IP를 바꾸세요.

- 이제 호스트의 `브릿지` 인터페이스를 세팅합니다

`ifconfig`에 나오는 이더넷 어댑터 `eno1`을 바꿉시다.

`/etc/network/interfaces`

```etc
# The primary network interface
#auto eno1
#iface eno1 inet dhcp

auto br0
iface br0 inet dhcp
        bridge_ports eno1
        bridge_stp off
        bridge_fd 0
        bridge_maxwait 0
```

KVM 호스트를 위해 동적 IP를 세팅했지만, 정적 IP로 세팅해도 됩니다.

- IP 포워딩과 브릿지 설정을 세팅합니다.

`/etc/sysctl.conf`를 수정하고 다음을 이어서 작성합니다:

```ini
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
net.ipv4.ip_forward = 1
```

VM을 재시작합니다.

`ifconfig`에서 `br0` 장치가 보여야 합니다. 이 장치는 IP 주소를 가지지만 `eno1`은 그렇지 않을겁니다.

### 1.2 CLI

설치과정에서 새로운 CLI가 추가 됩니다:

- `virsh` - 관리를 위해 사용, [libvirt](https://en.wikipedia.org/wiki/Libvirt)를 이용해 VM 생성 및 상세보기
- `virt-install` - 새 VM 부트스트랩 및 설치를 위해 사용

`virsh`는 모든 종류의 편리한 명령어가 있고 쉘 그자체로 동작할 수 있습니다. 가장 유용한 명령어는 실행 중이거나 정지한 VM을 보여주는 `virsh list --all` 입니다.

- VM 부팅 - `virsh start <vm>`
- VM 정지 - `virsh shutdown <vm>`
- VM 연기(suspend) - `virsh suspend <vm>`
- VM 삭제 - `virsh destroy <vm>` 및 `virsh undefine <vm>`

[여기](https://libvirt.org/sources/virshcmdref/html/)에서 사용가능한 virsh의 CLI 레퍼런스를 확인할 수 있습니다.

### 2.0 첫 VM 만들기

`virt-installer` 명령으로 우분투 VM을 만들겁니다. 터미널 세션을 사용하는 경우 VNC 또는 원격 데스크톱 툴을 사용하면 안됩니다.

인터넷으로 우분투 리눅스를 설치하기 때문에 ISO 이미지를 미리 준비하지 않아도 됩니다. 만약 오프라인으로 설치를 하고 싶으면 `--location` 대신 `--cdrom` 플래그를 추가하면 됩니다.

다음과 같이 `create-vm.sh`를 저장합니다:

```sh
#!/bin/sh

if [ -z "$1" ] ;
then
 echo Specify a virtual-machine name.
 exit 1
fi

sudo virt-install \
--name $1 \
--ram 4096 \
--disk path=/var/lib/libvirt/images/$1.img,size=30 \
--vcpus 2 \
--os-type linux \
--os-variant ubuntu16.04 \
--network bridge:br0,model=virtio \
--graphics none \
--console pty,target_type=serial \
--location 'http://gb.archive.ubuntu.com/ubuntu/dists/xenial/main/installer-amd64/' \
--extra-args 'console=ttyS0,115200n8 serial'
```

원한다면 `ram`이나 `vcpus`와 같은 파라미터 값을 조정해도 됩니다. NAS나 외장 하드디스크가 있다면 NFS 마운트 포인트 역시 다른 경로로 설정해도 됩니다. `size=30` 파라미터는 `--disk`의 기가바이트를 상세설정합니다.

우분투 Xenial 서버 에디션은 `--location` 플래그에서 선택됩니다.

저는 이 시점에 `tmux`을 실행하는걸 좋아합니다. 이렇게 하면 접속을 끊고 다시 접속해도 되고, 여러개의 설치를 동시에 할 수 있기 때문입니다.

스크립트에 실행권한을 주고 인자로 머신 이름을 넣어 실행하세요.

```sh
$ chmod +x create-vm.sh
$ ./create-vm.sh k8s-master
```

이제 아래에 있는 모든 지시사항을 따르세요.

- VM이 네트워크 주소를 가져와야 합니다.

![instruction](https://blog.alexellis.io/content/images/2018/02/dhcp.png)

- `Hostname`을 `k8s-master`로 합니다.

![instruction](https://blog.alexellis.io/content/images/2018/02/hostname.png)

- 유저 이름을 `ubuntu`로 합니다.

![instruction](https://blog.alexellis.io/content/images/2018/02/username.png)

- `OpenSSH server` 패키지를 선택합니다.(중요)

![instruction](https://blog.alexellis.io/content/images/2018/02/openssh.png)

- 인스톨러가 재부팅하게 둡니다.

`nmap` 명령으로 네트워크의 새 IP 주소를 스캔합니다:

```sh
$ sudo nmap -sP 192.168.0.0/24
```

> 노트: 여기서 `sudo`는 꼭 필요하므로 생략하지 마세요

다음과 같이 `QEMU virtual NIC` 제조업체에 새 항목이 표시되어야 합니다.

```etc
Nmap scan report for 192.168.0.66
Host is up (0.00049s latency).
MAC Address: 52:54:00:07:79:AF (QEMU virtual NIC)
```

IP 주소와 설치과정에서 만든 사용자 정보를 이용해 `ssh`로 로그인합니다:

```sh
$ ssh ubuntu@192.168.0.66

Welcome to Ubuntu 17.10.1 LTS (GNU/Linux 4.4.0-112-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Fri Feb  9 21:39:27 2018 from 192.168.0.22

ubuntu@k8s-master:~$

```

- 쿠버네티스는 스왑 공간을 끄도록 요구합니다.

`/etc/fstab`을 수정합니다:

```etc
# swap was on /dev/vda5 during installation
# UUID=a5f1c243-37a7-440c-88e0-d0fe71f05ce7 none            swap    sw              0       0
```

위와 같이 `swap`하는 행을 주석처리 합니다.

- 선택사항 - 고정 IP 설정

`/etc/network/interfaces`를 수정해서 VM의 고정 IP를 선택적으로 설정 할 수 있습니다. 설정한 IP와 홈 네트워크의 서브넷과 일치해야 합니다.

> IP 범위 `192.168.0.0/24`를 쓰는 경우 `192.168.0.200` 등과 같이
> 네트워크의 다른 장치에 할당하지 않을 IP 주소를 선택하세요

- 원하는 추가 가능한 패키지 추가

선택한 패키지에 따라 `apt install`을 통하여 `git`이나 `curl`과 같은 패키지를 설치하고 싶을수도 있습니다.

이게 전부입니다. 이제 우리 네트워크에 자신의 IP 주소를 갖는 첫번째 가상머신이 생겼습니다.

### 2.1 다음 머신 만들기

머신의과 디스크 이미지의 이름은 매번 실행할때마다 유일하게 하는 것이 좋습니다. `--name`과 `--disk`를 통해 설정합니다.

이제 우리가 만든 프로비저닝 스크립트를 `k8s-1` 또는 `k8s-2` 등과 같이 새 호스트 이름으로 실행합니다.

```sh
$ ./create-vm.sh k8s-1
```

인스톨러를 통해 [2.0 첫 VM 만들기](###-2.0-첫-VM-만들기)에서와 같은 방식으로 실행한 다음 IP 주소를 찾아서 시스템이 시작되었는지 확인하세요. `sudo swapoff -a`를 실행하고 `/etc/fstab`을 편집하세요. 이 지침사항은 생략하지 마세요.

### 3.0 쿠버네티스 컴포넌트 설치하기

이제 두 가지 선택사항이 있습니다. 저의 "인스턴트 가이드"를 따르거나 `kubeadm`을 경험했다면 아래 나열한 쉘 단계를 사용할 수 있습니다.

![k8s](https://upload.wikimedia.org/wikipedia/commons/thumb/6/67/Kubernetes_logo.svg/1024px-Kubernetes_logo.svg.png)

- 가이드:

이제 제 가이드 [Your instant Kubernetes cluster](https://blog.alexellis.io/your-instant-kubernetes-cluster/)를 참고해 쿠버네티스를 설치합시다.

`kubeadm init` 단계에서는 마스터로, `kubeadm join` 단계에서는 워커로 `ssh` 합니다.

- 가장 빠른 방법(쿠버네티스를 사용해본 경험이 있다면):

아래를 입력해 컴포넌트를 설치힙나다.

```sh
$ curl -sL https://gist.githubusercontent.com/alexellis/e8bbec45c75ea38da5547746c0ca4b0c/raw/23fc4cd13910eac646b13c4f8812bab3eeebab4c/configure.sh | sudo sh
```

- 마스터 초기화

마스터에 로그인하고 `sudo kubeadm init`을 실행하는 방법에 대해서는 Kubernetes 가이드를 참조하세요:

```sh
$ ubuntu@k8s-master:~$ sudo kubeadm init
```

이제 Weaveworks 네트워크를 적용합니다:

```sh
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

- 첫 워커 노드 조인하기

워커에 로그인하고 `sudo kubeadm join`을 이용해 클러스터에 조인하세요.

> 노트: 좀 더 자세한 쿠버네티스 설정은 위에 링크한 가이드를 읽으세요

### 3.1 애플리케이션 배포하기

이제 OpenFaaS - Serverless Functions Made Simple 애플리케이션을 배포할 수 있습니다. OpenFaaS는 내장 UI 대시보드가 있고 사용하기 쉽습니다.

![openfass](https://raw.githubusercontent.com/openfaas/media/master/OpenFaaS_logo_stacked_opaque.png)

마스터에서 아래를 실행합니다:

```sh
$ apt install git
$ git clone https://github.com/openfaas/faas-netes && cd faas-netes \
  && kubectl apply -f namespaces.yml,./yaml
```

이제 워커 노드에서 예약되고(scheduled) 도커 허브에서 pull한 서비스가 생성되는 것을 감시할 수 있습니다:

```sh
$ watch 'kubectl get pods --all-namespaces'
```

You're watching out for this on each line:

```etc
READY     STATUS    RESTARTS
1/1       Running   0
```

이제 OpenFaas Function Store에서 함수를 배포할수 있고, 브라우저에서 실행해 볼 수 있습니다. k8s-master 노드의 `31112` 포트를 열기만 하면 됩니다.

http://192.168.0.66:31112

- Function Store 아이콘을 클릭하세요:

![func_store](https://blog.alexellis.io/content/images/2018/01/Screen-Shot-2018-01-07-at-10.00.37.png)

- _Figlet_ 함수를 선택하세요:

![figlet](https://blog.alexellis.io/content/images/2018/01/Screen-Shot-2018-01-07-at-10.00.51.png)

- 적당한 텍스트를 _Request body_ 에 넣고 _Invoke_ 를 눌러 `figlet` 아스키 로고가 생성되는걸 보세요

![ascii_figlet](https://blog.alexellis.io/content/images/2018/01/Screen-Shot-2018-01-07-at-10.14.33.png)

### 4.0 돌아보기 및 주의사항

아래를 통해 우리가 한 것들을 되돌아봅시다.

- KVM과 CLI 툴 설치
- VM이 우리 라우터의 IP 주소를 가져오는 것을 허용코자 브릿지 인터페이스 생성
- 우분투에서 VM 두 개 설치
- `kubeadm`으로 쿠버네티스 클러스터 생성
- OpenFaaS 애플리케이션 배포
- 서버리스 함수 실행

여기까지는 그저 KVM 겉핥기일 뿐입니다. 이외의 할수 있는 일이 무궁무진하고, 우분투 뿐만 아니라 CentOS, SuSE 리눅스나 심지어 윈도우즈를 설치해서 사용할 수도 있습니다.

여기 몇 가지 주의사항이 있습니다:

- 우리는 마스터와 워커에서 동적 IP를 사용했습니다.

VM을 재시작하거나 다시 켤 때, 새 IP 주소를 가지고 시작할겁니다. 이는 양 노드 모두 `kubeadm`을 다시 실행해야 함을 뜻합니다.

- 우리는 인스톨러를 사용했습니다.

첫 VM을 템플릿처럼 복제하지 않았던 것이 좀 의아할 수도 있습니다. 머신ID, 호스트 이름, ssh 호스트 키 등을 바꾸더라도 복제한 머신에서는 쿠버네티스가 제대로 동작하지 않는걸 발견했기 때문입니다. 여러분과는 다를수 있습니다. 심지어 `virt-sysprep`라는 복제한 리눅스 VM을 "제거" 하기 위한 전문 도구에서도 같은 이슈가 발생하는걸 확인했습니다.

- KVM VM을 관리하기 위한 다른 방법도 존재합니다.

KVM을 UI로 관리할 수 있고 꽤나 유명한 방법입니다. 특히 VMWare의 vSphere/vCenter와 같은 작업에서 도구를 사용한적이 있다면 더 그렇습니다.

우분투는 오라클 VirtualBox에서 사용하는 UI와 같은 경험을 선사해줄 수 있는 그래픽 도구  [virt-manager](https://help.ubuntu.com/community/KVM/VirtManager)를 제공합니다.

![vmm](http://waste.mandragor.org/virt-manager-screenshot.png)

X11이나 VNC를 설치하고 싶지 않아서 경량(lean) 헤드리스 서버를 사용했습니다.

역주) [X11](https://ko.wikipedia.org/wiki/X_%EC%9C%88%EB%8F%84_%EC%8B%9C%EC%8A%A4%ED%85%9C), [VNC](https://ko.wikipedia.org/wiki/VNC) 참고

[Kimchi](https://github.com/kimchi-project/kimchi)는 모니터 연결, X11/VNC 설치 사이에 좋은 절충안이 될 수 있는 웹 UI입니다.
~~역주) 와..이런 코멘트 달지 않는 주의인데 김치에서 웃었습니다~~

![kimchi](https://github.com/kimchi-project/kimchi/raw/master/docs/kimchi-templates.png)

- 라즈베리 파이에서도 돌아갈까요?

제 생각에는 가능할것 같지만, 잘 알지도 못하고 저전력 장치에서 가상하는 것을 추천하고 싶지도 않습니다. 라즈베리 파이에서 다중 노드 클러스터를 만드는 방법은 아래에 링크를 달아놓았습니다.

## Wrapping up

In this blog post I set out to show you how to create and run a Kubernetes cluster on a single host and deploy an application. There are many ways to run Virtual Machines in your home-lab ranging from commercial solutions from VMware to built-in virtualization on Linux with KVM.

If you need a cluster fast then public cloud is hard to beat - but for those of you who like to know how things work and to tinker with CLIs I hope this has been a good primer for KVM. Go and experiment and let me know what you've learnt on the blog comments or [via Twitter @alexellisuk](https://twitter.com/alexellisuk).

My related articles:

- [Instant Kubenetes cluster](https://blog.alexellis.io/your-instant-kubernetes-cluster/)
- [You need to know Docker Swarm vs Kubernetes](https://blog.alexellis.io/you-need-to-know-kubernetes-and-swarm/)
- [Kubernetes home-lab with your Raspberry Pis](https://blog.alexellis.io/serverless-kubernetes-on-raspberry-pi/)
