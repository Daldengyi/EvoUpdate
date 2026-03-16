# EvoParsingD & EvoRelay

에볼루션 게임 사이트를 자동으로 접속·파싱하고, 여러 인스턴스를 **EvoRelay** 서버가 제어하는 구조입니다.  
**EvoParsingD**는 클라이언트(Windows WinForms + WebView2), **EvoRelay**는 제어·배정 서버(C# WinForms + Kestrel WebSocket)입니다.

---

## 목차

- [전체 구조](#전체-구조)
- [EvoRelay (제어 서버)](#evorelay-제어-서버)
- [EvoParsingD (클라이언트)](#evoparsingd-클라이언트)
- [EvoRelay와 EvoParsingD 동작 흐름](#evorelay와-evoparsingd-동작-흐름)
- [데이터 릴레이 (EvoRelayNode)](#데이터-릴레이-evorelaynode)
- [빌드 및 실행](#빌드-및-실행)
- [설정 파일](#설정-파일)

---

## 전체 구조

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    EvoRelay (제어 서버)                    │
                    │  포트 8765 · WinForms UI + Kestrel WebSocket               │
                    │  /control → 인스턴스 배정·상태  /lock → 재진입 잠금         │
                    │  lobby.configs.json, accounts.json                        │
                    └───────────────────────┬─────────────────────────────────┘
                                            │
         ┌──────────────────────────────────┼──────────────────────────────────┐
         │ WebSocket /control                │ WebSocket /lock                   │
         │ CONFIG·ASSIGN → 클라이언트         │ REENTRY_REQUEST/RELEASE           │
         │ STATE·ACCOUNT_CHANGE ← 클라이언트  │                                    │
         ▼                                   ▼                                    │
┌────────────────────────────────────────────────────────────────────────────────┐
│                         EvoParsingD (클라이언트 × N대)                           │
│  WinForms + WebView2 · relay.txt → 서버 주소                                     │
│  ControlClient → /control 연결, ASSIGN 수신 시 로그인·방 진입·작업               │
│  ReentryLockClient → 선제 재진입 시 /lock으로 한 명만 재진입                     │
│  RelaySender → video/baccarat/chat 데이터를 릴레이 서버(ingest)로 전송           │
└────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            │ ws://host:8766/ingest (데이터만)
                                            ▼
                    ┌─────────────────────────────────────────────────────────┐
                    │              EvoRelayNode (데이터 릴레이, 별도 프로젝트)     │
                    │  /ingest 수신 → 구독자(/sub, /sub/video 등)에게 전달        │
                    └─────────────────────────────────────────────────────────┘
```

- **제어(누가 어떤 방에서 작업할지, 휴식/작업 전환, 계정 배정)**: EvoRelay (C#, 포트 8765)
- **실제 게임 접속·파싱·배팅**: EvoParsingD (C#, WebView2)
- **video/baccarat/chat 소켓 데이터 중계**: EvoRelayNode (Node.js, 기본 ingest 8766) — 인스턴스 제어와 분리

---

## EvoRelay (제어 서버)

### 역할

- **EvoParsingD 인스턴스 제어 전용** 웹소켓 서버입니다.
- **video/baccarat/chat 데이터 릴레이는 하지 않습니다.** (EvoRelayNode에서 처리)

### 제공 엔드포인트

| 경로 | 용도 |
|------|------|
| **/control** | EvoParsingD와 1:1 연결. CONFIG/ASSIGN 전송, STATE/ACCOUNT_CHANGE 수신. 휴식 중인 클라이언트에 방·계정을 배정(ASSIGN). |
| **/lock** | 선제 재진입 시 한 번에 한 클라이언트만 재진입하도록 잠금. `REENTRY_REQUEST:roomKey` → `REENTRY_GRANTED` / `REENTRY_DENIED`, `REENTRY_RELEASE:roomKey` → 해제. |

### 동작 요약

1. **시작 시**  
   - Kestrel이 `server.config`의 host/port(기본 8765)로 리스닝.  
   - `lobby.configs.json`, `accounts.json` 로드.

2. **EvoParsingD가 /control 접속**  
   - 서버가 **CONFIG**(WorkingTime, RestTime, InstanceCount, SiteUrl, dataRelayUrl 등) 전송.  
   - 클라이언트가 **STATE:REST** 보내면, 해당 클라이언트를 “휴식 중”으로 등록.

3. **ASSIGN 배정**  
   - 방별로 `InstanceCount`만큼 “WORKING” 인스턴스를 유지하려고, **휴식 중이면서 가장 오래 휴식한** 클라이언트에게 **ASSIGN** 전송.  
   - ASSIGN 페이로드: 계정(id, password), 방(tableId, vtId, title), SiteUrl, dataRelayUrl, WorkingTime, RestTime, RoomTime, ReentryLockReleaseDelay, BetIntervalSeconds 등.

4. **상태 수신**  
   - **STATE:WORKING** — 배정받은 클라이언트가 작업 시작했을 때.  
   - **STATE:READY_TO_REST** — WorkingTime 경과 후 “휴식 대기”. 대체 인스턴스가 방에 들어올 때까지 대기.  
   - **STATE:IN_ROOM** — 방에 완전히 진입(video·baccarat 수신 등). 이때 **READY_TO_REST**였던 다른 클라이언트에게 **REST_GRANTED** 전송.  
   - **ACCOUNT_CHANGE:현재계정ID** — 서버가 사용 가능한 다른 계정을 골라 **NEW_ACCOUNT** 로 내려줌.

5. **계정 배정**  
   - `accounts.json`의 계정 중 **사용 중이 아닌 것**, **이용 횟수가 적은 것** 순으로 배정.

6. **UI (RelayMainForm)**  
   - 방 목록(사용 여부, 테이블명, table_id:vt_id, WORKING 수) 표시.  
   - 설정(WorkingTime, RestTime, InstanceCount, SiteUrl, dataRelayUrl 등) 편집 후 저장 → `lobby.configs.json` 갱신.  
   - 선택 방에 CMD(exit/restart), 전체 종료 등 버튼으로 제어.

### 설정 파일 (EvoRelay 실행 폴더)

- **server.config** — host, port, useHttps, certPath 등 (기본 127.0.0.1:8765)
- **lobby.configs.json** — WorkingTime, RestTime, RoomTime, InstanceCount, SiteUrl, dataRelayUrl, args.configs(방 목록) 등
- **accounts.json** — accounts 배열 { id, password }

---

## EvoParsingD (클라이언트)

### 역할

- 에볼루션 사이트에 **WebView2**로 접속해 로그인·로비·바카라 방 진입.
- **EvoRelay**에 연결해 “휴식/작업” 상태를 보고, **ASSIGN**을 받으면 지정된 계정·방으로 작업 시작.
- 방에서 수신한 **video / baccarat / chat** WebSocket 데이터를 **RelaySender**로 릴레이 서버(예: EvoRelayNode의 `/ingest`)에 전송.
- **선제 재진입**: 방 체류 시간(RoomTime) 경과 후, **ReentryLockClient**로 `/lock` 잠금을 받은 뒤 한 명만 나갔다가 다시 들어옴.

### 시작 시 동작

1. **relay.txt**에서 서버 주소 로드.  
   - 한 줄: `ws://호스트:8766/ingest` 또는 `ws://호스트:8765` 등.  
   - 8766/ingest이면 데이터 릴레이는 8766, **제어는 8765**로 유도.  
   - 없으면 기본 제어 주소(예: DefaultControlBaseUrl) 사용.

2. **ControlBaseUrl**이 있으면  
   - **ControlClient** → `ws://.../control` 연결  
   - **ReentryLockClient** → `ws://.../lock` (재진입 시만 사용)  
   - **RelayUrl**(또는 ASSIGN의 dataRelayUrl)이 있으면 **RelaySender** → ingest 주소로 연결.

3. **Control 모드**인 경우  
   - 처음에는 **휴식** 상태. rest.html 표시.  
   - `/control` 연결 후 **STATE:REST** 전송.  
   - EvoRelay가 **ASSIGN**을 보내면 그때 **작업 시작**(로그인 → 로비 → 배정된 방 진입).

### ASSIGN 수신 시

- **ControlClient_OnAssign**에서:  
  - SiteUrl, 계정(id, password), 방(tableId, vtId, title), WorkingTime, RestTime, RoomTime, dataRelayUrl, ReentryLockReleaseDelay, BetIntervalSeconds 등 적용.  
  - **STATE:WORKING** 전송.  
  - **RelaySender**가 없거나 dataRelayUrl이 바뀌면 새로 생성해 ingest에 연결.  
  - HttpClient 로그인 후 로비로 이동하고, 배정된 방으로 진입.

### 작업 중 상태 전환

- **WorkingTime** 경과 → **STATE:READY_TO_REST** 전송. (즉시 휴식이 아니라 “대체 인스턴스가 방 들어올 때까지” 대기.)
- 다른 인스턴스가 같은 방에 들어와 **STATE:IN_ROOM** 보내면, 서버가 대기 중인 클라이언트에게 **REST_GRANTED** 전송 → 해당 클라이언트는 휴식으로 전환(**STATE:REST**).
- **선제 재진입**: RoomTime(초) 경과 후 `/lock`에 **REENTRY_REQUEST** → **REENTRY_GRANTED** 받으면 방 나갔다가 재진입, video·baccarat 수신 후 N초 지나면 **REENTRY_RELEASE**.

### 데이터 릴레이 (RelaySender)

- WebView2에서 가로챈 소켓 메시지를 `channel`, `table_id-vt_id`, `dir`, `payload` 형태로 큐에 넣고,  
  `channel\x01table_id-vt_id\x01dir\x01payload` 형식으로 **RelayUrl**(ingest)에 전송.  
- EvoRelay는 이 데이터를 다루지 않고, **EvoRelayNode**가 ingest로 받아 구독자에게 전달.

### 설정

- **relay.txt** (실행 폴더): 한 줄에 `ws://호스트:8766/ingest` 또는 `ws://호스트:8765` 등.  
- 계정·방·사이트는 **EvoRelay ASSIGN**으로 받음. (relay.txt에는 서버 주소만.)

---

## EvoRelay와 EvoParsingD 동작 흐름

1. **EvoRelay** 실행  
   - server.config 기준으로 8765 리스닝.  
   - lobby.configs.json, accounts.json 로드.  
   - UI에서 방·설정 확인/수정 가능.

2. **EvoParsingD** 여러 대 실행  
   - 각 클라이언트가 relay.txt의 주소로 **/control** 연결.  
   - 연결 후 **STATE:REST** 전송 → 서버는 “휴식 중” 인스턴스 목록에 추가.

3. **배정(ASSIGN)**  
   - 서버는 방별로 **InstanceCount**만큼 WORKING을 유지하려고, 휴식 중인 클라이언트 중에서(가장 오래 휴식한 순) 계정을 골라 **ASSIGN** 전송.  
   - EvoParsingD는 ASSIGN 수신 시 지정된 계정·방으로 로그인 후 진입, **STATE:WORKING** 전송.

4. **작업 → 휴식**  
   - **WorkingTime**이 지나면 EvoParsingD는 **STATE:READY_TO_REST** 전송.  
   - 서버는 같은 방에 “대체”로 넣을 인스턴스를 배정(ASSIGN).  
   - 대체 인스턴스가 방에 들어와 **STATE:IN_ROOM**을 보내면, 서버가 **READY_TO_REST**였던 클라이언트에게 **REST_GRANTED** 전송.  
   - 해당 클라이언트는 **STATE:REST**로 바꾸고 휴식.  
   - 이후 다시 3번처럼 휴식 중인 클라이언트에 ASSIGN이 나갈 수 있음.

5. **선제 재진입**  
   - 방에 들어온 지 **RoomTime**(초) 정도 지나면, EvoParsingD가 **/lock**에 **REENTRY_REQUEST:roomKey** 전송.  
   - **REENTRY_GRANTED**를 받은 인스턴스만 방을 나갔다가 다시 들어옴.  
   - 재진입 후 video·baccarat 수신하고 **ReentryLockReleaseDelay** 초 지나면 **REENTRY_RELEASE**로 잠금 해제.

6. **계정 변경**  
   - EvoParsingD가 **ACCOUNT_CHANGE:현재ID** 전송 → 서버가 **NEW_ACCOUNT** (id, password) 전송 → 클라이언트는 로그아웃 후 해당 계정으로 재로그인.

7. **원격 제어 (CMD)**  
   - EvoRelay UI에서 “선택 방 인스턴스 종료/재실행” 또는 “모두 종료” 시, 해당 /control 클라이언트들에게 **CMD** (action: exit/restart, delaySeconds) 전송.  
   - EvoParsingD는 수신 시 지연 후 종료 또는 재실행.

---

## 데이터 릴레이 (EvoRelayNode)

- **video / baccarat / chat** 소켓 데이터만 릴레이.  
- **인스턴스 제어(/control, /lock)는 EvoRelay(C#)가 담당.**

역할 요약:

- **/ingest**: EvoParsingD(RelaySender)가 보내는 `channel\x01table_id-vt_id\x01dir\x01payload` 수신.
- **구독자**: `/sub`, `/sub/video`, `/sub/baccarat`, `/sub/chat` 등으로 채널별 구독.

자세한 사용법·포트(기본 ingest 8766, 구독 80/443)는 **EvoRelayNode/README.md**를 참고하세요.

---

## 빌드 및 실행

### 요구 사항

- .NET 8 (Windows)
- WebView2 런타임 (EvoParsingD용)
- EvoRelay: ASP.NET Core(Kestrel) 사용

### 빌드

```bash
# 솔루션 빌드
dotnet build EvoParsingD.sln

# EvoRelay만
dotnet build EvoRelay/EvoRelay.csproj

# EvoParsingD만
dotnet build EvoParsingD/EvoParsingD.csproj
```

### 실행 순서 (제어 구조 사용 시)

1. **EvoRelay** 실행  
   - `EvoRelay/bin/Debug/net8.0-windows/EvoRelay.exe`  
   - server.config, lobby.configs.json, accounts.json 확인.

2. **EvoParsingD** 여러 대 실행  
   - 실행 폴더에 **relay.txt** 한 줄: `ws://EvoRelay서버IP:8765` 또는 데이터 릴레이까지 쓰면 `ws://서버IP:8766/ingest` (이 경우 제어는 8765로 자동 유도).  
   - 각 클라이언트가 /control 연결 후 STATE:REST 전송.  
   - EvoRelay가 ASSIGN을 보내면 자동으로 작업 시작.

3. (선택) **EvoRelayNode** 실행  
   - 데이터 수집·구독이 필요할 때. ingest 8766, EvoRelay 8765와 포트 분리.

---

## 설정 파일

| 파일 | 위치 | 설명 |
|------|------|------|
| **relay.txt** | EvoParsingD 실행 폴더 | 서버 주소 한 줄. 예: `ws://호스트:8766/ingest` 또는 `ws://호스트:8765` |
| **server.config** | EvoRelay 실행 폴더 | host, port(기본 8765), useHttps, certPath 등 |
| **lobby.configs.json** | EvoRelay 실행 폴더 | WorkingTime, RestTime, RoomTime, InstanceCount, SiteUrl, dataRelayUrl, args.configs(방 목록) |
| **accounts.json** | EvoRelay 실행 폴더 | accounts: [ { id, password }, ... ] |

---

## 라이선스 / 기여

이 프로젝트는 에볼루션 게임 사이트 연동·파싱용 클라이언트 및 제어 서버입니다.  
사용 시 해당 사이트의 이용 약관 및 정책을 확인하세요.
