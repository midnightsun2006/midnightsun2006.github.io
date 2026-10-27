---
title: "[Embedded]Embedded System Analysis Basics"
date: 2026-08-31 13:44:26 +0900
categories: [Study, Embedded]
tags: [firmware, emulation, qemu, buildroot, watchdog, general]
media_subpath: /assets/img/posts/2026-08-31-embedded-basic
---

임베디드 시스템을 분석하는 순서로는 크게 다음과 같다.
(내가 이해한 내용을 바탕으로 순서를 정리하였다. 오류가 매우 많다.)

## 1. 펌웨어 추출
임베디드 시스템을 분석할때는 펌웨어 추출이 필수적이다. 만약 펌웨어 추출 없이 블랙박스 기반으로만 분석하고자 하면 정밀한 분석이 불가능해지며, 취약점을 트리거하기도 힘들다.

펌웨어를 추출하는 방법으로는 UART,Flash memory, JTAG, Flashrom등 다양한 방법이 존재한다.

해당 부분은 추후 별도로 작성한다.

## 2. 파일 시스템 분석
펌웨어를 추출했으면 binwalk와 같은 명령어로 내부 파일 시스템을 추출하여 분석한다.
![](1.png)

![](2.png)

파일 시스템 : 35000.squashfs, squashfs-root
- 35000.squashfs : 이미 압축되어 있는 파일 시스템, 에뮬레이션 할때 해당 압축 형태를 사용한다.(35000은 오프셋(hex)를 의미한다.)
- squashfs-root : 직접 분석할 수 있도록 압축 해제된 파일 시스템

커널 : 54798.7z, 54798
- 54798.7z : LZMA 압축 데이터, 커널이 압축되어 있다.
- 54798 : LZMA 압축 해제 데이터, 실제 커널이다.
- 54798은 오프셋(hex)를 의미한다.
- 이러한 커널 이미지들은 실기기 보드용이라 QEMU virt에서 동작하지 않을 수 있어 지금은 사용하지 않았다.

부트로더 : u-boot.bin


![](3.png)

우선 실행시 부트로더->커널->init->inittab, rcS가 실행되므로 inittab과 rcS를 확인한다.

![](4.png)
inittab은 설정파일로 busybox init이 읽는 고유 형식으로 작성되어 있다. system을 준비하고 rcS을 실행시키는 것을 알 수 있다.

자세한 내용은 추후 작성한다.

![](5.png)
rcS 실행 파일로 셸 스크립트 형식이다. /etc/init.d/S??* 의 파일들을 실행하는 것을 알 수 있다.

자세한 내용은 추후 작성한다.

![](6.png)
/etc/init.d/S??* 의 파일들은 다음과 같은데 S01syslogd, S02klogd, S02sysctl, S15watchdog, S20urandom, S40network은 기본 실행 파일들이고, S99app이 실제 서비스 파일 이다.

![](7.png)
S99app의 내용은 /app디렉터리로 이동후 실행시키는 app파일을 실행시킨다는 것을 알 수 있다.

![](8.png)
app과 dream_dict가 존재한다.

이 파일들을 ghidra로 정적분석을, gdb로 동적분석을 하여 분석하여 취약점을 찾는다.

추가적으로 기기에 접속하기 위해서는 계정 정보가 필요한데, 이 정보들은 passwd와 shadow을 분석하여 알아내야 한다.
![](9.png)
passwd구조는
계정:비밀번호:UID:GID:기타:홈 디렉터리:기본 쉘 이다.
여기서 비밀번호가 X이면 shadow파일에 암호화 되어 저장되어 있다.

shadow구조는
계정:비밀번호 암호화:암호생성일:최소사용기간:최대사용기간:만기전 경고일:...의 구조를 가지고 있다.

shawdow의 비밀번호를 john the ripper로 복호화를 시도할 수 있다.
![](10.png)



## 3. 빌드(생성과 재압축)
동적분석을 하기 위해서는 빌드가 필요한 경우가 많다. 그 이유는 실제 임베디드 시스템에는 gdb가 없는 경우가 많기에 분석하고자 하는 기기에 맞는 gdbserver을 빌드하여 집어 넣어야 한다. 

gdb를 집어넣지 못하는 이유는 gdb는 용량이 무거워 기기에 못 넣을 수도 있기 때문이다.

빌드는 buildroot로 진행하며, 빌드하기 전에 기기 맞춰 .config를 조정해야 한다.

![](11.png)

buildroot 옵션은 다음과 같다.
- Target option : CPU 설정 조정
- Toolchain : 컴파일 도구 결정
- Kernel : 커널 생성 유무
- Target packages : 생성할 프로그램 결정
- Filesystem images : 파일 시스템 종류 결정
- Bootloaders : 부트로더 생성 유무 결정

