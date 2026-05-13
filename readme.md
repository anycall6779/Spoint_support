# 머니 도우미 ( Companion App)

> KSPO 국민진공단 시설 방문 체크인 자동화를 위한 Android 보조 앱

---

## 📋 개요

**머니 도우미**는 KSPO(국민진공단) 머니 앱의 체육시설 방문 인증을 보조하는 Android WebView 앱입니다.  
WebView 기반 UI와 Java Native Bridge를 조합하여 다음 기능을 제공합니다:

- **GPS Mock (가상 위치 주입)** — Foreground Service로 백그라운드에서도 위치 지속 유지
- **KSPO 시설 검색 및 자동 방문 체크인** — KSPO API 연동 및 QR 인증 자동화
- **KSPO 로그인 세션 관리** — 오버레이 WebView로 KSPO 로그인 후 세션 쿠키 공유
- **QR 코드 생성 및 저장** — 인증 QR 코드 생성 및 갤러리 저장

---

## 🏗 프로젝트 구조

```
companion_app/
├── app/
│   ├── build.gradle                    # 앱 빌드 설정 (minSdk 26, targetSdk 34)
│   └── src/main/
│       ├── AndroidManifest.xml         # 권한 및 컴포넌트 선언
│       ├── assets/
│       │   ├── index.html              # 메인 WebView UI (전체 앱 화면)
│       │   ├── jsqr.js                 # QR 코드 스캔 라이브러리
│       │   ├── qrcode.min.js           # QR 코드 생성 라이브러리
│       │   └── qrcode_gen.js           # QR 코드 생성 유틸
│       └── java/com/tuntun/companion/
│           ├── MainActivity.java       # 진입점, WebView 초기화, 오버레이 관리
│           ├── JsBridge.java           # JavaScript ↔ Java 네이티브 브릿지
│           └── MockGpsService.java     # GPS Mock Foreground Service
├── gradle-8.4/                        # 로컬 Gradle 8.4 바이너리
├── gradle/wrapper/
│   └── gradle-wrapper.properties
├── local.properties                   # Android SDK 경로 설정
├── build.gradle                       # 루트 빌드 설정
├── settings.gradle
└── tuntun_release.keystore            # 릴리즈 서명 키스토어
```

---

## ⚙️ 핵심 컴포넌트

### `MainActivity.java`
- WebView 초기화 및 설정 (`JavaScript`, `DOM Storage`, `Mixed Content` 허용)
- `WebView.setWebContentsDebuggingEnabled(true)` — Chrome DevTools 원격 디버깅 지원
- **KSPO 로그인 오버레이** (`showKspoPage`) — 별도 WebView 오버레이로 KSPO 로그인 페이지 표시
- **QR 방문 인증 오버레이** (`showQrAuthFlow`) — 자동 로그인 후 `facilityQr.kspo` API 호출
- 이미지 갤러리 선택 및 Base64 인코딩 처리

### `JsBridge.java`
JavaScript에서 `Android.*` 형태로 호출할 수 있는 네이티브 API 모음:

| 메서드 | 설명 |
|--------|------|
| `startMockGps(lat, lng, name)` | GPS Mock 시작 (Foreground Service 구동) |
| `pauseMockGps()` | Mock 일시정지 (실제 GPS 복귀) |
| `resumeMockGps()` | Mock 재개 (저장된 좌표로 재시작) |
| `stopMockGps()` | Mock 완전 중지 |
| `isMockLocationRegistered()` | 현재 앱이 Mock Location 앱으로 등록됐는지 확인 |
| `httpPostAsync(url, body, callbackId)` | CORS 우회 HTTP POST (KSPO 세션 쿠키 자동 동기화) |
| `httpGetAsync(url, callbackId)` | HTTP GET |
| `httpGetWithHeaderAsync(url, header, callbackId)` | 커스텀 헤더 GET (Kakao Maps API 등) |
| `resolveRedirect(url, callbackId)` | 단축 URL 최종 목적지 추적 (최대 8회 리다이렉트) |
| `showKspoLogin()` | KSPO 로그인 오버레이 표시 |
| `startQrAuth(spoFaciSn, rebTb, standGrpCd)` | QR 방문 인증 자동화 |
| `getCookiesForUrl(url)` | 특정 URL의 WebView 쿠키 반환 |
| `saveCredentials(id, pw)` | KSPO 로그인 정보 저장 (SharedPreferences) |
| `loadId()` / `loadPw()` | 저장된 KSPO 계정 정보 조회 |
| `save(key, value)` / `load(key)` | 일반 설정 저장/불러오기 |
| `pickImage()` | 갤러리 이미지 선택 |
| `saveQrImage(base64Png, fileName)` | QR 이미지 갤러리 저장 |
| `isTuntunInstalled()` | 머니 앱 설치 여부 확인 |
| `openTuntunApp()` | 머니 앱 실행 |
| `openDeveloperSettings()` | Android 개발자 옵션 화면 열기 |
| `toast(msg)` | 토스트 메시지 표시 |

