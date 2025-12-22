## 설명 ##
- OSPF에서 해당 장비를 '**가능하면 사용하지 말아라**' 상태로 만드는 기능
- neighbor/interface down 없이 LSA 갱신으로 최적 경로(SPF)를 계산하는 방식
- neighbor에게 전달하는 lsa 정보에 metric을 높여서 전달하며, 이를 받은 neighbor는 해당 경로를 가장 낮은 우선순위로 함
- neighbor는 해당 lsa 정보를 토대로 라우팅 계산을 하며, 라우팅 테이블에는 해당 lsa를 제외한 최적 경로만 인스톨함

## metric ##
- metric은 cost 를 의미하며, 최대값은 다음과 같음
	- router-lsa :  65535(16bit)
	- include-stub : 65535
	- external-lsa : 1-16777215(default : 16711680) (24bit)
	- summary-lsa : 1-16777215(default : 16711680)

## 설정 ##
- ospf process에서 설정할 수 있으며 네트워크 상황에 맞는 옵션 적용 가능
- 적용된 추가 옵션(external-lsa, include-stub, 등) 설정 삭제시 router-lsa는 설정에 남아 있을 수 있음(확인 후 삭제 필요)
```
(config-router-ospf)#max-metric router-lsa ?
  external-lsa  Override external-lsa metric with
  include-stub  Set maximum metric for stub links in router-LSAs
  on-startup    Set maximum metric temporarily after reboot
  summary-lsa   Override summary-lsa metric with max-metric value
```

## 옵션별 설명 ##
### router-lsa ###
- router lsa(type-1)의 metric의 cost를 max로 변경
	- 다른 스위치/라우터와 연결된 링크
	- OSPF peering 인터페이스를 의미하며 transit network & point-to-point network가 포함됨
	- stub network는 해당하지 않음
	- 해당 스위치를 목적지로하는 트래픽은 허용하지만, 경유하는 트래픽(transit)은 허용하지 않음
- 설정
```
router ospf 1
   max-metric router-lsa
```
- router-lsa 적용 전
```
#show ip ospf database router 172.17.100.1

Link connected to: a Point-to-point Network
(Link ID) Neighboring Router ID: 172.17.100.4
(Link Data) 172.17.0.2
Number of TOS metrics: 0
TOS 0 Metrics: 10

Link connected to: a Transit Network
(Link ID) Designated Router address: 172.17.0.1
(Link Data) Router Interface address: 172.17.0.0
Number of TOS metrics: 0
TOS 0 Metrics: 10
```
- router-lsa 적용 후 metric이 변경된 것을 확인 할 수 있음
```
#show ip ospf database router 172.17.100.1

Number of Links: 6
Link connected to: a Point-to-point Network
(Link ID) Neighboring Router ID: 172.17.100.4
(Link Data) 172.17.0.2
Number of TOS metrics: 0
TOS 0 Metrics: 65535

Link connected to: a Transit Network
(Link ID) Designated Router address: 172.17.0.1
(Link Data) Router Interface address: 172.17.0.0
Number of TOS metrics: 0
TOS 0 Metrics: 65535
```

### include-stub ###
- stub 경로까지 모두 metric의 cost를 max로 변경
	- router-lsa에 추가로 해당 스위치가 목적지인 네트워크까지 포함
	- 해당 스위치를 목적지로하는 트래픽과 경유하는 트래픽(transit)을 모두 허용하지 않음
- 설정
```
router ospf 1
   max-metric router-lsa include-stub
```
- include-stub 적용 전
```
#show ip ospf database router adv-router 172.17.100.1
    Link connected to: a Point-to-point Network
     (Link ID) Neighboring Router ID: 172.17.100.4
     (Link Data)  172.17.0.2
      Number of TOS metrics: 0
       TOS 0 Metrics: 10

    Link connected to: a Stub Network
     (Link ID) Network/subnet number: 11.1.1.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metrics: 10

    Link connected to: a Transit Network
     (Link ID) Designated Router address: 172.17.0.1
     (Link Data) Router Interface address: 172.17.0.0
      Number of TOS metrics: 0
       TOS 0 Metrics: 10

#show ip route 11.1.1.0/24
 O        11.1.1.0/24 [110/20]
           via 172.17.0.0, Ethernet11
           via 172.17.1.0, Ethernet12
```

- include-stub 적용 후 metric이 변경된 것을 확인 할 수 있음
```
#show ip ospf database router adv-router 172.17.100.1
    Link connected to: a Point-to-point Network
     (Link ID) Neighboring Router ID: 172.17.100.4
     (Link Data)  172.17.0.2
      Number of TOS metrics: 0
       TOS 0 Metrics: 65535

    Link connected to: a Stub Network
     (Link ID) Network/subnet number: 11.1.1.0
     (Link Data) Network Mask: 255.255.255.0
      Number of TOS metrics: 0
       TOS 0 Metrics: 65535

    Link connected to: a Transit Network
     (Link ID) Designated Router address: 172.17.0.1
     (Link Data) Router Interface address: 172.17.0.0
      Number of TOS metrics: 0
       TOS 0 Metrics: 65535


---
#show ip route 11.1.1.0/24
 O        11.1.1.0/24 [110/20]
           via 172.17.1.0, Ethernet12

```

### external-lsa ###
- external lsa(type-5)의 metric cost를 max로 변경
	- 재분배된 네트워크(redistribute)