![](12.png)
Target option에서

운영체제는 ARM-V7,
ABI는 EABI,
Floating point는 VFPv4-D16,
ARM mode는 ARM,
Target Binary는 ELF형식이므로 위와 같이 맞춰준다.

![](13.png)
Toolchain에서

C library는 print, malloc 같은 함수가 든 라이브러리 이므로 기기와 맞춰야 한다. 이는 qemu부팅 로그로 확인 가능하다.

또한 gdbserver를 빌드할 계획이므로 C++ support를 활성화 한다. 
![](14.png)
Target packages에서 gdbserver을 활성화 한다. 이는 Debugging, profiling and benchmark에 있다.
<br/>
Kernel은 위에서 언급했듯이 qemu의 virt보드에 맞지 않을 수 있어 이번에는 사용하지 않고, 문제에서 제공해준 zIamge커널 이미지를 사용하므로 세팅하지 않는다. kernel또한 버전에 맞춰서 build가 가능하다.

Filesystem images는 gdbserver만 생성할 뿐 filesystem을 별도로 생성하지 않으므로 세팅하지 않는다.

Bootloaders는 qemu가 -kernel로 직접 올리므로 세팅하지 않는다.
<br/>
이후 save하여 .config파일을 저장한 후 make하여 빌드한다.

![](15.png)
make한 산출물은 output폴더에 저장된다.

![](16.png)
이를 squashfs-root에 복사하여 파일시스템을 재압축한 후 에뮬레이션한다.

![](17.png)
mksquashfs로 gdbserver와 lib을 넣어 재압축을 해준다. 이때 -b 131072 -comp xz옵션은 binwalk한 결과를 참조하여 결정한다.
![](18.png)

-b 131072는 block size 131072bye임을 의미한다.
-comp xz는 compressiond의 결과가 xz임을 의미한다.



## 4. 에뮬레이션
이제 커널과 파일시스템이 모두 준비되었으므로 에뮬레이션을 진행할 수 있다.
![](19.png)

에뮬레이션을 진행할때는 qemu-system-arm을 사용하며, 옵션은 다음과 같다.

`-kernel ./zImage` : 커널 이미지는 제공된 zImage를 사용한다.

`-initrd ./_Target_Firmware.bin.extracted/new_fs` : 파일시스템은 gdbserver를 넣고 재압축시킨 파일시스템을 사용한다. 이때 파일시스템을 Ram disk로 올린다.

`-append "root=/dev/ram"` : root 파일시스템 경로를 지정한다. -initrd로 Ram disk로 올렸으므로 /dev/ram이 root가 된다. 자세한건 밑에서 다룬다.

`-M virt` : virt 가상 보드를 사용한다.

`-nographic` : CLI환경에서 작업한다. 만약 CLI환경지정을 안하면 GUI창을 띄우려다 오류가 발생하므로 CLI환경 지정이 필수다.

`-netdev user,id=eth0,hostfwd=tcp::1234-:1234` : 호스트(본 환경) 쪽 가상 네트워크 설정이다. 자세한건 밑에서 다룬다.

`-device virtio-net-device,netdev=eth0` : 게스트(qemu 환경) 쪽 가상 랜카드 설정이다. netdev=eth0으로 위에 eth0과 연결한다.

`-m 256M` : 게스트에 메모리를 할당한다. 실행한 후 오류 메세지가 발생하면 그에따라 조금씩 메모리를 늘린다.
![](20.png)
위와 같이 에뮬레이션이 성공하면 서비스에 접속을 위한 로그인을 요구한다. 이때 passwd를 분석하여 얻은 계정정보로 접속한다.

계정이름은 root, 패스워드는 dream 이었으므로 root에 접속할 수 있다.


<br/>
에뮬레이션의 옵션에서 Disk 설정과 네트워크 설정에서 주요하게 차이가 있다.

### RAM disk vs Block device
파일시스템을 Qemu에 올리는 방식은 크게 두가지가 있다.
**1. Ram disk(-initrd)**

- 파일시스템을 RAM에 통째로 올린다.
- 재부팅하면 변경사항이 날아간다.
- 설정이 간단하며, 실기기와 유사하다.
- 읽기 전용이다.(squashfs 이므로)
```bash
-initrd ./new_fs \
-append "root=/dev/ram"
```
{: file="qemu-system-arm (RAM disk)" }

**2. Block device(-device + -drive)**

