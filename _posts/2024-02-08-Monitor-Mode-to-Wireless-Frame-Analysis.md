---
layout: post
title: "Dog Sniffing The Network: Monitor Mode To Wireless Frame Analysis"
summary: ""
author: david232818
category: ['learning']
thumbnail: /assets/img/posts/default.png
keywords: monitor mode, wireless frame analysis
usemathjax: false
permalink: /blog/Monitor-Mode-To-Wireless-Frame-Analysis
---


Contents
--------
1. Introduction
2. Monitor Mode To Capture Wireless Frames
   1. Opening wireless adapters via system calls
   2. Ioctls of wireless adapters in Linux
   3. Capturing wireless frames in the air
3. Wireless Frame Anaylsis
   1. Radiotap header
   2. IEEE 802.11 frame
   3. Channel hopping
4. Conclusion
0. References



## Introduction
와이파이 혹은 무선 랜 (Wireless LAN)상에서의 프레임을 캡처하는 프로그램을 작성하는 예제는 이미 많이 작성되어 있다. 그리고 이들은 대부분 libpcap이나 scapy, airodump-ng, airmon-ng 등을 사용한다. 물론, 이렇게 라이브러리를 사용하여 실습을 진행하는 것으로부터 많은 것을 배울 수 있다. 하지만 가능한 OS API와 표준 라이브러리만을 사용하여 구현해본다면, 보다 많은 것을 배울 수 있을 것이다.


이에 본 글에서는 외부 라이브러리의 사용을 최소화하여 무선 랜 프레임을 캡처하는 프로그램을 작성하기 위한 지식을 다룬다. 여기서 다루는 코드를 컴파일하고 실행하기 위해서는 우선 모니터 모드를 지원하는 무선 랜 어댑터가 있어야 한다: ipTIME N150UA2 (2.4GHz) [3], ipTIME A2000UA-4dBi (2.4 & 5GHz) [4]. 그리고 개발 환경으로는 우분투 22.04 [5] (리눅스 커널 버전: 6.5.0-15-generic)를 권장하며, `sudo dmesg` [6]를 사용할 수 있는 상태로 가정한다 (즉, 루트 권한도 사용할 수 있어야 한다).