- 설정
```
router ospf 1
   max-metric router-lsa external-lsa
```
- external-lsa 적용 전
	- 10.1.1.0/24 -> redistribute connect
	- 172.17.200.0/24 -> redistribute static
```
#show ip ospf database external adv-router 172.17.100.1
  LS Age: 39
  Options: (E DC)
  LS Type: AS External Links
  Link State ID: 172.17.200.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x80000014
  Checksum: 0x53a1
  Length: 36
  Network Mask: 255.255.255.0
        Metric Type: 2
        Metric: 1
        Forwarding Address: 0.0.0.0
        External Route Tag: 0
  LS Age: 39
  Options: (E DC)
  LS Type: AS External Links
  Link State ID: 10.1.1.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x80000019
  Checksum: 0xe188
  Length: 36
  Network Mask: 255.255.255.0
        Metric Type: 2
        Metric: 1
        Forwarding Address: 0.0.0.0
        External Route Tag: 0


---
#show ip route 172.17.200.0/24
 O E2     172.17.200.0/24 [110/1]
           via 172.17.0.0, Ethernet11
           via 172.17.1.0, Ethernet12

#show ip route 10.1.1.0/24
 O E2     10.1.1.0/24 [110/1]
           via 172.17.0.0, Ethernet11
           via 172.17.1.0, Ethernet12
```
- external-lsa 적용 후 metric이 변경된 것을 확인 할 수 있음
	- 10.1.1.0/24 -> redistribute connect
	- 172.17.200.0/24 -> redistribute static
```
#show ip ospf database external adv-router 172.17.100.1
  LS Age: 5
  Options: (E DC)
  LS Type: AS External Links
  Link State ID: 172.17.200.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x80000015
  Checksum: 0x47ad
  Length: 36
  Network Mask: 255.255.255.0
        Metric Type: 2
        Metric: 16711680
        Forwarding Address: 0.0.0.0
        External Route Tag: 0
  LS Age: 5
  Options: (E DC)
  LS Type: AS External Links
  Link State ID: 10.1.1.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x8000001a
  Checksum: 0xd594
  Length: 36
  Network Mask: 255.255.255.0
        Metric Type: 2
        Metric: 16711680
        Forwarding Address: 0.0.0.0
        External Route Tag: 0


---
#show ip route 172.17.200.0/24
 O E2     172.17.200.0/24 [110/1]
           via 172.17.1.0, Ethernet12

#show ip route 10.1.1.0/24
 O E2     10.1.1.0/24 [110/1]
           via 172.17.1.0, Ethernet12
```

### summary-lsa ###
- summary lsa(type-3)의 metric cost를 max로 변경
	- 다른 에어리어 네트워크(ABR)
- 설정
```
router ospf 1
   max-metric router-lsa summary-lsa
```
- summary-lsa 적용 전
```
#show ip ospf database summary adv-router 172.17.100.1
  LS Age: 234
  Options: (E DC)
  LS Type: Summary Links
  Link State ID: 12.1.1.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x80000015
  Checksum: 0xb23b
  Length: 28
    Mask: 255.255.255.0
    Metric: 10


---
#show ip route 12.1.1.0
 O IA     12.1.1.0/24 [110/20]
           via 172.17.0.0, Ethernet11
           via 172.17.1.0, Ethernet12
```
- summary-lsa 적용 후 metric이 변경되었음을 확인할 수 있음
```
#show ip ospf database summary adv-router 172.17.100.1
  LS Age: 4
  Options: (E DC)
  LS Type: Summary Links
  Link State ID: 12.1.1.0
  Advertising Router: 172.17.100.1
  LS Seq Number: 0x80000016
  Checksum: 0x4caa
  Length: 28
    Mask: 255.255.255.0
    Metric: 16711680


---
#show ip route 12.1.1.0
 O IA     12.1.1.0/24 [110/20]
           via 172.17.1.0, Ethernet12
```

### on-startup ###
- reboot 후 일정 시간동안 max-metric의 동작을 유지하며, 타이머 만료 후 자동으로 default metric으로 lsa를 다시 업데이트 함
- 타이머 시간 설정 후 
- 설정
```
(config-router-ospf)#max-metric router-lsa on-startup 600 ?
  external-lsa  Override external-lsa metric with
  include-stub  Set maximum metric for stub links in router-LSAs
  summary-lsa   Override summary-lsa metric with max-metric value
```

### wait-for-bgp ###
- reboot 후 BGP 상태가 안정 될때까지 max-metric을 유지
- BGP가 안정될때까지(BGP up, RIB 안정화, 등) 해당 스위치를 OSPF 경로로 사용 금지
	- BGP가 안정화 될때까지 max-metric을 유지할 옵션들 선택 가능
```
(config-router-ospf)#max-metric router-lsa on-startup wait-for-bgp ?
  external-lsa  Override external-lsa metric with
  include-stub  Set maximum metric for stub links in router-LSAs
  summary-lsa   Override summary-lsa metric with max-metric value
```


## 적용 후 확인 방법 ##
- OSPF database에서 cost가 max로 변경되었는지 확인
	- router-lsa / include-stub
		- show ip ospf database router ~~~
	- external-lsa
		- show ip ospf database external ~~~
	- summary-lsa
		- show ip ospf databse summary ~~~
	- 
- 라우팅 테이블에서 best path가 변경되었는지 확인
	- show ip route IP주소
- 트래픽이 best path로만 경유하는지 확인
	- show interface counters rates

