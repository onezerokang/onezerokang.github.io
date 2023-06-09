# 네트워크

네트워크란 두 대 이상의 컴퓨터가 데이터를 주고 받기 위해 연결된 시스템이다.
여기서 한 건물 정도의 범위를 갖는 네트워크를 LAN(Local Area Network)라고 하며 다수의 LAN을 연결한 광범위한 크기의 네트워크를 WAN(Wide Area Network)라고 한다. 또한 전 세계적으로 연결된 수많은 네트워크의 집합체를 인터넷(inter-network)라고 한다.

이러한 네트워크는 컴퓨터간의 물리적인 연결을 지원하는 하드웨어와 네트워크를 관리하고 데이터 전송을 위한 규약을 정의하는 프로토콜로 구성된다.

![IMG_47D71AA12FEA-1](https://github.com/onezerokang/onezerokang.github.io/assets/60874549/dabf00e2-fc3b-4ecc-95b1-b4c93923c58b)

## 네트워크를 구성하는 하드웨어

다음은 네트워크를 구성하는 하드웨어의 종류이다.

- 허브(Hub): 다수의 컴퓨터를 연결하는 장치로, 한개의 포트에 데이터가 전송되면 모든 포트로 데이터를 전송한다.
  데이터를 모든 포트로 뿌린다는 특징으로 인해 OO단점이 있으며 현재는 스위치로 대체되었다.
- 스위치(Switch): MAC 주소와 포트를 매핑하는 테이블을 내장하고 있어 데이터의 MAC 주소를 확인하고 그에 매핑되는 포트로 데이터를 전달한다.
- 게이트웨이(Gateway): LAN 범위 내에서 데이터 전송을 담당하는 허브와 스위치와 다르게 네트워크 간의 데이터 전송을 담당하는 장치이다.
  IP 주소를 기반으로 패킷을 전송한다.

## 프로토콜

컴퓨터간에 데이터를 주고 받기 위해서는 사전에 어떤 양식으로 데이터를 주고 받을지 미리 약속을 해야 하는데, 이런 약속을 프로토콜(규약)이라고 한다.
프로토콜의 종류로는 HTTP(Hyper Text Transfer Protocol), FTP(File Transfer Protocol), TCP(Transmisson Control Protocol), IP(Internet Protocol) 등이 있다.

인터넷에서 데이터를 주고 받기 위한 프로토콜의 집합을 인터넷 프로토콜 스위트 혹은 TCP/IP 모델이라고 한다.