본 글에서 사용하는 예제 코드들은 대부분 코드 조각이므로 완전한 예제 코드는 Wireless tool in C repo (URL: https://github.com/david232818/Wireless-tool-in-C)를 참고하라.


마지막으로, 당사자(송신인과 수신인)의 동의 없이, 그리고 통신비밀보호법, 형사소송법, 군사법원법의 규정에 의하지 않고 무선 랜 프레임을 캡처하는 것은 통신비밀보호법 제3조(통신 및 대화비밀의 보호) 등의 위반에 해당하며 제16조(벌칙) 등에 근거하여 법적 처벌을 받을 수 있다. 따라서 본 글의 내용은 모든 경우에 합법적이고, 연구 목적으로 사용되어야만 한다[11].

## Monitor Mode To Capture Wireless Frames
무선 랜 프레임을 캡처하기 위해서는 무선 랜 어댑터를 모니터 모드로 설정해야 하며, 이를 위해서는 해당 무선 랜 어댑터가 모니터 모드를 지원해야 한다. 이렇게 모니터 모드로 설정된 어댑터는 수신 가능한 범위에서 전송되고 있는 모든 패킷을 수신할 수 있다. 모니터 모드와 비슷하게 프레임을 캡처할 수 있는 프로미스큐어스 모드 (promiscuous mode)가 존재한다. 하지만 모니터 모드는 Access Point (AP)에 연결되지 않고도, 즉 공유기에 연결되지 않아도 프레임 캡처가 가능한 반면, 프로미스큐어스 모드는 공유기에 연결되어야 프레임 캡처가 가능하다[1, 2].


우분투가 설치된 컴퓨터에 무선 랜 어댑터를 연결하면 커널은 해당 어댑터의 드라이버로 등록된 커널 모듈을 로드한다. 이는 `sudo dmesg` 명령어로 확인할 수 있으며, 무선 랜 어댑터의 경우 해당 어댑터에 해당하는 무선 랜 인터페이스의 이름이 표시된다[6].

```Bash
$ sudo dmesg
...
[91112.413237] usb 2-1: new high-speed USB device number 16 using ehci-pci
[91112.790635] usb 2-1: New USB device found, idVendor=148f, idProduct=7601, bcdDevice= 0.00
[91112.790650] usb 2-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[91112.790659] usb 2-1: Product: 802.11 n WLAN
[91112.790669] usb 2-1: Manufacturer: MediaTek
[91112.790677] usb 2-1: SerialNumber: 1.0
[91113.133362] usb 2-1: reset high-speed USB device number 16 using ehci-pci
[91113.502626] mt7601u 2-1:1.0: ASIC revision: 76010001 MAC revision: 76010500
[91113.515830] mt7601u 2-1:1.0: Firmware Version: 0.1.00 Build: 7640 Build time: 201302052146____
[91116.893475] mt7601u 2-1:1.0: Vendor request req:07 off:0730 failed:-110
[91118.927602] mt7601u 2-1:1.0: EEPROM ver:0d fae:00
[91119.633770] ieee80211 phy14: Selected rate control algorithm 'minstrel_ht'
[91119.808372] mt7601u 2-1:1.0 wlx588694f7cb14: renamed from wlan0
...
$
```

위 로그에서 "wlx588694f7cb14"가 무선 랜 인터페이스의 이름이 된다.

### Opening wireless adapters via system calls
일반적으로 userland에서 디바이스를 제어하는 방법은 디바이스 파일을 열고 `ioctl()`의 command를 사용하는 것이다. 그럼 우리는 다음 두 가지를 알아야 한다:

 - Opening wireless adpater like device file
 - `ioctl()` command for wireless adapter that linux kernel provides


그럼 먼저 무선 랜 인터페이스를 디바이스 파일과 같이 여는 방법을 알아보겠다. 그 방법은 바로 소켓을 여는 것이다. 즉, 디바이스 파일 기술자에 적용되는 `ioctl()` 연산이 네트워크 소켓에도 적용될 수 있다는 것이다[8].

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <linux/if_ether.h>

int open_wlan_if(void)
{
    int fd;

    fd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (fd == -1)
        fprintf(stderr, "Cannot open socket..\n");
    return fd;
}
```

그러나 이 소켓이 디바이스 파일처럼 그 디바이스를 특정하는 것은 아니다. 그래서 이 소켓 파일을 `ioctl()`에 전달하여 무선 랜 인터페이스를 제어하기 위해서는 해당 인터페이스를 특정하기 위한 정보를 커널에게 전달해주어야 한다. Linux `ioctl()`에는 command에 해당하는 데이터를 인자로 전달할 수 있다. 따라서 인터페이스를 특정하기 위한 정보를 이 인자에 포함시켜 `ioctl()`에 전달하면 된다[7].


이제 특정 무선 랜 인터페이스를 `ioctl()`로 제어하는 방법을 알아보겠다. Linux는 네트워크 디바이스를 설정하기 위한 표준적인 ioctls를 지원한다. 이는 모든 소켓 파일 기술자와 함께 사용될 수 있고, 다음과 같은 `struct ifreq`를 인자로 전달하여 사용하게 된다[9]:

```C
struct ifreq {
    char ifr_name[IFNAMSIZ]; /* Interface name */
    union {
        struct sockaddr ifr_addr;
        struct sockaddr ifr_dstaddr;
        struct sockaddr ifr_broadaddr;
        struct sockaddr ifr_netmask;
        struct sockaddr ifr_hwaddr;
        short           ifr_flags;
        int             ifr_ifindex;
        int             ifr_metric;
        int             ifr_mtu;
        struct ifmap    ifr_map;
        char            ifr_slave[IFNAMSIZ];
        char            ifr_newname[IFNAMSIZ];
        char           *ifr_data;
    };
};
```

이때 커널은 `ifr_name[]`으로 인터페이스를 식별하고, `union` structure에 해당하는 값을 가져오거나, 해당하는 값으로 설정한다. 위 코드의 `union` structure에서 무선 랜 어댑터를 모니터 모드로 바꾸기 위한 멤버는 `ifr_flags`이다. 이 멤버는 해당 디바이스 (여기서는 무선 랜 인터페이스)의 활성화된 플래그를 얻거나 (`ioctl` command: `SIOCGIFFLAGS`) 설정할 때 (`ioctl` command: `SIOCSIFFLAGS`) 사용된다. 이 멤버는 다음과 같은 값들에 대한 비트 마스크를 가질 수 있다[7, 9]:

```
IFF_UP            Interface is running.
IFF_BROADCAST     Valid broadcast address set.
IFF_DEBUG         Internal debugging flag.
IFF_LOOPBACK      Interface is a loopback interface.
IFF_POINTOPOINT   Interface is a point-to-point link.
IFF_RUNNING       Resources allocated.
IFF_NOARP         No arp protocol, L2 destination address not
                set.
IFF_PROMISC       Interface is in promiscuous mode.
IFF_NOTRAILERS    Avoid use of trailers.
IFF_ALLMULTI      Receive all multicast packets.
IFF_MASTER        Master of a load balancing bundle.
IFF_SLAVE         Slave of a load balancing bundle.
IFF_MULTICAST     Supports multicast
IFF_PORTSEL       Is able to select media type via ifmap.
IFF_AUTOMEDIA     Auto media selection active.
IFF_DYNAMIC       The addresses are lost when the interface
                goes down.
IFF_LOWER_UP      Driver signals L1 up (since Linux 2.6.17)
IFF_DORMANT       Driver signals dormant (since Linux 2.6.17)
IFF_ECHO          Echo sent packets (since Linux 2.6.25)
```

위 값들 중에서 여기서 관심 가지는 비트 마스크는 `IFF_UP` 뿐임을 기억하라. 이 값은 인터페이스 모드를 바꿀 때 잠시 인터페이스를 down시키고 다시 up시키는데 사용된다. 이를 C 코드로 작성하면 다음과 같다[9, 10]:

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <linux/if_ether.h>

int if_down(const int fd, /* socket file descriptor */
          const char *ifname) /* interface name */
{
    struct ifreq ifr;
    
    if (fd == -1)
        return -1;

    strncpy(ifr.ifr_name, ifname, IFNAMSIZ);    
    ifr.ifr_flags = (short) 0xffff;
    ifr.ifr_flags &= ~IFF_UP; /* interface down */
    if (ioctl(fd, SIOCSIFFLAGS, &ifr) == -1) {
        fprintf(stderr, "Interface down failed..\n");
	return -1;
    }
    return 0;
}

int if_up(const int fd, /* socket file descriptor */
          const char *ifname) /* interface name */
{
    struct ifreq ifr;
    
    if (fd == -1)
        return -1;
    
    strncpy(ifr.ifr_name, ifname, IFNAMSIZ);    
    ifr.ifr_flags = 0;
    ifr.ifr_flags |= IFF_UP; /* interface up */
    if (ioctl(fd, SIOCSIFFLAGS, &ifr) == -1) {
        fprintf(stderr, "Interface up failed..\n");
	return -1;
    }
    return 0;
}
```

그럼 무선 랜 인터페이스 모드는 어디에 정의되어 있을까? 바로 `include/uapi/linux/wireless.h`에 정의되어 있다[9, 10].

```C
/* Modes of operation */
#define IW_MODE_AUTO	0	/* Let the driver decides */
#define IW_MODE_ADHOC	1	/* Single cell network */
#define IW_MODE_INFRA	2	/* Multi cell network, roaming, ... */
#define IW_MODE_MASTER	3	/* Synchronisation master or Access Point */
#define IW_MODE_REPEAT	4	/* Wireless Repeater (forwarder) */
#define IW_MODE_SECOND	5	/* Secondary master/repeater (backup) */
#define IW_MODE_MONITOR	6	/* Passive monitor (listen only) */
#define IW_MODE_MESH	7	/* Mesh (IEEE 802.11s) network */
```

그러나 `struct ifreq`에는 위 값들을 유의미하게 저장할 멤버가 없다. 즉, 위 값들을 가져오거나 설정하기 위해서는 다른 `ioctl` command를 사용해야 하고, 이에 해당하는 구조체가 필요하다. 그 command와 구조체는 `include/uapi/linux/wireless.h`에 정의되어 있으며, 각각 다음과 같다[10]:

```C
/* ... */

#define SIOCSIWMODE	0x8B06		/* set operation mode */
#define SIOCGIWMODE	0x8B07		/* get operation mode */

/* ... */

/*
 * This structure defines the payload of an ioctl, and is used
 * below.
 *
 * Note that this structure should fit on the memory footprint
 * of iwreq (which is the same as ifreq), which mean a max size of
 * 16 octets = 128 bits. Warning, pointers might be 64 bits wide...
 * You should check this when increasing the structures defined
 * above in this file...
 */
union iwreq_data {
	/* Config - generic */
	char		name[IFNAMSIZ];
	/* Name : used to verify the presence of  wireless extensions.
	 * Name of the protocol/provider... */

	struct iw_point	essid;		/* Extended network name */
	struct iw_param	nwid;		/* network id (or domain - the cell) */
	struct iw_freq	freq;		/* frequency or channel :
					 * 0-1000 = channel
					 * > 1000 = frequency in Hz */

	struct iw_param	sens;		/* signal level threshold */
	struct iw_param	bitrate;	/* default bit rate */
	struct iw_param	txpower;	/* default transmit power */
	struct iw_param	rts;		/* RTS threshold */
	struct iw_param	frag;		/* Fragmentation threshold */
	__u32		mode;		/* Operation mode */
	struct iw_param	retry;		/* Retry limits & lifetime */

	struct iw_point	encoding;	/* Encoding stuff : tokens */
	struct iw_param	power;		/* PM duration/timeout */
	struct iw_quality qual;		/* Quality part of statistics */

	struct sockaddr	ap_addr;	/* Access point address */
	struct sockaddr	addr;		/* Destination address (hw/mac) */

	struct iw_param	param;		/* Other small parameters */
	struct iw_point	data;		/* Other large parameters */
};

/*
 * The structure to exchange data for ioctl.
 * This structure is the same as 'struct ifreq', but (re)defined for
 * convenience...
 * Do I need to remind you about structure size (32 octets) ?
 */
struct iwreq {
	union
	{
		char	ifrn_name[IFNAMSIZ];	/* if name, e.g. "eth0" */
	} ifr_ifrn;

	/* Data part (defined just above) */
	union iwreq_data	u;
};

/* ... */
```

위 코드에서 `ifr_name[]` 역할을 하는 것은 `ifrn_name[]`이다. 그리고 `struct iwreq_data`의 `mode` 멤버를 통해 현재 모드 값을 가져오거나, 모드 값을 설정할 수 있다. 이를 C 코드로 작성하면 다음과 같다[10]:

```C
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <linux/wireless.h>

int iw_set_monitor_mode(const int fd, /* socket file descriptor */
                        const char *ifname) /* interface name */
{
    struct iwreq iwr;
    
    if (fd == -1)
        return -1;

    strncpy(iwr.ifr_ifrn.ifrn_name, ifname, IFNAMSIZ);
    iwr.u.mode = IW_MODE_MONITOR;
    if (ioctl(fd, SIOCSIWMODE, &iwr) == -1) {
        fprintf(stderr, "Setting monitor mode failed..\n");
	return -1;
    }
    return 0;
}
```


지금까지 설명한 것을 종합하여 어댑터 모드를 설정하는 순서를 정리하면 다음과 같다:

```
1. Open socket file
2. Set interface flag to down
3. Set interface wireless mode flag to IW_MODE_MONITOR
4. Set inter face flag to up
```

위 순서에 대한 완전한 예제 코드는 Wireless tool in C의 `prv_ieee80211_set_if_mode()`를 참고하라.

### Capturing wireless frames in the air
이렇게 모니터 모드가 구성되었으므로 무선 랜 프레임을 캡처할 수 있다. 그 방법은 바로 `read()` system call을 호출하는 것이다[16].

```C
#include <stdio.h>
#include <stdint.h>
#include <stddef.h>
#include <unistd.h>

/*
 * admp_hexdump: hexdump function written by ccbrown;
 * thanks to ccbrown!!
 */
void ccbrown_hexdump(const void* data, size_t size)
{
    char ascii[17];
    size_t i, j;

    ascii[16] = '\0';
    for (i = 0; i < size; ++i) {
	printf("%02X ", ((unsigned char *) data)[i]);
	if (((unsigned char *) data)[i] >= ' ' 
	    && ((unsigned char *) data)[i] <= '~') {
	    ascii[i % 16] = ((unsigned char *) data)[i];
	} else {
	    ascii[i % 16] = '.';
	}
	if ((i + 1) % 8 == 0 || i + 1 == size) {
	    printf(" ");
	    if ((i + 1) % 16 == 0) {
		printf("|  %s \n", ascii);
	    } else if (i + 1 == size) {
		ascii[(i + 1) % 16] = '\0';
		if ((i + 1) % 16 <= 8) {
		    printf(" ");
		}
		for (j = (i + 1) % 16; j < 16; ++j) {
		    printf("   ");
		}
		printf("|  %s \n", ascii);
	    }
	}
    }
}

int ieee80211_pcap(int fd) /* socket file descriptor */
{
    uint8_t buff[BUFSIZ];
    ssize_t len;
    
    if (fd == -1)
        return -1;
    
    while ((len = read(fd, buff, BUFSIZ)) >= 0)
        ccbrown_hexdump(buff, len);
    return 0;
}
```

위 코드를 보면 무선 랜 프레임을 캡처하는 것은 매우 간단해보인다. 그러나 문제는 위 코드로 출력되는 값들의 의미를, 아스키 코드로 표현되는 것이 아니라면 전혀 알 수 없다는 것이다. 그래서 무선 랜 프레임을 분석해볼 필요가 있다.

## Wireless Frame Analysis
무선 랜 프레임에는 그 크기가 정해진 필드도 있지만 가변 필드도 존재한다. 그리고 이러한 가변 필드들은 무선 랜 프레임 파싱을 어렵게 만든다. 하지만 가변 필드에도 나름의 원칙이 있기 때문에 이를 기억한다면 파싱하는 것은 그다지 어렵지 않다. 그럼 Beacon frame (무선 랜 AP 또는 와이파이 공유기가 자신의 존재를 알리는 프레임)으로부터 SSID (무선 랜 AP의 이름 또는 와이파이 공유기 이름)를 파싱하는 것을 목표로, 시작해보겠다.


무선 랜 프레임은 일반적으로 radiotap header (de facto standard)와 IEEE 802.11 frame으로 구성된다. 여기서는 IEEE 802.11 frame에서 SSID만 추출할 것이기 때문에 radiotap header가 그다지 비중있게 다루어지지는 않지만, radiotap header는 신호 세기와 같은 유용한 정보들을 담고 있고 이를 파싱하는 것은 예제로 제공할 것이다[13, 14].


Radiotap header를 분석하기에 앞서서, 가변 필드의 구성 원칙을 알아보자. 무선 랜 프레임에서 가변 필드는 TLD 형식으로 구성된다. TLD 형식이란 Type (EID로 표현됨), Length, Data로 필드를 구성하는 형식을 의미한다. 이때 type은 해당 필드가 어떤 정보를 가지고 있는지, length는 해당 필드의 길이, data는 실제 데이터를 의미한다. 그럼 우리는 다음과 같이 가변 필드에 대한 구조체를 정의할 수 있다[12]:

```C
struct tld {
    unsigned int type;
    size_t len;
    unsigned char data[];
};
```

### Radiotap header
Radiotap header는 다음과 같이 구성된다[13]:

```
             [version][padding][length][present|extended][ ... ]
byte length:    1        1        2       4       4
```

위 그림에서 `present` 부분이 의미하는 것은 4 바이트 값인 `present`에 따라 `extended`가 될 수도 있고 안될 수도 있다는 것이다. Radiotap header에는 `present` 값에 따라 추가되는 필드들이 존재한다. 즉, `present` 값에 따라 시작 주소에서 추출하고자 하는 필드까지의 offset이 달라진다는 것이다[13].


이를 파싱하기 위해서는 `present`의 각 비트에 따른 길이 값을 테이블로 만들고 ([13]의 https://www.radiotap.org/fields/defined 참고) 인덱스 기반으로 offset을 구해야 한다. 이러한 방식으로 신호 세기 필드까지의 offset을 구하는 C 코드를 작성하면 다음과 같다[10, 13, 14]:

```C
#include <stdint.h>
#include <stddef.h>

struct radiotap_hdr {
    uint8_t ver; /* version */
    uint8_t pad; /* padding */
    uint16_t len; /* entire length of radiotap header */
    uint32_t present;
    uint8_t variable[]; /* payload */
} __attribute__((__packed__));

unsigned int radiotap_get_offset(uint64_t present,
			        unsigned int bit_loc) /* bit location of
				                         specific field */
{
    unsigned int offsettab[] = {
	sizeof(uint64_t),	/* TSFT */
	sizeof(uint8_t),	/* Flags */
	sizeof(uint8_t),	/* Rate */
	sizeof(uint16_t),	/* Channel */
	sizeof(uint8_t) + sizeof(uint8_t) /* hop set + hop pattern */

	/* [TODO] Add defined fields */
    };
    unsigned int offset;
    uint64_t mask, i;

    offset = 0;
    for (i = 0; i < bit_loc; i++) {
	mask = 1 << i;
	if (mask & present)
	    offset += offsettab[i];
    }
    return offset;
}

int get_antsig(void *buff, uint16_t len)
{
    unsigned int offset;
    struct radiotap_hdr *header;
    
    header = buff;
    offset = radiotap_get_offset(header->present, 5);
    if (header->present & (1 << 31))
        offset += sizoef(uint32_t);
    return *(header->variable + offset);
}
```

그럼 `bit_loc`은 어떻게 구할 수 있을까? 이것 또한 [13]의 https://www.radiotap.org/fields/defined 를 참고하여 해당 필드가 몇 번째 비트로 정의되어 있는지 알 수 있다.


이러한 radiotap header가 앞에 존재할 때 IEEE 802.11 frame의 시작 주소를 구하는 방법은 무엇일까? 바로 상기의 `struct radiotap_hdr`의 `len` 멤버를 캡처된 데이터의 시작주소에 더하는 것이다. 이제 IEEE 802.11 frame을 분석할 차례다.

### IEEE 802.11 Frame Analysis
Generic한 무선 랜 프레임은 다음과 같이 구성된다[15]:

```
             [Frame  ][Duration][addr1][addr2][addr3][Sequence][addr4][ ... ][FCS]
              Control][ID      ][     ][     ][     ][Control ][     ][     ][   ]
byte length:    2        2         6      6      6      2         6    0-2312  4
```

Generic한 프레임이라는 것은 프레임의 종류에 따라 헤더의 구성이 달라질 수 있다는 뜻이다. 그리고 프레임의 종류는 위 헤더의 `Frame Control` 필드의 값에 따라 결정된다. 이 필드는 다시 여러 비트 필드로 나뉘어지는데, 여기서 관심을 가지는 필드는 type과 subtype이다. 전자는 프레임의 종류를 결정하고, 전자로부터 결정된 프레임 종류 내에서의 하위 분류를 나타낸다[15].

```
      [Protocl][Type ][Subtype ][To DS][From DS][More][Retry][Power][WEP][Rsvd]
      [Version][     ][        ][     ][       ][Flag][     ][Mgmt ][   ][    ]
bit   :  2        2         4       1       1       1     1     1     1    1
length
```

여기서 추출하고자 하는 정보는 beacon frame에서의 SSID이므로 beacon frame에 초점을 맞출 것이다. 먼저 frame control 필드의 type은 다음과 같은 값을 가질 수 있고,

```
@ 00: Management frame
@ 01: Control frame
@ 10: Data frame
```

management frame의 subtype은 다음과 같이 값에 따라 나뉘어진다[15]:

```
@ 0000: Association request frame
@ 0001: Association response frame
@ 0010: Reassociation request frame
@ 0011: Reassociation response frame
@ 0100: Probe request frame
@ 0101: Probe response frame
@ 1000: Beacon frame
@ 1001: ATIM
@ 1010: Association destroy frame
@ 1011: Authentication frame
@ 1100: Deauthentication frame
```

위 정보를 이용하여 캡처된 프레임이 management frame인지 확인하는 것을 C 코드로 작성하면 다음과 같다[10]:

```C
#include <stdio.h>
#include <string.h>
#include <stdint.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>

struct admp_ieee80211_frame_control {
    uint16_t proto: 2,
	    type: 2,
	    subtype: 4,
	    ds: 2,		/* [tods][fromds] */
	    moreflag: 1,
	    retry: 1,
	    pwrmgmt: 1,
	    moredata: 1,
	    wep: 1,
	    order: 1;
} __attribute__((__packed__));

struct admp_ieee80211_sequence_control {
    uint16_t fragnum: 4,
	    seqnum: 12;
} __attribute__((__packed__));

/*
 * This header structure is for generic case. But in IEEE 802.11,
 * header structure varies by frame categories.
 */
struct admp_ieee80211_mac_frame {
    struct admp_ieee80211_frame_control fc;
    uint16_t durid;		/* Duration / ID */
    uint8_t addr1[6];
    uint8_t addr2[6];
    uint8_t addr3[6];
    struct admp_ieee80211_sequence_control seqctl;
    uint8_t addr4[6];
    uint8_t variable[];
} __attribute__((__packed__));

struct admp_ieee80211_mac_trailer {
    uint32_t fcs;
} __attribute__((__packed__));

/* ieee80211_parser: parse given buff as a generic frame */
static int ieee80211_parser(void *buff, /* captured frame */
	      	            ssize_t len) /* captured frame's length */
{
    struct admp_ieee80211_mac_frame *frame;

    if (buff == NULL || ap == NULL || len < 0) {
        fprintf(stderr, "Invalid arguments..\n");
	return -1;
    }
    
    frame = buff;
    switch (frame->fc.type) {
    case 0b00:
	/* Management frame */
	break;
    case 0b01:
	/* Control frame */
	break;
    case 0b10:
	/* Data frame */
	break;
    default:
	break;
    }
    return 0;
}
```

이렇게 수신된 프레임이 management frame으로 확인되면, 헤더의 구성이 달라진다. Management frame, 그 중에서도 beacon frame의 헤더 구성은 다음과 같다[15]:

```
             [Frame  ][Duration][addr1][addr2][addr3][Sequence][ ... ][FCS]
              Control][ID      ][     ][     ][     ][Control ][     ][   ]
byte length:    2        2         6      6      6      2       0-2312  4


[ ... ]:     [Timestamp][Beacon  ][Capacity   ][  SSID  ][Option field]
             [         ][Interval][Information][        ][            ]
byte length:     8          2          3        variable    variable
                                                length      length
             <-----------------Required-----------------><--Optional-->
```

위 헤더의 가변 길이로 표시된 SSID와 Option field가 바로 가변 필드에 해당하는 부분이며, TLD 형식을 따른다. 여기서 상기에 설명한 TLD 구성 원칙을 기억하면서 beacon frame을 파싱하는 코드를 작성하면 다음과 같다[10, 12, 15]:

```C
#include <stdio.h>
#include <string.h>
#include <stdint.h>
#include <stddef.h>
#include <unistd.h>
#include <sys/types.h>

#define IS_X_IN_RANGE(min, x, max) (((x) >= (min)) && (x) < (max))

struct admp_ieee80211_mgmt_frame {
    /* header */
    struct admp_ieee80211_frame_control fc;
    uint16_t durid;
    uint8_t addr1[6];
    uint8_t addr2[6];
    uint8_t addr3[6];
    struct admp_ieee80211_sequence_control seqctl;
    uint8_t variable[];
} __attribute__((__packed__));

struct admp_ieee80211_beacon {
    uint64_t tmstamp;		/* Timestamp */
    uint16_t interval;		/* Beacon interval */
    uint16_t capinfo;		/* Capability information */
    uint8_t variable[];		/* Information Elements */
} __attribute__((__packed__));

struct admp_ieee80211_tlv {
    uint8_t type;		/* EID */
    uint8_t len;		/* Length */
    uint8_t value[];		/* Data */
} __attribute__((__packed__));

/* ieee80211_beacon_parser: parse given frame as a beacon frame */
static int
ieee80211_beacon_parser(struct admp_ieee80211_mgmt_frame *frame,
			    ssize_t len)
{
    unsigned int done;
    struct admp_ieee80211_beacon *beacon;
    struct admp_ieee80211_tlv *ie;

    if (frame == NULL || ap == NULL || len < 0) {
        fprintf(stderr, "Invalide argument..\n");
	return -1;
    }

    beacon = (struct admp_ieee80211_beacon *) frame->variable;
    ie = (struct admp_ieee80211_tlv *) beacon->variable;
    done = 0;
    while (len > 0 && done == 0) {
	if (!IS_X_IN_RANGE(0, ie->len, len))
	    break;
	
	switch (ie->type) {
	case 0x00:
	    /* SSID */
	    strncpy(ap->ssid, (char *) ie->value, ie->len);
	    break;
	default:
	    done = 1;
	    break;
	}
	ie = (struct admp_ieee80211_tlv *) ((uint8_t *) ie + ie->len);
	len -= ie->len;
    }
    return 0;
}

/* ieee80211_mgmt_parser: parse given buff as a management frame */
static int ieee80211_mgmt_parser(void *buff, /* captured frame */
					    ssize_t len) /* captured frame's length */
{
    struct admp_ieee80211_mgmt_frame *frame;

    if (buff == NULL || ap == NULL || len < 0) {
        fprintf(stderr, "Invalid arguments..\n");
	return -1;
    }
    
    frame = buff;

    /* printf("%d\n", frame->fc.subtype); */
    switch (frame->fc.subtype) {
    case 0b0000:
	/* Association request frame */
	break;
    case 0b1000:
	/* Beacon frame */
	ieee80211_beacon_parser(frame, len);
	break;
    default:
	break;
    }
    return 0;
}
```

## Conclusion
지금까지 가능한 linux kernel이 제공하는 system call과 C 언어의 표준 라이브러리가 제공하는 기능만을 사용하여 무선 랜 프레임을 캡처하는 프로그램을 작성하기 위한 지식을 알아보았다. 이는 무선 랜 인터페이스를 모니터 모드로 바꾸는 것으로 시작하여 무선 랜 프레임 중 beacon frame을 파싱하는 코드를 작성하는 것으로 끝을 맺었다. 비록 코드 조각으로 예제를 제공하였지만, 완전한 예제 코드를 이해하는 데에는 무리가 없을 것이라고 생각한다.


물론, 여기서 제공한 예제들은 (심지어 완전한 예제 코드라고 지칭한 것도) 한정적인 목적에만 부합하도록 작성되어 있다. 따라서 이들이 완전한 파서로 기능하기 위해서는 다른 radiotap header 필드, 다른 프레임들, 다른 TLD 형식 값들도 고려되어야 할 것이다.

## References
1. He-Jun Lu and Yang Yu, "Research on WiFi Penetration Testing with Kali Linux," Hindawi, vol. 2021, Article ID 5570001, 2021.
2. Vladimir Leiv, "What is the difference between Promiscuous and Monitor mode in Wireless Networks?," 2013. [Online]. Available: https://security.stackexchange.com/questions/36997/what-is-the-difference-between-promiscuous-and-monitor-mode-in-wireless-networks, [Accessed Feb. 06, 2024].
3. "ipTIME N150UA2", [Online]. Available: https://iptime.com/iptime/?page_id=11&pf=9&page=&pt=499&pd=1, [Accessed Feb. 06, 2024].
4. "ipTIME A2000UA-4dBi", [Online]. Available: https://iptime.com/iptime/?page_id=11&pf=8&page=&pt=248&pd=1, [Accessed Feb. 06 2024].
5. "Ubuntu 22.04.3 LTS (Jammy Jellyfish)," 2022. [Online]. Available: https://releases.ubuntu.com/jammy/, [Accessed Feb. 06, 2024].
6. "dmesg(1) -- Linux manual page," 2023. [Online]. Available: https://man7.org/linux/man-pages/man1/dmesg.1.html, [Accessed Feb. 06, 2024].
7. "ioctl(2) -- Linux manual page," [Online]. Available: https://man7.org/linux/man-pages/man2/ioctl.2.html, [Accessed Feb. 06, 2024].
8. Jean Tourrilhes, "Wireless Extensions for Linux," 1997. [Online]. Available: https://hewlettpackard.github.io/wireless-tools/Linux.Wireless.Extensions.html, [Accessed Feb. 07, 2024].
9. "netdevice(7) -- Linux manual page," [Online]. Available: https://man7.org/linux/man-pages/man7/netdevice.7.html, [Accessed Feb. 07, 2024].
10. torvalds, "Linux kernel," 2024. [Online]. Available: https://github.com/torvalds/linux, [Accessed Feb. 07, 2024].
11. "통신비밀보호법", 법무부(공공형사과) 및 과학기술정보통신부(통신자원정책과), 2022. [Online]. Available: https://www.law.go.kr/%EB%B2%95%EB%A0%B9/%ED%86%B5%EC%8B%A0%EB%B9%84%EB%B0%80%EB%B3%B4%ED%98%B8%EB%B2%95, [Accessed Feb. 07, 2024].
12. Nico Waisman, "Anatomy of a Coffee Bean (Wireless Vulnerabilities in Linux Kernel)," 2019. [Online]. Available: https://securitylab.github.com/research/anatomy-of-a-coffee-bean-wireless-vulnerabilities-in-linux-kernel/, [Accessed Feb. 08, 20224].
13. "Radiotap," [Online]. Available: https://www.radiotap.org/, [Accessed Feb. 08, 2024].
14. 2N(nms200299), "[802.11] RadioTab Header(헤더) 분석", 2021. [Online]. Available: https://blog.naver.com/nms200299/222267279476, [Accessed Feb. 08, 2024].
15. "802.11 MAC Frame, WLAN MAC Frame 802.11 MAC 프레임", 정보통신기술용어해설. [Online]. Available: http://www.ktword.co.kr/test/view/view.php?m_temp1=3352, [Accessed Feb. 07, 2024].
16. ccbrown, "Compact C Hex Dump Function w/ASCII," [Online]. Available: https://gist.github.com/ccbrown/9722406, [Accessed Feb. 08, 2024].