### `MockGpsService.java`
- **Foreground Service** — 화면 꺼짐/백그라운드 상태에서도 GPS Mock 0.8초마다 지속 갱신
- 지원 Provider: `GPS_PROVIDER`, `NETWORK_PROVIDER`, `FUSED` (Android 12+)
- 알림 액션: **일시정지** (실제 GPS 복귀) / **중지** (서비스 완전 종료)
- `START_STICKY` — 비정상 종료 시 마지막 좌표로 자동 재시작

---

## 🔐 권한

| 권한 | 용도 |
|------|------|
| `ACCESS_FINE_LOCATION` | GPS 위치 접근 |
| `ACCESS_COARSE_LOCATION` | 네트워크 위치 접근 |
| `ACCESS_MOCK_LOCATION` | **개발자 옵션 가상 위치 앱 등록** (필수) |
| `FOREGROUND_SERVICE` | Foreground Service 실행 |
| `FOREGROUND_SERVICE_LOCATION` | 위치 타입 Foreground Service |
| `POST_NOTIFICATIONS` | Android 13+ 알림 권한 |
| `INTERNET` | 네트워크 요청 |
| `CAMERA` | QR 스캔 카메라 |
| `READ_MEDIA_IMAGES` | 갤러리 이미지 선택 |
| `WRITE_EXTERNAL_STORAGE` | 갤러리 저장 (Android 9 이하) |

---

## 🛠 빌드 환경

| 항목 | 버전 |
|------|------|
| Android SDK | API 34 (Android 14) |
| Min SDK | API 26 (Android 8.0) |
| Gradle | 8.4 |
| Java | 17+ (JDK 21 권장) |
| Android Build Tools | 34.x |

### SDK 경로 설정

`local.properties` 파일에 SDK 경로를 지정합니다:

```properties
sdk.dir=C\:\\Users\\USER\\Desktop\\zeropay_\\android_sdk
```

---

## 🔨 빌드 방법

### Gradle 직접 실행 (gradlew 없을 경우)

```powershell
# 릴리즈 빌드
.\gradle-8.4\bin\gradle.bat assembleRelease --project-dir . --no-daemon

# 디버그 빌드
.\gradle-8.4\bin\gradle.bat assembleDebug --project-dir . --no-daemon
```

### 출력 APK 위치

```
app/build/outputs/apk/release/app-release.apk
app/build/outputs/apk/debug/app-debug.apk
```

### 릴리즈 서명

`tuntun_release.keystore`가 프로젝트 루트에 포함되어 있으며 `app/build.gradle`에서 자동 서명됩니다:

```gradle
signingConfigs {
    release {
        storeFile file('../tuntun_release.keystore')
        storePassword 'tuntun2024'
        keyAlias 'tuntun'
        keyPassword 'tuntun2024'
    }
}
```

---

## 📱 설치 및 초기 설정

### 1. APK 설치

```bash
adb install app-release.apk
```

또는 기기에 파일 전송 후 직접 설치.

### 2. 가상 위치 앱 등록 (필수)

GPS Mock 기능 사용을 위해 반드시 수행:

1. **설정 → 개발자 옵션** 진입  
   *(개발자 옵션이 없으면: 설정 → 휴대폰 정보 → 빌드 번호를 7번 탭)*
2. **가상 위치 앱 선택** → **튼튼머니 도우미** 선택
3. 앱 내 **GPS Mock 시작** 버튼으로 위치 주입 시작

> **참고**: 앱 내 "개발자 설정 열기" 버튼을 누르면 개발자 옵션으로 바로 이동합니다.

### 3. KSPO 계정 등록

1. 앱 하단 **설정 탭** 이동
2. KSPO 아이디 / 비밀번호 입력 후 저장
3. **KSPO 로그인** 버튼으로 세션 초기화

---

## 🌐 연동 서비스

| 서비스 | 용도 |
|--------|------|
| `.kspo.or.kr` | KSPO 스포츠포인트 시설 검색 API |
| `.kspo.or.kr` | KSPO 앱 방문 인증 API (QR 체크인) |
| Kakao Maps API | 시설 좌표 변환 (커스텀 헤더 GET 사용) |
| V-World API | 좌표 지오코딩 |

---

## 🐛 디버깅

앱은 Chrome DevTools 원격 디버깅을 지원합니다:

1. USB 케이블로 기기 연결 (USB 디버깅 활성화)
2. Chrome에서 `chrome://inspect/#devices` 접속
3. **튼튼머니 도우미** → **inspect** 클릭
4. DevTools에서 JavaScript 콘솔, 네트워크, 소스 디버깅 가능

---

## HOW TO?
 [logic](./bypasslogic.md)

---

## ⚠️ 주의사항

- `ACCESS_MOCK_LOCATION` 권한은 **개발자 도구용 권한**으로, Google Play 배포 시 거부될 수 있습니다.
- GPS Mock은 **개발자 옵션에서 해당 앱을 가상 위치 앱으로 지정**해야만 동작합니다.
- KSPO 서버 정책 변경에 따라 API 엔드포인트나 인증 방식이 달라질 수 있습니다.
- 본 앱은 개인 자동화 보조 도구이며, 공식 튼튼머니 앱과 무관합니다.

---

## 📄 라이선스

Private / Personal Use Only