- 파일시스템을 가상 디스크로 붙인다.
- ext2로 가상 디스크를 만들어 붙이면 쓰기도 가능하다.
- 재부팅해도 변경사항이 유지된다.
- 설정이 복잡하다.
- ext2로 붙이고자 하면 먼저 ext2파일 시스템을 생성해야 한다.
```bash
qemu-img create disk.img 512M -> 빈 이미지
mkfs.ext2 ./disk.img -> ext2 포맷
sudo mount ./disk.img ./test/ -> 마운트
cp -R ./squashfs-root/* ./test/ -> 내용 복사
sudo umount ./test -> 해제
```
{: file="ext2 disk image" }
- 그 후 device와 drive로 파일시스템을 ext2로 붙인다.
```bash
-device virtio-blk-device,drive=vd \
-drive file=./disk.img,format=raw,id=vd,if=none \
-append "root=/dev/vda"
```
{: file="qemu-system-arm (Block device)" }

### User mode vs TAP
에뮬레이션한 기기는 Qemu안에 갇혀 있으므로 통신하기 위해서는 네트워크 설정을 해야한다. `-netdev`로 호스트 네트워크를 설정하고, `-device`로 게스트 네트워크를 설정한다.

에뮬레이션 할때 네트워크 설정하는 방법은 크게 두가지가 있다.
**1. User mode(-netdev user)**

- qemu가 만든 가상 네트워크이다.
- host 네트워크에 얹혀간다.
- guest만 사설 ip이다.(10.0.2.15)
- 포트에 접근하기 위해 hostfwd 포워딩이 필요하다.
- 설정이 간단하다.

`-netdev user,id=eth0,hostfwd=tcp::1234-:1234`
netdev user : User mode로 설정.
id=eth0 : 해당 netdev의 이름은 eth0로 설정.
hostfwd=tcp::1234-:1234 : hostfwd=[프로토콜]:[호스트IP]:[호스트포트]-[게스트IP]:[게스트포트]의 구조를 가진다.
- tcp : protocol 이다.
- (생략):1234 : 호스트 ip와 호스트 포트이다. 생략한 경우 모든 인터페이스를 의미한다. 포트는 1234를 사용한다. 
- -(생략):1234 : 게스트 ip와 게스트 포트이다. 생략한 경우 게스트 기본 ip를 의미한다. 포트는 1234를 사용한다. 

`-device virtio-net-device,netdev=eth0`
device virtio-net-device : 게스트의 랜카드는 virtio-net-device로 설정.
netdev=eth0 : host의 id=eth0과 연결.

즉 host의 1234 포트로 오는 연결을 guest의 1234 포트로 전달한다는 의미이다.

그래서 host의 gdb가 127.0.0.1:1234로 접급하면 qemu가 그걸 포워딩 하여 guest의 gdbserver에 10.0.2.15:1234로 전달한다.


**2.TAP(-netdev tap)**

- qemu가 만든 가상 네트워크가 아니라 진짜 가상 네트워크를 생성한다.
- host에 가상 랜카드를 만들어 실제 두 대의 컴퓨터와 같이 대등 통신을 한다.
- host와 guest모두 static ip이다.
- 포트에 접근하기 위해 ip로 직접한다.
- 직접 연결되는 방식으로 실기기와 유사하다.
- 설정이 복잡하다.

`-netdev tap,id=eth0,ifname=tap0`
netdev user : User mode로 설정.
id=eth0 : 해당 netdev의 이름은 eth0로 설정.
ifname=tap0 : host에 만들 인터페이스 이름을 tap0으로 설정.

`-device virtio-net-device,netdev=eth0`
device virtio-net-device : 게스트의 랜카드는 virtio-net-device로 설정.
netdev=eth0 : host의 id=eth0과 연결.

위와 같은 옵션으로 에뮬레이션하면 host에 tap0이 생성되며 양쪽에 모두 수동으로 ip를 할당 시켜 설정을 마무리 한다.
`ifconfig eth0 192.168.1.10 -> guest`
`ifconfig tap0 192.168.1.11 -> host`



## 5. watchdog 비활성화
에뮬레이션한 기기에 접속해서 gdbserver을 붙이면 watchdog이 활성화 되면서 다시 재부팅 된다.
![](21.png)

watchdog은 시스템의 비정상적인 동작을 탐지하고 재시작과 복구를 하는 기능이다.

app은 내부에서 thread를 하나 돌리며 watchdog timer에게 주기적으로 alive 신호를 보내 자신이 정상 동작 중임을 알린다.

그런데 디버깅을 위해 gdbserver로 app에 attach하면 프로세스가 정지되고, 정지된 동안에는 thread도 멈춰 alive 신호를 보내지 못한다.

