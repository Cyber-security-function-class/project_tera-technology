# VXLAN을 이용한 PXE 구축

# 프로젝트 개요

### 팀원 역할 분담

민제민 : VXLAN, DNS, IIS

허린 : VXLAN, PXE, DHCP, TFTP

### 시나리오

민제민과 허린은 Tera Technology의 인프라 담당을 하게 되었다.

Tera Technology의 인트라 넷은 각 지점 끝 라우터에 연결되어 있는 End Device 들을 묶은 의사 회선(pseudowire)으로 연결 되어 있고, 의사 회선에는 DHCP가 설정 되어 있으며, Tera Technology의 회사 사람들은 PXE를 사용하여 부팅 이미지를 서버에서 받아 사용한다. (PC에 하드디스크가 없음)

Tera Technology의 회사 사람들은 서울 본점과 부산 지점이 의사 회선 망에 연결되어 있어 각 지점 사람들 끼리의 통신이 자유롭다. 

PXE Server의 상태를 확인하기 위해 웹 서버를 구측하였고, 서버의 접근성을 향상시키기 위해 DNS 서버를 구축하여 PXE Server의 도메인을 설정해주었다. 

마지막으로, Tera Technology 회사는 현재 1급 기밀 프로젝트를 진행 하고 있기 때문에, WAN 과의 연결이 제한되어 있다. (Default Gateway가 없음)

# 토플로지

