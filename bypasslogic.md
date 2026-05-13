
---

## 🔓 `isFakeGps` 방어가 뚫린 원리

### 1. 앱의 방어 메커니즘이 뭔지 먼저 이해

보고서 L34에 나온 `isFakeGps`는 다음 방식으로 탐지합니다:

```java
// 튼튼머니 앱 내부 추정 로직
Location loc = locationManager.getLastKnownLocation(provider);
loc.isFromMockProvider()  // ← 핵심 탐지 포인트
```

Android의 `Location` 객체에는 **`isFromMockProvider()`** 라는 플래그가 있습니다.  
일반적인 Fake GPS 앱(ex: 앱스토어에서 받는 "Fake GPS Go" 류)이 위치를 주입하면 이 플래그가 **`true`** 로 세팅되어서 탐지됩니다.

---

### 2. 우리 방식이 다른 이유 — `addTestProvider` 계층

우리 `MockGpsService.java`는 전혀 다른 계층에서 동작합니다:

```
[일반 Fake GPS 앱]
  → LocationManager에 위치 요청
  → 자기 앱 레이어에서 가짜 좌표 반환
  → Location.isFromMockProvider() = true  ← 탐지됨 ❌

[우리 companion_app의 방식]
  → lm.addTestProvider(GPS_PROVIDER, ...)   ← OS 레벨 프로바이더 교체
  → lm.setTestProviderEnabled(...)
  → lm.setTestProviderLocation(...)         ← OS가 직접 위치를 공급
  → 튼튼머니 앱이 GPS_PROVIDER에 위치 요청
  → OS가 우리가 심은 좌표를 "진짜 GPS 위치"로 반환
  → Location.isFromMockProvider() = ???
```

---

### 3. 핵심: 개발자 옵션 "가상 위치 앱"으로 등록된 앱의 특권

여기가 핵심입니다.

Android는 **개발자 옵션 → 가상 위치 앱**으로 지정된 앱에 한해 `addTestProvider()`로 OS의 위치 공급자 자체를 덮어씌울 수 있는 권한을 줍니다.

```
[개발자 옵션에 등록 안 된 상태]
  addTestProvider() 호출 → SecurityException 발생 → 실패

[개발자 옵션에 등록된 상태 (우리 앱)]
  addTestProvider(GPS_PROVIDER) 호출 → 성공
  → Android LocationManager가 공급하는 GPS_PROVIDER 자체를 교체
  → 이후 어떤 앱이 GPS 좌표를 물어봐도 우리 좌표가 나옴
```

그리고 이렇게 OS 레벨에서 provider를 교체했을 때, **`isFromMockProvider()`의 반환값은 Android 버전에 따라 달라집니다:**

| Android 버전 | `isFromMockProvider()` 결과 |
|---|---|
| Android 11 이하 | `false` 반환 → 탐지 불가 ✅ |
| Android 12+ | OS 정책에 따라 다름 |

Android 12 이전까지는 `addTestProvider`로 주입한 위치는 `isFromMockProvider = false`로 표시되어 **탐지 자체가 불가능**했습니다.

---

### 4. 전체 우회 흐름 정리

```
① companion_app을 개발자 옵션 "가상 위치 앱"으로 등록
        ↓
② companion_app이 GPS_PROVIDER, NETWORK_PROVIDER, FUSED 
   세 채널 모두 addTestProvider()로 OS 레벨에서 교체
        ↓
③ 800ms마다 setTestProviderLocation()으로 시설 좌표 주입
        ↓
④ 튼튼머니 앱이 gpsLocation() 호출
        ↓
⑤ Android OS가 우리가 심은 좌표를 "정식 GPS 위치"로 반환
        ↓
⑥ isFakeGps() → isFromMockProvider() = false → 정상 GPS로 판정
        ↓
⑦ distanceInMeters() → 시설과 현재위치 거리 = 0~수m → 인증 통과
```

---

### 5. 추가 방어였을 `TouchEn mVaccine`은 왜 막지 못했나?

보고서 L37에 나온 백신은 **루팅, 악성앱, 피싱앱** 을 검사하지, GPS 조작 자체를 별도로 검사하는 로직이 아닙니다. companion_app은 루팅 없이 **공식 Android API만 사용**하기 때문에 백신 탐지 대상이 아닙니다.