watchdog timer는 정해진 시간(timeout) 안에 alive 신호를 받지 못하면 시스템이 비정상이라 판단해 재시작을 시킨다. 따라서 디버깅을 하려면 이 watchdog을 먼저 비활성화해야 한다.

watchdog을 비활성화 하는 방법은 크게 2가지 있다.
### 매직클로즈
/dev/watchdog에 'V'를 입력하고 정상 close하면 watchdog이 비활성화 된다. 이를 매직 클로즈라 한다.

매우 간단헤 보이지만 실제로 `echo 'V' > /dev/watchdog`으로 /dev/watchdog에 'V'를 넣으려고 하면 app이 /dev/watchdog을 이미 점유하고 사용중이기 때문에 오류가 발생한다.
![](22.png)

따라서 먼저 gdb에서 /dev/watchdog을 점유중인 프로세스를 중지해야한다.
그 후 echo `'V' > /dev/watchdog`로 watchdog을 비활성화 한다.
다만 watchdog의 재시작 시간이 5초로 매우 짧기때문에 미리 setting파일을 만들어 gdb연결 + 프로세스 중지 + gdb연결 해제를 한번에 진행한다.
![](23.png)

아래와 같이 watchdog이 비활성화된 모습을 볼 수 있다.
![](24.png)
![](25.png)



### LD_PRELOAD
LD_PRELOAD는 리눅스의 환경변수로 ld.so가 함수를 실행하기 위해서는 실행파일인 libc를 참조해야 하는데, libc보다도 더 먼저 참조하는 값이다.

즉 일반적인 동적링커의 순서는
`system() -> ld -> libc`
지만 LD_PRELOAD는
`System() -> ld -> LD_PRELOAD -> libc`
의 순서를 가진다.

따라서 이러한 LD_PRELOAD를 이용하여 watchdog의 실행흐름을 가져와 watchdog를 비활성화 할 수 있다.

이러한 작업은 open과, ioctl함수를 LD_PRELOAD로 흐름을 변경시켜 수행할 수 있다.
![](26.png)
/dev/watchdog을 open하려고 하면 fd번호 100을 반환하고 그 외는 정상적으로 open한다.

fd번호가 100일때 0을 반환하고 그 외는 정상적으로 ioctl한다.
여기서 ioctl은 watchdog_thread가 watchdog에 timeout 설정하고 alive 신호 보낼 때 사용하는 함수이다.

<br/>

watchdog_thread: open("/dev/watchdog")
- LD_PRELOAD로 다른 open실행, fd=100반환
- watchdog_thread는 /dev/watchdog를 open했다고 착각

watchdog_thread: ioctl(100, timeout)
- LD_PRELOAD로 다른 ioctl실행
  -  fd==100이므로 바로 return 0
- watchdog_thread는 timeout을 설정했다고 착각

watchdog_thread: ioctl(100, alive 신호) 반복
- LD_PRELOAD로 다른 ioctl실행
- 진짜 watchdog엔 신호가 안 감

<br/>

그 후 파일시스템에 넣어주기 위해서 ARM 크로스 컴파일을 해야하며, 실행파일이 아니라 공유 라이브러리(.lib)로 만들어야 한다.

![](27.png)
```bash
TC=~/tools/buildroot-2023.02/output/host/bin/arm-buildroot-linux-gnueabi-gcc
$TC -fPIC -c debug.c -o debug.o
$TC -shared -o debug.lib debug.o -ldl
```
{: file="build debug.lib" }
- fPIC : 공유 라이브러리에 필요한 위치 독립 코드로 컴파일한다.
- c : 오브젝트 파일까지만 만든다.
- shared : 공유 라이브러리로 만든다.
- o debug.lib : 결과물 이름은 debug.lib이다.
- ldl : dlsym이 든 libdl 링크가 ldl이다.

![](28.png)
그런 다음 debug.lib을 같은 app디렉터리에 복사하고 똑같이 에뮬레이션을 진행한고

LD_PRELOAD로 app을 실행 시킨다.
```bash
cd /app
LD_PRELOAD=./debug.lib ./app &
```
{: file="guest shell" }

다만 현재 서비스하는 실행하자 마자 app이 자동으로 실행된다. 따라서 LD_PRELOAD로 app을 실행시키리면 kill을 해야하는데 kill하게 되면 watchdog이 실행되므로 LD_PRELOAD말고 매직바이트로 종료해야한다.

## 6. 분석
gdbserver을 이용하여 분석하는 단계로, 일반적인 분석과 동일하다.

해당 부분은 추후 별도로 작성한다.