![Untitled](https://user-images.githubusercontent.com/66944342/144621652-ba62b09d-fd10-4f4e-909c-d24b91cf3e0f.png)

# 네트워크 정보

> OpenWRT는 다양한 임베디드 기기를 위한 리눅스 배포판이다.

> 커스텀 펌웨어로 개발이 시작되었다가 점차 지원 대상이 확대되어 다양한 인터넷 공유기를 지원하는 완전한 리눅스 배포판이 되었다.

> OpenWRT를 사용하는 이유는 인터넷 공유기를 목적으로 만든 리눅스 배포판이기 때문에 다양한 네트워크 프로토콜들이 OpenWRT에 구현되어 있어 OpenWRT를 사용하게 되었다.

| 호스트명         | 호스트 역할          | OS                  |
| ------------ | --------------- | ------------------- |
| VPN-Provider | VXLAN           | OpenWRT 21.04       |
| VPN-Customer | VXLAN           | OpenWRT 21.04       |
| PXE-Server   | DHCP, TFTP, PXE | Rocky Linux 8       |
| WEB-Server   | DNS, WEB        | Windows Server 2019 |
| PC01~04      | Client          | Windows 7 PE        |



IP 정보

| 호스트명         | IP              | IP             | VIP                  |
| ------------ | --------------- | -------------- | -------------------- |
| VPN-Provider | 53.82.37.28/24  |                | 87.27.97.1/24        |
| VPN-Customer | 37.99.46.78/24  |                | 87.27.97.2/24        |
| PXE-Server   |                 |                | 89.27.97.254/24      |
| WEB-Server   |                 |                | 89.27.97.100/24      |
| PC01~04      |                 |                | 87.27.97.0/24 (DHCP) |
| R1           | 53.82.37.254/24 | 10.0.0.2/32    |                      |
| R2           | 10.0.0.1/32     | 10.0.1.2/32    |                      |
| R3           | 10.0.1.1/32     | 10.0.2.2/32    |                      |
| R4           | 10.0.2.1/32     | 10.0.3.2/32    |                      |
| R5           | 10.0.3.1/32     | 37.99.47.28/24 |                      |

# 서비스 정보

## VXLAN(Virtual eXtensible Local Area Network)

모든 IP 라우팅 프로토콜을 사용하여 3계층(L3)을 통해 물리적인 환경의 제약 없이 2계층(L2) 네트워크를 확장할 수 있는 기술이다.

PXE는 DHCP를 이용해 IP를 할당받아 TFTP에서 이미지 파일을 받아 부팅 하므로, 같은 LAN 네트워크로 묶어주어야 하기 때문에 VXLAN을 설정한다.

### 특징

![Untitled 1](https://user-images.githubusercontent.com/66944342/144621556-43fd4c7d-e2d1-48de-8f78-ff705ac902c0.png)

MAC-in-UDP 캡슐화를 사용한다.

3계층의 설계 장점(확장, 대규모 네트워크 범위, 결함 도메인 최소화)과 2계층의 유동적인 특성(VLAN 및 MAC 주소 이동성)을 함께 제공 하므로 2계층과 3계층 설계의 단점을 모두 방지 할 수 있다.

### Config

1. 패키지 설치

`root@VPN-Provider:/# opkg install vxlan uci-proto-vxlan luci-proto-relay`

![Untitled 2](https://user-images.githubusercontent.com/66944342/144621569-e0914a47-60d7-4e3e-8d3f-816156414c47.png)

2. `/etc/config/netwark, /etc/config/firewall` 설정

`VPN-Provider /etc/config/netwark`

![Untitled 3](https://user-images.githubusercontent.com/66944342/144621576-0f9ec349-0d6f-49d4-a35b-3ed69cf8980c.png)

![Untitled 4](https://user-images.githubusercontent.com/66944342/144621581-bb241f4c-2f89-4f3b-8d37-39a3efced66a.png)



`VPN-Customer /etc/config/netwark`

![Untitled 5](https://user-images.githubusercontent.com/66944342/144621588-0ed55bf5-4aaf-41ef-b69d-bb15b63c4fab.png)

![Untitled 6](https://user-images.githubusercontent.com/66944342/144621589-2fa7b41b-692e-4799-9c0b-c43a9c0f0bf3.png)



`/etc/config/firewall`

![Untitled 7](https://user-images.githubusercontent.com/66944342/144621595-96c18979-ce77-4f0d-b0aa-a39a0cde0d18.png)

## PXE(Pre-boot eXecution Environment)

네트워크 인터페이스를 통해 컴퓨터를 부팅할 수 있게 해주는 환경이다.

### 구성요소

PXE Server

- 클라이언트에게 원격 부팅 및 운영체제 설치 환경을 제공하는 서버
- PXE Client에게 DHCP로 IP를 부여
- 부트 이미지 파일을 전송

PXE Client(PC)

- 운영체제를 설치하고자 하는 컴퓨터. 메인보드가 PXE를 지원해야 하고 PXE 지원 네트워크 카드가 필요함 (대부분은 지원)

### 과정

1. PXE Client가 부팅되면서 PXE Server에게 DHCP로 IP를 받아옴
2. PXE Server는 이용 가능한 운영체계가 들어있는 부트 서버의 목록을 PXE Client에게 보냄
3. PXE Client는 필요한 부트 서버를 찾은 다음, 다운로드할 파일이름을 받음 
4. PXE Client는 TFTP를 사용하여 파일을 다운로드하며, 그것을 실행시킴으로써 OS를 적재

### 특징

PXE의 특징으로는 설치함에 있어서 기술에 대한 요구와 서버당 소요되는 시간이 줄어든다. 

또한, 자동화로 인해서 에러가 덜 발생하고 OS 설치 도구는 중앙화되어 업데이트가 쉬워진다.

단, 서버 또는 네트워크 장애시 전체 시스템이 마비되는 단점이 있다.

## DHCP(Dynamic Host Configuration Protocol)

호스트의 IP주소와 각종 TCP/IP 프로토콜의 기본 설정을 클라이언트에게 자동적으로 제공해주는 프로토콜이다.

### Config

1. 패키지 설치

`[root@PXE-Server ~]# dnf install dhcp-server`

![Untitled 8](https://user-images.githubusercontent.com/66944342/144621596-9c484c3a-ddd4-476e-a88a-03632aa8be6e.png)



2. `/etc/dhcp/dhcpd.conf` 설정

![Untitled 9](https://user-images.githubusercontent.com/66944342/144621599-c6b21bc6-9654-432d-aed8-70a91a024f2b.png)

## TFTP(Trivial File Transfer Protocol)

FTP와 마찬가지로 파일을 전송하기 위한 프로토콜이지만, FTP보다 더 단순한 방식으로 파일을 전송한다.

FTP처럼 복잡한 프로토콜을 사용하지 않기 때문에 구현이 간단하여 운영체제 업로드에 주로 사용한다.

### Config

1. 패키지 설치

`[root@PXE-Server ~]# dnf install syslinux tftp-server xinetd`

![Untitled 10](https://user-images.githubusercontent.com/66944342/144621601-f373bb79-d741-436f-ab32-542eafd643d0.png)



2. firewall 설정

```fsharp
[root@PXE-Server ~]# firewall-cmd --permanent --add-service=ftp
[root@PXE-Server ~]# firewall-cmd --permanent --add-service=tftp
[root@PXE-Server ~]#firewall-cmd --permanent --add-service=dhcp
[root@PXE-Server ~]# firewall-cmd --permanent --add-service=proxy-dhcp
[root@PXE-Server ~]# firewall-cmd --add-port=69/tcp --permanent
[root@PXE-Server ~]# firewall-cmd --add-port=69/udp --permanent
[root@PXE-Server ~]# firewall-cmd --add-port=4011/udp --permanent
```

![Untitled 11](https://user-images.githubusercontent.com/66944342/144621604-5493ef5c-edfc-4df9-ae6f-b14c56cb9089.png)



3. `/etc/xinetd.d/tftp` 설정

`disable = no, server_args= -s /tftpboot` 로 변경 

![Untitled 12](https://user-images.githubusercontent.com/66944342/144621605-d9fa6426-6592-4cfa-9bc1-c9ed6f749f62.png)



4. ISO syslinux 부팅파일 복사

```fsharp
[root@PXE-Server ~]# cp /usr/share/syslinux/ldlinux.c32 /tftpboot/
[root@PXE-Server ~]# cp /usr/share/syslinux/libuial.c32 /tftpboot/
[root@PXE-Server ~]# cp /usr/share/syslinux/memdisk /tftpboot/
[root@PXE-Server ~]# cp /usr/share/syslinux/menu.c32 /tftpboot/
[root@PXE-Server ~]# cp /usr/share/syslinux/pxelinux.0 /tftpboot/
```

![Untitled 13](https://user-images.githubusercontent.com/66944342/144621607-cf979f6b-18bc-4a37-9fe4-2fce031661e4.png)



5. 부팅 관련 디렉터리와 설정 파일 생성

![Untitled 14](https://user-images.githubusercontent.com/66944342/144621609-c56cb145-f626-4d5d-86c8-ee36ccf6c9f1.png)



`/tftpboot/pxelinux.cfg/default`

![Untitled 15](https://user-images.githubusercontent.com/66944342/144621610-88fe09a2-cebf-42a6-a8a7-b937a885e27a.png)

![Untitled 16](https://user-images.githubusercontent.com/66944342/144621611-2abba5cd-ed3d-4bc3-bae8-fbb166e4c88a.png)

## DNS(Domain Name System)

도메인 이름을 호스트의 네트워크 주소로 바꾸거나 그 반대의 변환을 수행하는 시스템이다.

### 특징

DNS 서버는 계층 구조로 이루어져 있고 루트 DNS 서버, 최상위 레벨 서버, 책임 DNS 서버로 나누고 추가로 로컬 DNS 서버가 존재한다. 로컬 DNS 서버는 사용자에게 직접적으로 도메인에 대한 질의를 받고 그에 대한 응답을 해주는 서버의 역할을 수행한다.

### Config

1. 패키지 설치

DNS, IIS 설치

![Untitled 17](https://user-images.githubusercontent.com/66944342/144621612-cdad0268-b88e-4312-a8e0-7d5bea2264bd.png)



2. DNS 영역 설정

DNS 관리자 ⇒ 주 영역 ⇒ 영역이름 설정 "sunrin.com" ⇒ 영역파일 생성(다음 이름으로 새 파일 만들기) ⇒ 동적 업데이드(동적 업데이트 허용 안함)

![Untitled 18](https://user-images.githubusercontent.com/66944342/144621614-0d9cbda5-dc38-475a-a006-0a323ad9606f.png)



3. 호스트 추가

도메인 선택 후 우클릭 ⇒ 새 호스트(A 또는 AAAA) 선택 ⇒ 원하는 이름을 입력후 IP주소에 서버 IP를 넣고 호스트 추가

![Untitled 19](https://user-images.githubusercontent.com/66944342/144621617-0dd7b857-6b27-4821-ba1e-853c48872220.png)



4. 역방향 DNS 설정

역방향 DNS 우클릭후 새 영역 ⇒ 주 영역 선택 ⇒ IPv4 역방향 조회 영역 선택 ⇒ 네트워크 id에 현재 서버 IP ⇒ 동적 업데이드(동적 업데이트 허용 안함)

![Untitled 20](https://user-images.githubusercontent.com/66944342/144621619-03667fd0-b83f-4b30-87e6-3ca482f35d77.png)



5. 역방향 DNS PTR 포인터 추가

도메인 선택 후 우클릭 ⇒ 새 PTR 포인터⇒ IP주소에 서버 IP, 호스트 이름에 [`www.sunrin.com`](http://www.sunrin.com) 입력

![Untitled 21](https://user-images.githubusercontent.com/66944342/144621622-6dc5acf9-b7c9-4415-8326-90894d061e24.png)

## IIS(Internet Information Services)

마이크로소프트 윈도우를 사용하는 서버들을 위한 인터넷 기반 서비스들의 모임이다.

### 특징

OS 이용자의 대부분이 윈도우를 사용하여 쉽게 설치가 가능하며, 시각적으로 창(Window)에서 작업을 하는 경우가 많아 일반적인 텍스트(Text)로 작업을 할 때 보다는 훨씬 용이한 작업이 가능하다. 또한 ASP 스크립트 언어를 사용할 수 있다.

### Config

![Untitled 22](https://user-images.githubusercontent.com/66944342/144621624-012ea2ec-692c-4037-8632-0462385fad6e.png)



`C:\inetpub\sunrin\index.html` 설정

![Untitled 23](https://user-images.githubusercontent.com/66944342/144621628-f813c0ef-1d03-40f6-b629-94f326c9b875.png)

# 구축 결과

PC를 키면 바이오스가 DHCP 서버에서 IP를 할당받는다.

![Untitled 24](https://user-images.githubusercontent.com/66944342/144621630-1ee435de-9905-4457-9795-f7582a09bc94.png)



IP를 할당 받은 후, 부팅 가능한 OS 리스트를 선택하면 부팅 파일을 다운받는다.

![Untitled 25](https://user-images.githubusercontent.com/66944342/144621635-3c269445-fcb9-40fb-8576-a185f8957a9f.png)



OS 이미지를 다운 받아 메모리에 로드 한 후, 부팅 된 모습이다.

![Untitled 26](https://user-images.githubusercontent.com/66944342/144621638-885bc292-b472-494b-ba0d-49f4ee0df0ab.png)

![Untitled 27](https://user-images.githubusercontent.com/66944342/144621642-ee22e172-e386-4657-874d-3deca2290d62.png)



PC01과 PC03이 서로 통신된다.

![Untitled 28](https://user-images.githubusercontent.com/66944342/144621644-a4b300c8-805d-4fef-aa03-0a4fdfe0d360.png)

![Untitled 29](https://user-images.githubusercontent.com/66944342/144621650-0f98dacb-2ab2-4493-9ee2-910f9b7ba781.png)

# 느낀점

### 민제민

OpenWRT를 처음 사용해 봐서 인지 많은 게 낯설고 어려웠던 것 같다. L2TPv3를구축하다가 프로젝트 시간이 촉박한 관계로 VXLAN을 이용해서 의사 회선을 구축했는데 Cisco 라우터를 사용하지 못해서 IPSec을 통한 암호화를 하지 못한 게 아쉽다.

의사 회선과 DHCP 서버 구축 후에 GNS3에서 제공하는 VPC를 통해서 DHCP 테스트를 진행했는데 VPC의 문제로 DHCP Request 패킷이 전송되지 않는 문제가 있었는데(VPC에 문제가 있었다.) 이를 모르고 트러블슈팅을 하는 과정에서 MTU의 심화 이론, MSS Clamping에 대하여 더 자세하게 알게 되는 계기가 된 것 같다.

### 허린

상당히 많은 시간을 쏟아 부었다. OpenWRT를 사용하기 전 Rocky Linux로 L2TPv3를 구축하여 의사회선을 만들려 보려 했지만, 해당 프로토콜이 제대로 구현되어 있지 않아 OpenWRT에서와 L2TPv3 두 운영체제 모두 처음에 선택한 L2TPv3로 의사회선을 만들지 못해 아쉬웠다. (VXLAN은 기본적으로 IPsec 패킷으로 encapsulation하지 않기 때문에 보안에 취약하다.)

또한, GNS3의 한계로 시스코 라우터끼리의 통신이 매우 느리기 때문에, PC3과 PC4에서 PXE-Server로 부터의 부팅 이미지 다운로드가 극단적으로 느렸다.  이부분도 매우 아쉬웠다.

그래도 PXE를 처음 성공적으로 구축 한 것과, 2달 전부터 머릿속으로 그린 토플로지를 이번 서버구축 프로젝트를 통해 구축 할 수 있게 되어 행복했다.