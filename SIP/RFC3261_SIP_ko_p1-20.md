# RFC 3261: SIP: Session Initiation Protocol (국문 번역, p.1-20)

> 원문: https://www.rfc-editor.org/rfc/rfc3261.html  
> 번역 범위: 원문 [Page 1] ~ [Page 20]

---

<a id="page-1"></a>
## [Page 1]

Network Working Group  
Request for Comments: 3261  
Obsoletes: [2543](https://www.rfc-editor.org/rfc/rfc2543)  
Category: Standards Track  
June 2002

# SIP: Session Initiation Protocol

## 이 메모의 상태 (Status of this Memo)

이 문서는 Internet 커뮤니티를 위한 standards track 프로토콜을 명시하며, 개선을 위한 토론과 제안을 요청한다.  
이 프로토콜의 표준화 상태와 현황은 "Internet Official Protocol Standards"(STD 1)의 최신 판을 참조하기 바란다.  
이 메모의 배포에는 제한이 없다.

## 저작권 고지 (Copyright Notice)

Copyright (C) The Internet Society (2002). All Rights Reserved.

## 초록 (Abstract)

이 문서는 Session Initiation Protocol (SIP)을 설명한다. SIP은 하나 이상의 참여자와 세션을 생성, 수정, 종료하기 위한 application-layer control (signaling) 프로토콜이다.  
이러한 세션에는 Internet 전화 통화, 멀티미디어 배포, 멀티미디어 회의가 포함된다.

세션 생성을 위한 SIP invitation에는 세션 기술(session description)이 담기며, 이를 통해 참여자들은 상호 호환 가능한 media type 집합에 합의할 수 있다.  
SIP은 proxy server라 불리는 요소를 활용하여 요청을 사용자의 현재 위치로 라우팅하고, 서비스에 대한 사용자 인증 및 권한 부여를 수행하며, provider의 call-routing 정책을 구현하고, 사용자 기능을 제공한다.  
또한 SIP은 registration 기능을 제공하여 사용자가 자신의 현재 위치를 등록(upload)하고, proxy server가 이를 활용할 수 있게 한다.  
SIP은 여러 transport protocol 위에서 동작한다.

---

<a id="page-2"></a>
## [Page 2]

## 목차 (Table of Contents)

아래 항목은 원문 목차를 p.1-20 범위 기준으로 재구성한 것이다.  
현재 문서 내에 실제로 포함된 섹션만 내부 링크를 제공한다.

- [1 Introduction](#section-1) ........................................ 8
- [2 Overview of SIP Functionality](#section-2) ....................... 9
- [3 Terminology](#section-3) ......................................... 10
- [4 Overview of Operation](#section-4) ............................... 10
- [5 Structure of the Protocol](#section-5) ........................... 18
- [6 Definitions](#section-6) ......................................... 20
- 7 SIP Messages ...................................................... 26
- 8 General User Agent Behavior ....................................... 34
- 9 Canceling a Request ............................................... 53
- 10 Registrations .................................................... 56
- 11 Querying for Capabilities ........................................ 66
- 12 Dialogs .......................................................... 69
- 13 Initiating a Session ............................................. 77
- 14 Modifying an Existing Session .................................... 86
- 15 Terminating a Session ............................................ 89
- 16 Proxy Behavior ................................................... 91
- 17 Transactions ..................................................... 122
- 18 Transport ........................................................ 141
- 19 Common Message Components ........................................ 147
- 20 Header Fields .................................................... 159
- 21 Response Codes ................................................... 182
- 22 Usage of HTTP Authentication ..................................... 193
- 23 S/MIME ........................................................... 201
- 24 Examples ......................................................... 213
- 25 Augmented BNF for the SIP Protocol ............................... 219
- 26 Security Considerations .......................................... 232
- 27 IANA Considerations .............................................. 252
- 28 Changes From RFC 2543 ............................................ 255
- 29 Normative References ............................................. 261
- 30 Informative References ........................................... 262
- Appendix A Table of Timer Values .................................... 265

---

<a id="page-3"></a>
## [Page 3]

### 목차 계속 (원문 p.3)

- 8.2.6 Generating the Response ...................................... 49
- 8.2.6.1 Sending a Provisional Response ............................. 49
- 8.2.6.2 Headers and Tags ........................................... 50
- 8.2.7 Stateless UAS Behavior ....................................... 50
- 8.3 Redirect Servers ............................................... 51
- 9 Canceling a Request .............................................. 53
- 9.1 Client Behavior ................................................ 53
- 9.2 Server Behavior ................................................ 55
- 10 Registrations ................................................... 56
- 10.1 Overview ...................................................... 56
- 10.2 Constructing the REGISTER Request ............................. 57
- 10.2.1 Adding Bindings ............................................. 59
- 10.2.1.1 Contact 주소의 Expiration Interval 설정 .................. 60
- 10.2.1.2 Contact 주소 간 선호도 ................................... 61
- 10.2.2 Removing Bindings ........................................... 61
- 10.2.3 Fetching Bindings ........................................... 61
- 10.2.4 Refreshing Bindings ......................................... 61
- 10.2.5 내부 시계 설정 .............................................. 62
- 10.2.6 Registrar 탐색 .............................................. 62
- 10.2.7 Request 전송 ................................................ 62
- 10.2.8 오류 응답 ................................................... 63
- 10.3 REGISTER Request 처리 ......................................... 63
- 11 Querying for Capabilities ....................................... 66
- 11.1 OPTIONS Request 구성 .......................................... 67
- 11.2 OPTIONS Request 처리 .......................................... 68
- 12 Dialogs ......................................................... 69
- 12.1 Dialog 생성 ................................................... 70
- 12.1.1 UAS behavior ................................................ 70
- 12.1.2 UAC behavior ................................................ 71
- 12.2 Dialog 내부 Request ........................................... 72
- 12.2.1 UAC behavior ................................................ 73
- 12.2.1.1 Request 생성 .............................................. 73
- 12.2.1.2 Response 처리 ............................................. 75
- 12.2.2 UAS behavior ................................................ 76
- 12.3 Dialog 종료 ................................................... 77
- 13 Initiating a Session ............................................ 77
- 13.1 Overview ...................................................... 77
- 13.2 UAC Processing ................................................ 78
- 13.2.1 Initial INVITE 생성 ......................................... 78
- 13.2.2 INVITE Response 처리 ........................................ 81

---

<a id="page-4"></a>
## [Page 4]

### 목차 계속 (원문 p.4)

- 13.2.2.1 1xx Responses ............................................. 81
- 13.2.2.2 3xx Responses ............................................. 81
- 13.2.2.3 4xx, 5xx, 6xx Responses ................................... 81
- 13.2.2.4 2xx Responses ............................................. 82
- 13.3 UAS Processing ................................................ 83
- 13.3.1 INVITE 처리 ................................................. 83
- 13.3.1.1 Progress .................................................. 84
- 13.3.1.2 INVITE가 Redirect되는 경우 ................................. 84
- 13.3.1.3 INVITE가 Rejected되는 경우 ................................. 85
- 13.3.1.4 INVITE가 Accepted되는 경우 ................................. 85
- 14 Modifying an Existing Session ................................... 86
- 14.1 UAC Behavior .................................................. 86
- 14.2 UAS Behavior .................................................. 88
- 15 Terminating a Session ........................................... 89
- 15.1 BYE Request로 세션 종료 ....................................... 90
- 15.1.1 UAC Behavior ................................................ 90
- 15.1.2 UAS Behavior ................................................ 91
- 16 Proxy Behavior .................................................. 91
- 16.1 Overview ...................................................... 91
- 16.2 Stateful Proxy ................................................ 92
- 16.3 Request Validation ............................................ 94
- 16.4 Route Information Preprocessing ............................... 96
- 16.5 Request Target 결정 ........................................... 97
- 16.6 Request Forwarding ............................................ 99
- 16.7 Response Processing ........................................... 107
- 16.8 Timer C 처리 .................................................. 114
- 16.9 Transport 오류 처리 ........................................... 115
- 16.10 CANCEL 처리 .................................................. 115
- 16.11 Stateless Proxy .............................................. 116
- 16.12 Proxy Route 처리 요약 ........................................ 118
- 16.12.1 예시 ...................................................... 118
- 16.12.1.1 Basic SIP Trapezoid ...................................... 118
- 16.12.1.2 Strict-Routing Proxy 통과 ................................. 120
- 16.12.1.3 Record-Route Header Field 값 재작성 ...................... 121
- 17 Transactions .................................................... 122
- 17.1 Client Transaction ............................................ 124
- 17.1.1 INVITE Client Transaction ................................... 125

---

<a id="page-5"></a>
## [Page 5]

### 목차 계속 (원문 p.5)

- 17.1.1.1 INVITE Transaction 개요 ................................... 125
- 17.1.1.2 Formal Description ........................................ 125
- 17.1.1.3 ACK Request 구성 .......................................... 129
- 17.1.2 Non-INVITE Client Transaction ............................... 130
- 17.1.2.1 non-INVITE Transaction 개요 ............................... 130
- 17.1.2.2 Formal Description ........................................ 131
- 17.1.3 Response-Client Transaction 매칭 ............................ 132
- 17.1.4 Transport 오류 처리 ......................................... 133
- 17.2 Server Transaction ............................................ 134
- 17.2.1 INVITE Server Transaction ................................... 134
- 17.2.2 Non-INVITE Server Transaction ............................... 137
- 17.2.3 Request-Server Transaction 매칭 ............................. 138
- 17.2.4 Transport 오류 처리 ......................................... 141
- 18 Transport ....................................................... 141
- 18.1 Clients ....................................................... 142
- 18.1.1 Sending Requests ............................................ 142
- 18.1.2 Receiving Responses ......................................... 144
- 18.2 Servers ....................................................... 145
- 18.2.1 Receiving Requests .......................................... 145
- 18.2.2 Sending Responses ........................................... 146
- 18.3 Framing ....................................................... 147
- 18.4 Error Handling ................................................ 147
- 19 Common Message Components ....................................... 147
- 19.1 SIP/SIPS URI .................................................. 148
- 19.1.1 SIP/SIPS URI 구성요소 ....................................... 148
- 19.1.2 문자 Escape 요구사항 ......................................... 152
- 19.1.3 SIP/SIPS URI 예시 ........................................... 153
- 19.1.4 URI 비교 .................................................... 153
- 19.1.5 URI로부터 Request 생성 ...................................... 156
- 19.1.6 SIP URI와 tel URL의 관계 .................................... 157
- 19.2 Option Tags ................................................... 158
- 19.3 Tags .......................................................... 159
- 20 Header Fields ................................................... 159
- 20.1 Accept ........................................................ 161
- 20.2 Accept-Encoding ............................................... 163
- 20.3 Accept-Language ............................................... 164
- 20.4 Alert-Info .................................................... 164
- 20.5 Allow ......................................................... 165
- 20.6 Authentication-Info ........................................... 165
- 20.7 Authorization ................................................. 165
- 20.8 Call-ID ....................................................... 166
- 20.9 Call-Info ..................................................... 166
- 20.10 Contact ...................................................... 167
- 20.11 Content-Disposition .......................................... 168
- 20.12 Content-Encoding ............................................. 169
- 20.13 Content-Language ............................................. 169
- 20.14 Content-Length ............................................... 169
- 20.15 Content-Type ................................................. 170
- 20.16 CSeq ......................................................... 170

---

<a id="page-6"></a>
## [Page 6]

### 목차 계속 (원문 p.6)

- 20.17 Date ......................................................... 170
- 20.18 Error-Info ................................................... 171
- 20.19 Expires ...................................................... 171
- 20.20 From ......................................................... 172
- 20.21 In-Reply-To .................................................. 172
- 20.22 Max-Forwards ................................................. 173
- 20.23 Min-Expires .................................................. 173
- 20.24 MIME-Version ................................................. 173
- 20.25 Organization ................................................. 174
- 20.26 Priority ..................................................... 174
- 20.27 Proxy-Authenticate ........................................... 174
- 20.28 Proxy-Authorization .......................................... 175
- 20.29 Proxy-Require ................................................ 175
- 20.30 Record-Route ................................................. 175
- 20.31 Reply-To ..................................................... 176
- 20.32 Require ...................................................... 176
- 20.33 Retry-After .................................................. 176
- 20.34 Route ........................................................ 177
- 20.35 Server ....................................................... 177
- 20.36 Subject ...................................................... 177
- 20.37 Supported .................................................... 178
- 20.38 Timestamp .................................................... 178
- 20.39 To ........................................................... 178
- 20.40 Unsupported .................................................. 179
- 20.41 User-Agent ................................................... 179
- 20.42 Via .......................................................... 179
- 20.43 Warning ...................................................... 180
- 20.44 WWW-Authenticate ............................................. 182
- 21 Response Codes .................................................. 182
- 21.1 Provisional 1xx ............................................... 182
- 21.1.1 100 Trying .................................................. 183
- 21.1.2 180 Ringing ................................................. 183
- 21.1.3 181 Call Is Being Forwarded ................................ 183
- 21.1.4 182 Queued .................................................. 183
- 21.1.5 183 Session Progress ........................................ 183
- 21.2 Successful 2xx ................................................ 183
- 21.2.1 200 OK ...................................................... 183
- 21.3 Redirection 3xx ............................................... 184
- 21.3.1 300 Multiple Choices ........................................ 184
- 21.3.2 301 Moved Permanently ....................................... 184
- 21.3.3 302 Moved Temporarily ....................................... 184
- 21.3.4 305 Use Proxy ............................................... 185
- 21.3.5 380 Alternative Service ..................................... 185
- 21.4 Request Failure 4xx ........................................... 185
- 21.4.1 400 Bad Request ............................................. 185
- 21.4.2 401 Unauthorized ............................................ 185
- 21.4.3 402 Payment Required ........................................ 186
- 21.4.4 403 Forbidden ............................................... 186
- 21.4.5 404 Not Found ............................................... 186
- 21.4.6 405 Method Not Allowed ...................................... 186
- 21.4.7 406 Not Acceptable .......................................... 186
- 21.4.8 407 Proxy Authentication Required ........................... 186
- 21.4.9 408 Request Timeout ......................................... 186
- 21.4.10 410 Gone ................................................... 187
- 21.4.11 413 Request Entity Too Large ............................... 187
- 21.4.12 414 Request-URI Too Long ................................... 187
- 21.4.13 415 Unsupported Media Type ................................. 187
- 21.4.14 416 Unsupported URI Scheme ................................. 187
- 21.4.15 420 Bad Extension .......................................... 187
- 21.4.16 421 Extension Required ..................................... 188
- 21.4.17 423 Interval Too Brief ..................................... 188
- 21.4.18 480 Temporarily Unavailable ................................ 188
- 21.4.19 481 Call/Transaction Does Not Exist ........................ 188
- 21.4.20 482 Loop Detected .......................................... 188
- 21.4.21 483 Too Many Hops .......................................... 189
- 21.4.22 484 Address Incomplete ..................................... 189

---

<a id="page-7"></a>
## [Page 7]

### 목차 계속 (원문 p.7)

- 21.4.23 485 Ambiguous .............................................. 189
- 21.4.24 486 Busy Here .............................................. 189
- 21.4.25 487 Request Terminated ..................................... 190
- 21.4.26 488 Not Acceptable Here .................................... 190
- 21.4.27 491 Request Pending ........................................ 190
- 21.4.28 493 Undecipherable ......................................... 190
- 21.5 Server Failure 5xx ............................................ 190
- 21.5.1 500 Server Internal Error ................................... 190
- 21.5.2 501 Not Implemented ......................................... 191
- 21.5.3 502 Bad Gateway ............................................. 191
- 21.5.4 503 Service Unavailable ..................................... 191
- 21.5.5 504 Server Time-out ......................................... 191
- 21.5.6 505 Version Not Supported ................................... 192
- 21.5.7 513 Message Too Large ....................................... 192
- 21.6 Global Failures 6xx ........................................... 192
- 21.6.1 600 Busy Everywhere ......................................... 192
- 21.6.2 603 Decline ................................................. 192
- 21.6.3 604 Does Not Exist Anywhere ................................. 192
- 21.6.4 606 Not Acceptable .......................................... 192
- 22 Usage of HTTP Authentication .................................... 193
- 22.1 Framework ..................................................... 193
- 22.2 User-to-User Authentication ................................... 195
- 22.3 Proxy-to-User Authentication .................................. 197
- 22.4 The Digest Authentication Scheme .............................. 199
- 23 S/MIME .......................................................... 201
- 23.1 S/MIME Certificates ........................................... 201
- 23.2 S/MIME Key Exchange ........................................... 202
- 23.3 Securing MIME bodies .......................................... 205
- 23.4 SIP Header Privacy/Integrity with S/MIME ...................... 207
- 23.4.1 SIP Header의 Integrity/Confidentiality 속성 ................. 207
- 23.4.1.1 Integrity ................................................. 207
- 23.4.1.2 Confidentiality ........................................... 208
- 23.4.2 Tunneling Integrity and Authentication ...................... 209
- 23.4.3 Tunneling Encryption ........................................ 211
- 24 Examples ........................................................ 213
- 24.1 Registration .................................................. 213
- 24.2 Session Setup ................................................. 214
- 25 Augmented BNF for the SIP Protocol .............................. 219
- 26 Security Considerations ......................................... 232
- 27 IANA Considerations ............................................. 252
- 28 Changes From RFC 2543 ........................................... 255
- 29 Normative References ............................................ 261
- 30 Informative References .......................................... 262
- Appendix A Table of Timer Values ................................... 265
- Acknowledgments .................................................... 266
- Authors' Addresses ................................................. 267
- Full Copyright Statement ........................................... 269

---

<a id="page-8"></a>
## [Page 8]

<a id="section-1"></a>
## 1 Introduction

Internet에는 세션 생성 및 관리가 필요한 애플리케이션이 매우 많다. 여기서 세션은 참여자들의 연관(association) 사이에서 이루어지는 데이터 교환으로 본다.  
이러한 애플리케이션의 구현은 참여자 사용 방식 때문에 복잡해진다. 사용자는 endpoint 간 이동할 수 있고, 여러 이름으로 식별될 수 있으며, 서로 다른 여러 media를 (때로는 동시에) 사용할 수 있다.  
음성, 비디오, 텍스트 메시지와 같은 다양한 실시간 멀티미디어 세션 데이터를 전달하는 프로토콜이 이미 다수 정의되어 있다.  
Session Initiation Protocol (SIP)은 이러한 프로토콜들과 함께 동작하여, Internet endpoint(즉, user agent)가 서로를 발견하고 공유하고자 하는 세션의 특성에 합의할 수 있게 해준다.

---

<a id="page-9"></a>
## [Page 9]

SIP은 prospective session participant를 찾기 위해, 그리고 그 외의 기능을 위해, user agent가 registration, session invitation, 기타 request를 보낼 수 있는 network host 인프라(proxy server)를 구성할 수 있게 한다.  
SIP은 기저 transport protocol에 독립적이며, 설정되는 세션 종류에 의존하지 않고 세션을 생성, 수정, 종료할 수 있는 민첩하고 범용적인 도구다.

<a id="section-2"></a>
## 2 Overview of SIP Functionality

SIP은 Internet telephony call과 같은 multimedia session(conference)을 설정, 수정, 종료할 수 있는 application-layer control protocol이다.  
또한 multicast conference처럼 이미 존재하는 세션에 참여자를 초대할 수도 있다.  
기존 세션에 media를 추가하거나 제거할 수 있다.  
SIP은 name mapping과 redirection 서비스를 투명하게 지원하고, 이는 personal mobility를 지원한다([27]). 즉 사용자는 네트워크 위치와 무관하게 외부에 단일 식별자를 유지할 수 있다.

SIP은 멀티미디어 통신의 설정과 종료에서 다음 다섯 측면을 지원한다.

- User location: 통신에 사용할 end system의 결정
- User availability: 피호출 party가 통신에 응할 의사가 있는지 판단
- User capabilities: 사용할 media 및 media parameter 결정
- Session setup: "ringing", 그리고 호출자/피호출자 양측의 session parameter 확립
- Session management: 세션 transfer/termination, session parameter 변경, 서비스 호출

SIP은 수직 통합형(vertically integrated) 통신 시스템이 아니다.  
오히려 SIP은 다른 IETF 프로토콜과 결합해 완전한 멀티미디어 아키텍처를 구성하기 위한 구성요소다.  
일반적으로 이러한 아키텍처는 실시간 데이터 전송과 QoS 피드백을 위한 RTP([RFC 1889](https://www.rfc-editor.org/rfc/rfc1889), [28]), 스트리밍 제어를 위한 RTSP([RFC 2326](https://www.rfc-editor.org/rfc/rfc2326), [29]), PSTN gateway 제어를 위한 MEGACO([RFC 3015](https://www.rfc-editor.org/rfc/rfc3015), [30]), 그리고 multimedia session 기술을 위한 SDP([RFC 2327](https://www.rfc-editor.org/rfc/rfc2327), [1]) 등을 포함한다.

---

<a id="page-10"></a>
## [Page 10]

따라서 SIP은 사용자에게 완전한 서비스를 제공하기 위해 다른 프로토콜과 함께 사용되어야 한다.  
다만 SIP의 기본 기능과 동작 자체는 이러한 프로토콜에 의존하지 않는다.

SIP은 서비스 자체를 제공하지는 않는다.  
대신 다양한 서비스를 구현할 수 있는 primitive를 제공한다. 예를 들어 SIP은 사용자를 찾고 opaque object를 사용자의 현재 위치로 전달할 수 있다.  
이 primitive를 SDP session description 전달에 사용하면 endpoint들은 세션 parameter에 합의할 수 있다. 같은 primitive를 세션 설명과 호출자 사진 전송에 쓰면 "caller ID" 서비스도 쉽게 구현된다.  
즉 하나의 primitive가 보통 여러 서비스를 구현하는 데 재사용된다.

SIP은 floor control, voting 같은 conference control 서비스를 제공하지 않으며, conference 관리 방식도 규정하지 않는다.  
SIP은 다른 conference control protocol을 사용하는 세션을 시작하는 데 활용될 수 있다.  
또한 SIP 메시지와 SIP이 설정한 세션은 서로 다른 네트워크를 통과할 수 있으므로, SIP은 네트워크 자원 예약 기능을 제공하지 않는다.

제공되는 서비스의 성격상 보안은 특히 중요하다.  
이를 위해 SIP은 denial-of-service 방지, authentication(사용자-사용자, proxy-사용자), integrity protection, encryption/privacy 서비스를 포함하는 보안 기능군을 제공한다.

SIP은 IPv4와 IPv6를 모두 지원한다.

<a id="section-3"></a>
## 3 Terminology

이 문서에서 "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", "OPTIONAL"은 [BCP 14](https://www.rfc-editor.org/bcp/bcp14), [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) [2]에 정의된 의미로 해석하며, SIP 구현 준수 수준(requirement level)을 나타낸다.

<a id="section-4"></a>
## 4 Overview of Operation

이 절은 간단한 예를 사용해 SIP의 기본 동작을 소개한다.  
튜토리얼 성격의 설명이며, normative statement는 포함하지 않는다.

---

<a id="page-11"></a>
## [Page 11]

첫 번째 예는 SIP의 기본 기능을 보여준다. endpoint 위치 탐색, 통신 의사 전달(signal), 세션 설정을 위한 session parameter 협상, 그리고 세션 종료(teardown)까지를 다룬다.

Figure 1은 사용자 Alice와 Bob 사이에서 이루어지는 전형적인 SIP 메시지 교환을 보여준다.  
(각 메시지는 본문 참조를 위해 "F"+번호로 라벨링되어 있다.)  
이 예에서 Alice는 PC의 SIP 애플리케이션(softphone)을 사용해 Internet을 통해 Bob의 SIP phone으로 통화한다.  
또한 Alice와 Bob을 대신해 세션 설정을 돕는 두 개의 SIP proxy server도 함께 표시되어 있다.  
점선이 만드는 기하학적 형태 때문에, 이 전형적 구조를 흔히 "SIP trapezoid"라고 부른다.

Alice는 Bob의 SIP identity를 사용해 Bob을 "호출"한다. 이는 SIP URI라 불리는 URI 형태다. SIP URI는 [Section 19.1](https://www.rfc-editor.org/rfc/rfc3261.html#section-19.1)에 정의되어 있다.  
형식은 email 주소와 유사하며 보통 username과 host name을 포함한다. 이 예에서는 `sip:bob@biloxi.com`이며, biloxi.com은 Bob의 SIP service provider domain이다.  
Alice의 SIP URI는 `sip:alice@atlanta.com`이다. Alice는 Bob의 URI를 직접 입력했거나 hyperlink 혹은 address book 항목을 클릭했을 수 있다.

SIP은 secure URI인 SIPS URI도 제공한다. 예: `sips:bob@biloxi.com`.  
SIPS URI로 시작된 call은 호출자에서 피호출자 domain까지 모든 SIP 메시지 전송에 secure/encrypted transport(TLS)가 사용됨을 보장한다.  
그 이후 피호출자까지의 보안 전달 방식은 피호출자 domain policy에 따른다.

SIP은 HTTP 유사 request/response transaction 모델을 기반으로 한다.  
각 transaction은 서버의 특정 method/function을 호출하는 request와, 그에 대한 하나 이상의 response로 구성된다.  
이 예에서 transaction은 Alice softphone이 Bob의 SIP URI로 INVITE request를 보내며 시작된다. INVITE는 요청자(Alice)가 서버(Bob)에게 수행을 요구하는 동작을 나타내는 SIP method의 예다.  
INVITE request에는 여러 header field가 포함된다. header field는 메시지에 대한 부가 정보를 전달하는 이름 있는 attribute다.

---

<a id="page-12"></a>
## [Page 12]

다음은 Figure 1과 INVITE 예시이다.

```text
                     atlanta.com  . . . biloxi.com
                 .      proxy              proxy     .
               .                                       .
       Alice's  . . . . . . . . . . . . . . . . . . . .  Bob's
      softphone                                        SIP Phone
         |                |                |                |
         |    INVITE F1   |                |                |
         |--------------->|    INVITE F2   |                |
         |  100 Trying F3 |--------------->|    INVITE F4   |
         |<---------------|  100 Trying F5 |--------------->|
         |                |<-------------- | 180 Ringing F6 |
         |                | 180 Ringing F7 |<---------------|
         | 180 Ringing F8 |<---------------|     200 OK F9  |
         |<---------------|    200 OK F10  |<---------------|
         |    200 OK F11  |<---------------|                |
         |<---------------|                |                |
         |                       ACK F12                    |
         |------------------------------------------------->|
         |                   Media Session                  |
         |<================================================>|
         |                       BYE F13                    |
         |<-------------------------------------------------|
         |                     200 OK F14                   |
         |------------------------------------------------->|
         |                                                  |

         Figure 1: SIP session setup example with SIP trapezoid
```

```text
INVITE sip:bob@biloxi.com SIP/2.0
Via: SIP/2.0/UDP pc33.atlanta.com;branch=z9hG4bK776asdhds
Max-Forwards: 70
To: Bob <sip:bob@biloxi.com>
From: Alice <sip:alice@atlanta.com>;tag=1928301774
Call-ID: a84b4c76e66710@pc33.atlanta.com
CSeq: 314159 INVITE
Contact: <sip:alice@pc33.atlanta.com>
Content-Type: application/sdp
Content-Length: 142

(Alice's SDP not shown)
```

텍스트 인코딩 메시지의 첫 줄에는 method name(INVITE)가 들어간다. 이어지는 줄은 header field 목록이다. 위 예시는 최소 필수 집합을 담고 있다.

---

<a id="page-13"></a>
## [Page 13]

각 header field의 의미는 다음과 같다.

- Via: Alice가 이 request에 대한 response를 받길 기대하는 주소(`pc33.atlanta.com`)를 담고, transaction 식별용 branch parameter를 포함한다.
- To: display name(Bob)과 request가 원래 향하던 SIP/SIPS URI(`sip:bob@biloxi.com`)를 담는다. display name은 [RFC 2822](https://www.rfc-editor.org/rfc/rfc2822) [3]에 설명된다.
- From: 요청 발신자(Alice)를 나타내는 display name과 SIP/SIPS URI(`sip:alice@atlanta.com`)를 담는다. 또한 softphone이 추가한 임의 문자열 tag(`1928301774`)를 포함하며, 식별에 사용된다.
- Call-ID: 무작위 문자열과 softphone host name 또는 IP address 조합으로 만든 전역 유일 식별자다.
- CSeq(Command Sequence): 정수와 method name으로 구성된다. dialog 내 새 request마다 CSeq number가 증가한다.
- Contact: Alice에 직접 도달할 수 있는 SIP/SIPS URI를 담는다. 보통 FQDN 기반 username 형태이며, FQDN이 없으면 IP address도 허용된다.
- Max-Forwards: request가 목적지까지 거칠 수 있는 hop 수 제한. 각 hop마다 1 감소한다.
- Content-Type: message body 설명을 담는다.
- Content-Length: message body의 octet(byte) 길이를 담는다.

SIP header field 전체 집합은 [Section 20](https://www.rfc-editor.org/rfc/rfc3261.html#section-20)에 정의되어 있다.

세션 상세(예: media type, codec, sampling rate)는 SIP 자체가 아닌 다른 포맷의 body로 기술된다.  
대표적으로 Session Description Protocol(SDP, [RFC 2327](https://www.rfc-editor.org/rfc/rfc2327) [1])가 사용된다.

---

<a id="page-14"></a>
## [Page 14]

softphone은 Bob의 위치나 biloxi.com domain의 SIP server 위치를 모르기 때문에, INVITE를 Alice domain(atlanta.com)을 담당하는 SIP server로 보낸다.  
atlanta.com SIP server 주소는 Alice softphone에 미리 설정되었거나, 예를 들어 DHCP로 발견될 수 있다.

atlanta.com SIP server는 proxy server다.  
proxy server는 SIP request를 받아 requestor를 대신해 전달(forward)한다.  
이 예에서 proxy server는 INVITE를 수신하고 Alice softphone으로 100 (Trying) response를 돌려준다. 이는 INVITE 수신 및 목적지 라우팅 처리 중임을 의미한다.

SIP response는 3자리 code와 설명 phrase로 구성된다. 이 response에는 INVITE와 동일한 To, From, Call-ID, CSeq, 그리고 Via의 branch parameter가 포함되어 Alice softphone이 응답을 INVITE와 연관지을 수 있다.

atlanta.com proxy는 biloxi.com proxy를 찾는다. 이는 biloxi.com domain을 담당하는 SIP server를 찾기 위한 DNS lookup을 통해 이뤄질 수 있다([4]).  
그 결과 biloxi.com proxy의 IP address를 얻어 INVITE를 그쪽으로 proxying한다. 전달 전에 atlanta.com proxy는 자신의 주소를 담은 Via 값을 하나 더 추가한다(원래 INVITE에는 Alice 주소의 Via가 있음).

biloxi.com proxy는 INVITE를 수신하고 atlanta.com proxy로 100 (Trying)을 보낸다. 이어 location service라 불리는 데이터베이스를 조회해 Bob의 현재 IP 주소를 찾는다.  
(이 데이터베이스가 어떻게 채워지는지는 다음 절에서 설명된다.) biloxi.com proxy는 자신의 Via를 추가한 뒤 INVITE를 Bob의 SIP phone으로 전달한다.

Bob의 SIP phone은 INVITE를 수신하고 Bob에게 수신 호출을 알린다(전화가 울림).  
이 상태를 180 (Ringing) response로 나타내며, 응답은 두 proxy를 거쳐 역방향으로 라우팅된다. 각 proxy는 Via를 사용해 다음 hop을 결정하고 자신 항목을 상단에서 제거한다.  
그 결과 초기 INVITE 라우팅에는 DNS/location lookup이 필요했지만, 180 (Ringing)은 추가 lookup이나 proxy state 유지 없이도 호출자에게 반환될 수 있다.

---

<a id="page-15"></a>
## [Page 15]

이 방식은 INVITE를 본 각 proxy가 해당 INVITE의 모든 response도 보게 된다는 장점이 있다.

Alice softphone이 180 (Ringing)을 받으면 ringback tone을 재생하거나 화면 메시지를 표시하는 식으로 Alice에게 알려준다.

이 예에서 Bob이 수신을 결정하면 handset을 들면서 200 (OK) response를 보낸다. 이는 call 응답 완료를 의미한다.  
200 (OK)에는 Bob이 Alice와 수립할 의사가 있는 세션 유형을 담은 SDP media description이 body에 포함된다.

즉 SDP는 2단계 교환이다. Alice가 Bob에게 하나를 보내고, Bob이 Alice에게 하나를 보낸다.  
이는 offer/answer 모델 기반의 기본 협상 기능을 제공한다. Bob이 응답을 원치 않거나 통화 중이라면 200 (OK) 대신 오류 응답이 전송되어 media session은 수립되지 않는다.  
SIP response code 전체 목록은 [Section 21](https://www.rfc-editor.org/rfc/rfc3261.html#section-21)에 있다.

Bob이 내보내는 200 (OK) 예시는 다음과 같다.

```text
SIP/2.0 200 OK
Via: SIP/2.0/UDP server10.biloxi.com
   ;branch=z9hG4bKnashds8;received=192.0.2.3
Via: SIP/2.0/UDP bigbox3.site3.atlanta.com
   ;branch=z9hG4bK77ef4c2312983.1;received=192.0.2.2
Via: SIP/2.0/UDP pc33.atlanta.com
   ;branch=z9hG4bK776asdhds ;received=192.0.2.1
To: Bob <sip:bob@biloxi.com>;tag=a6c85cf
From: Alice <sip:alice@atlanta.com>;tag=1928301774
Call-ID: a84b4c76e66710@pc33.atlanta.com
CSeq: 314159 INVITE
Contact: <sip:bob@192.0.2.4>
Content-Type: application/sdp
Content-Length: 131

(Bob's SDP not shown)
```

응답 첫 줄은 response code(200)와 reason phrase(OK)를 포함한다.  
Via, To, From, Call-ID, CSeq는 INVITE에서 복사된다. Bob phone은 To에 tag parameter를 추가한다.

---

<a id="page-16"></a>
## [Page 16]

이 tag는 양 endpoint의 dialog에 포함되고, 이후 이 call의 모든 request/response에 사용된다.  
Contact는 Bob의 SIP phone에 직접 도달 가능한 URI를 담는다. Content-Type과 Content-Length는 Bob의 SDP body(생략됨)를 가리킨다.

이 예의 DNS/location lookup 외에도 proxy server는 request 전달 대상을 유연하게 결정할 수 있다.  
예를 들어 Bob phone이 486 (Busy Here)를 반환하면 biloxi.com proxy는 INVITE를 Bob의 voicemail server로 보낼 수 있다.  
또한 여러 위치로 동시에 INVITE를 보내는 parallel search(forking)도 가능하다.

이 경우 200 (OK)는 두 proxy를 거쳐 Alice softphone으로 전달되고, softphone은 ringback tone을 멈추고 통화 연결을 표시한다.  
마지막으로 Alice softphone은 최종 응답(200 (OK)) 수신 확인을 위해 Bob phone에 ACK를 보낸다.  
이 예에서 ACK는 두 proxy를 우회해 Alice에서 Bob으로 직접 전송된다.

이는 INVITE/200 (OK) 교환을 통해 endpoint들이 서로의 Contact 주소를 알게 되었기 때문이다. 초기 INVITE 시점에는 이 정보가 없었다.  
proxy들이 수행했던 lookup은 더 이상 필요 없어지고, proxy는 call flow에서 빠진다.  
이로써 SIP session 설정에 사용되는 INVITE/200/ACK 3-way handshake가 완료된다. 상세는 [Section 13](https://www.rfc-editor.org/rfc/rfc3261.html#section-13)에 있다.

Alice와 Bob의 media session이 시작되면, 양측은 SDP 협상으로 합의한 포맷으로 media packet을 주고받는다.  
일반적으로 end-to-end media packet 경로는 SIP signaling 메시지 경로와 다를 수 있다.

세션 중 어느 쪽이든 media session 특성을 바꿀 수 있으며, 새 media description을 담은 re-INVITE로 이를 수행한다.  
이 re-INVITE는 기존 dialog를 참조하므로 상대는 새 세션 생성이 아니라 기존 세션 수정임을 알 수 있다.  
상대가 수락하면 200 (OK), 요청자는 ACK로 응답한다. 수락하지 않으면 488 (Not Acceptable Here) 같은 오류 응답을 보내고 이 역시 ACK를 받는다.  
하지만 re-INVITE 실패가 기존 call 실패를 의미하지는 않는다. 기존 협상 특성으로 세션은 계속된다. 상세는 [Section 14](https://www.rfc-editor.org/rfc/rfc3261.html#section-14)에 있다.

---

<a id="page-17"></a>
## [Page 17]

통화 종료 시 Bob이 먼저 끊으면 BYE 메시지를 생성한다.  
BYE는 proxy를 우회해 Alice softphone으로 직접 전달된다.  
Alice는 200 (OK)로 BYE 수신을 확인하고, 세션과 BYE transaction이 종료된다.

BYE의 200 (OK)에는 ACK를 보내지 않는다. ACK는 INVITE request에 대한 response에만 사용된다.  
INVITE의 특수 처리 이유는 SIP의 신뢰성 메커니즘, ringing 응답까지 걸리는 시간, forking과 관련되며 이후 절에서 설명된다.  
이 때문에 SIP의 request 처리는 보통 INVITE와 non-INVITE(즉 INVITE 외 모든 method)로 나누어 설명된다. 세션 종료 상세는 [Section 15](https://www.rfc-editor.org/rfc/rfc3261.html#section-15)를 참조하라.

[Section 24.2](https://www.rfc-editor.org/rfc/rfc3261.html#section-24.2)는 Figure 1 메시지를 전체 형태로 설명한다.

일부 경우에는 세션 기간 동안 endpoint 간 모든 SIP 메시징을 signaling path 상의 proxy가 계속 보는 것이 유용하다.  
예를 들어 biloxi.com proxy가 초기 INVITE 이후에도 경로에 남고 싶다면, 자신 호스트명/IP로 resolve되는 URI를 담은 Record-Route header field를 INVITE에 추가한다.

이 정보는 Bob phone과 Alice softphone(200 (OK)로 Record-Route가 되돌아오기 때문) 모두가 받아 dialog 기간 동안 저장한다.  
그러면 biloxi.com proxy는 ACK, BYE, BYE에 대한 200 (OK)까지 수신 및 중계하게 된다.  
각 proxy는 후속 메시지 수신 여부를 독립적으로 결정할 수 있고, 수신하기로 한 proxy들을 통해 메시지가 통과한다. 이는 mid-call feature 제공 proxy에서 자주 활용된다.

Registration도 SIP의 일반적인 동작이다. Registration은 biloxi.com server가 Bob의 현재 위치를 학습하는 방법 중 하나다.

---

<a id="page-18"></a>
## [Page 18]

초기화 시점과 주기적 간격으로 Bob의 SIP phone은 biloxi.com domain의 SIP registrar에 REGISTER를 보낸다.  
REGISTER는 Bob의 SIP/SIPS URI(`sip:bob@biloxi.com`)를 현재 로그인된 장치(Contact header field의 SIP/SIPS URI)와 연결한다.

registrar는 이 연관(=binding)을 location service DB에 기록하며, biloxi.com domain proxy가 이를 라우팅에 활용한다.  
많은 경우 registrar와 proxy는 같은 물리 서버에 공존한다. 중요한 점은 SIP server 유형 구분이 물리적이 아니라 논리적이라는 것이다.

Bob은 단일 device에서만 등록할 필요가 없다. 예를 들어 집과 사무실 SIP phone 모두 등록할 수 있다.  
이 정보는 location service에 함께 저장되며, proxy는 다양한 검색으로 Bob을 찾을 수 있다.  
유사하게 하나의 device에 여러 사용자가 동시에 등록될 수도 있다.

location service는 추상 개념이다. 일반적으로 proxy가 URI를 넣으면 request 전달 대상이 되는 0개 이상의 URI 집합을 반환하는 정보를 담는다.  
Registration은 이 정보 생성 방법 중 하나일 뿐 유일한 방법은 아니다. 관리자는 임의 매핑 함수를 구성할 수 있다.

마지막으로 SIP에서 registration은 inbound SIP request 라우팅에 사용되며, outbound request 권한 부여에는 쓰이지 않는다는 점이 중요하다.  
SIP의 authorization/authentication은 request별 challenge/response 메커니즘 또는 [Section 26](https://www.rfc-editor.org/rfc/rfc3261.html#section-26)에서 논의하는 하위 계층 방식을 사용한다.

이 registration 예시의 전체 메시지 상세는 [Section 24.1](https://www.rfc-editor.org/rfc/rfc3261.html#section-24.1)에 있다.

또한 SIP의 다른 동작들(예: OPTIONS로 서버/클라이언트 capability 조회, CANCEL로 pending request 취소)은 뒤 절에서 소개된다.

<a id="section-5"></a>
## 5 Structure of the Protocol

SIP은 layered protocol 구조를 가진다. 즉 각 단계 간 결합도가 느슨한 비교적 독립적인 처리 단계 집합으로 동작을 설명한다.  
이 레이어링은 설명을 위한 것으로, 여러 요소에 공통된 기능을 한곳에 서술하기 위함이다. 구현 방식을 강제하지 않는다.  
어떤 요소가 특정 레이어를 "포함한다"고 말할 때, 이는 해당 레이어가 정의한 규칙 집합을 준수한다는 뜻이다.

모든 프로토콜 요소가 모든 레이어를 포함하는 것은 아니다.  
또한 SIP의 요소는 물리 요소가 아니라 논리 요소다.  
하나의 물리 구현이 여러 논리 요소로 동작할 수 있고, transaction마다 역할을 달리할 수도 있다.

SIP의 최하위 레이어는 syntax/encoding 레이어다.  
인코딩은 확장 Backus-Naur Form(BNF)으로 명시된다.  
전체 BNF는 [Section 25](https://www.rfc-editor.org/rfc/rfc3261.html#section-25)에, SIP 메시지 구조 개요는 [Section 7](https://www.rfc-editor.org/rfc/rfc3261.html#section-7)에 있다.

---

<a id="page-19"></a>
## [Page 19]

두 번째 레이어는 transport 레이어다.  
클라이언트가 request를 보내고 response를 받는 방법, 서버가 request를 받고 response를 보내는 방법을 네트워크 관점에서 정의한다.  
모든 SIP 요소는 transport 레이어를 포함한다. transport 레이어는 [Section 18](https://www.rfc-editor.org/rfc/rfc3261.html#section-18)에 설명되어 있다.

세 번째 레이어는 transaction 레이어다. transaction은 SIP의 핵심 구성요소다.  
transaction은 client transaction이 transport 레이어를 통해 server transaction으로 보낸 request와, 그 request에 대한 모든 response를 포함한다.  
transaction 레이어는 application-layer retransmission, response-request 매칭, application-layer timeout을 처리한다.  
UAC가 수행하는 어떤 작업도 일련의 transaction으로 이뤄진다. transaction은 [Section 17](https://www.rfc-editor.org/rfc/rfc3261.html#section-17)에 논의되어 있다.

User agent와 stateful proxy는 transaction 레이어를 포함한다. stateless proxy는 transaction 레이어를 포함하지 않는다.  
transaction 레이어는 client component(client transaction)와 server component(server transaction)로 구성되며, 각각 특정 request 처리를 위한 finite state machine으로 표현된다.

transaction 레이어 위 레이어를 transaction user(TU)라 한다. stateless proxy를 제외한 모든 SIP entity는 TU다.  
TU가 request를 보내려면 client transaction 인스턴스를 만들고, request와 함께 목적지 IP/port/transport를 전달한다. client transaction을 만든 TU는 이를 cancel할 수도 있다.  
client가 transaction을 cancel하면 server가 추가 처리를 중단하고, transaction 이전 상태로 되돌아가며, 해당 transaction에 대해 특정 오류 응답을 생성하도록 요청한다.  
이는 CANCEL request로 수행되며, CANCEL 자체도 독립 transaction이지만 취소 대상 transaction을 참조한다([Section 9](https://www.rfc-editor.org/rfc/rfc3261.html#section-9)).

SIP 요소(UAC/UAS, stateless/stateful proxy, registrar)는 서로를 구분하는 core를 가진다. stateless proxy를 제외한 core는 TU다.  
UAC/UAS core 동작은 method에 따라 달라지지만, 모든 method에 공통인 규칙도 있다([Section 8](https://www.rfc-editor.org/rfc/rfc3261.html#section-8)).  
UAC에는 request 생성 규칙, UAS에는 request 처리 및 response 생성 규칙이 적용된다.

---

<a id="page-20"></a>
## [Page 20]

Registration이 SIP에서 중요하므로, REGISTER를 처리하는 UAS는 registrar라는 특별한 이름으로 부른다.  
[Section 10](https://www.rfc-editor.org/rfc/rfc3261.html#section-10)은 REGISTER method의 UAC/UAS core 동작을, [Section 11](https://www.rfc-editor.org/rfc/rfc3261.html#section-11)은 UA capability 확인에 쓰이는 OPTIONS method의 UAC/UAS core 동작을 설명한다.

일부 request는 dialog 내부에서 전송된다. dialog는 두 user agent 사이에 일정 시간 지속되는 peer-to-peer SIP 관계다.  
dialog는 메시지 순서 제어와 user agent 간 올바른 request 라우팅을 돕는다.  
이 명세에서 dialog를 설정하는 방법은 INVITE method뿐이다.  
UAC가 dialog 문맥 내 request를 보낼 때는 [Section 8](https://www.rfc-editor.org/rfc/rfc3261.html#section-8)의 공통 UAC 규칙과 mid-dialog request 규칙을 함께 따른다.  
[Section 12](https://www.rfc-editor.org/rfc/rfc3261.html#section-12)는 dialog 생성/유지 절차와 dialog 내 request 생성 절차를 다룬다.

SIP에서 가장 중요한 method는 INVITE이며, 참여자 간 session 수립에 사용된다. session은 통신 목적의 참여자 집합과 그 사이 media stream의 모음이다.  
[Section 13](https://www.rfc-editor.org/rfc/rfc3261.html#section-13)은 session initiation(결과적으로 하나 이상의 SIP dialog 생성)을, [Section 14](https://www.rfc-editor.org/rfc/rfc3261.html#section-14)은 dialog 내 INVITE를 통한 session 특성 수정 방법을, [Section 15](https://www.rfc-editor.org/rfc/rfc3261.html#section-15)은 session termination을 다룬다.

Section 8, 10, 11, 12, 13, 14, 15 절차는 UA core에 집중한다.  
(Section 9의 cancellation은 UA core와 proxy core 모두에 적용된다.)  
[Section 16](https://www.rfc-editor.org/rfc/rfc3261.html#section-16)은 user agent 간 메시지 라우팅을 담당하는 proxy 요소를 설명한다.

<a id="section-6"></a>
## 6 Definitions

다음 용어들은 SIP에서 특별한 의미를 가진다.

- Address-of-Record: AOR은 SIP/SIPS URI로서, location service를 가진 domain을 가리키며, 해당 service는 이 URI를 사용자가 실제로 reachable할 수 있는 다른 URI로 매핑할 수 있다. 일반적으로 location service는 registration으로 채워진다. AOR은 흔히 사용자의 "public address"로 간주된다.
- Back-to-Back User Agent: B2BUA는 request를 받아 UAS로 처리하는 논리 엔티티다. 응답 방식을 결정하기 위해 UAC로 동작해 request를 생성한다. proxy와 달리 dialog state를 유지하며 자신이 수립한 dialog의 모든 request에 반드시 참여한다. UAC와 UAS의 결합체이므로 별도의 동작 정의가 추가로 필요하지 않다.

---

## 번역 메모

- p.1~20 범위를 우선 번역했다.
- SIP method/헤더/고유 용어(INVITE, ACK, BYE, CANCEL, REGISTER, OPTIONS, Via, Contact, Call-ID 등)는 원문 표기를 유지했다.
- 내부 링크는 현재 문서 내에 존재하는 대상(Section 1~6, Page 1~20)에 대해서만 동작하도록 구성했다.
