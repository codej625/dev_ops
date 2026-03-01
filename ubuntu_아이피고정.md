# 우분투에서 아이피 고정

<br />
<br />

* 아이피를 왜 고정할까?

---

```
서버에서 아이피를 고정하는 이유는
네트워크 장비에서 특정 아이피 대역을 예약하고,
해당 아이피를 디바이스에서 사용하기 위함이다.

만약 사용하려는 아이피가 계속해서 변경된다면
제대로 된 서버 요청이 이루어지지 않는다.
```

<br />
<br />
<br />
<br />

1. 네트워크 인터페이스 & 게이트웨이 확인

```zsh
ip a # 인터페이스 이름 확인
ip route | grep default # 게이트웨이 확인
```

<br />
<br />
<br />

2. 공유기에서 IP 예약

```
공유기 관리 페이지에서 MAC 주소 기반으로 특정 IP를 예약한다.

DHCP 서버 주소관리에서 해당 아이피를 사용하지 못하게 한다.

* MAC 주소 확인 -> ip a 출력의 link/ether 값
```

<br />
<br />
<br />

3. Netplan 설정 파일 수정

```zsh
sudo nano /etc/netplan/00-installer-config.yaml

* {ip}/24 에서 24는 서브넷 마스크 255.255.255.0를 말한다.
```

```vim
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0: # 사용하는 인터페이스 이름
      dhcp4: no # DHCP 끄기
      addresses:
        - 192.168.0.221/24 # 예약한 고정 IP
      routes:
        - to: default
          via: 192.168.0.1 # 게이트웨이 (공유기 IP)
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

<br />
<br />
<br />

4. 적용 & 확인

```zsh
sudo netplan apply

ip a show eth0 # inet 192.168.0.221/24 ... valid_lft forever 확인
ping 8.8.8.8 # 외부 통신 확인
```
