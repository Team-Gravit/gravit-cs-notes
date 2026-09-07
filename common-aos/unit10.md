## 권한과 보안

안드로이드 앱은 **샌드박스** 안에서 실행되며, 카메라·위치·연락처처럼 사용자 프라이버시와 직결된 자원은 **런타임 권한(Runtime Permission)**으로 사용자 동의를 받아야 쓸 수 있다. 한편 앱이 저장하는 토큰·개인정보와 APK 안의 코드는 루팅된 기기나 리버스 엔지니어링에 노출될 수 있으므로, **민감 데이터 저장**과 **난독화(Obfuscation)**는 출시 전 반드시 점검해야 한다. 이 유닛은 권한 요청 흐름과 거부 처리, 안전한 저장 방법, 그리고 R8 난독화의 동작과 한계를 정리한다.

<br>

### 1. 안드로이드 보안 모델의 기본

- **앱 샌드박스**: 앱마다 고유한 리눅스 UID가 부여되어 다른 앱의 프로세스·파일에 접근할 수 없다. 내부 저장소(`filesDir`)는 이 덕분에 별도 암호화 없이도 다른 앱이 읽지 못한다
- **권한(Permission)**: 샌드박스 밖의 자원(하드웨어, 사용자 데이터, 다른 앱의 컴포넌트)에 접근하려면 매니페스트에 선언하고, 보호 수준에 따라 사용자 동의를 받는다
- **컴포넌트 노출 제어**: `android:exported`와 `permission` 속성으로 다른 앱이 내 컴포넌트를 호출할 수 있는지 결정한다 (unit01 참고)

| **보호 수준**           | **동의 방식**                       | **예시**                                              |
| ----------------------- | ----------------------------------- | ----------------------------------------------------- |
| **일반(Normal)**        | 매니페스트 선언만으로 자동 허용     | `INTERNET`, `ACCESS_NETWORK_STATE`, `VIBRATE`         |
| **위험(Dangerous)**     | **런타임에 사용자가 명시적으로 허용** | `CAMERA`, `ACCESS_FINE_LOCATION`, `READ_CONTACTS`, `POST_NOTIFICATIONS` |
| **특수(Special)**       | 설정 화면에서 사용자가 직접 켬      | `SYSTEM_ALERT_WINDOW`, `SCHEDULE_EXACT_ALARM`, `MANAGE_EXTERNAL_STORAGE` |
| **서명(Signature)**     | 같은 키로 서명된 앱만               | 같은 개발사 앱 간 내부 API                           |

<br>

### 2. 런타임 권한 흐름

Android 6.0(API 23)부터 위험 권한은 **설치 시가 아니라 사용 시점**에 요청한다. 흐름은 다음과 같다.

```
기능 진입 (예: 사진 촬영 버튼)
   │
   ▼
checkSelfPermission == GRANTED ?  ──예──▶ 기능 실행
   │ 아니오
   ▼
shouldShowRequestPermissionRationale ?
   │                        │
  예 (이전에 거부함)       아니오 (첫 요청 또는 "다시 묻지 않음" 상태)
   │                        │
   ▼                        ▼
왜 필요한지 설명 UI ──▶ requestPermission (시스템 대화상자)
                                │
                ┌───────────────┼────────────────┐
             허용          한 번만 허용        거부
               │               │                 │
            기능 실행     이번 세션만 실행    대체 경로 제공 /
                                            설정 화면 안내 (영구 거부 시)
```

```kotlin
class CameraFragment : Fragment() {
    private val requestCamera = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) openCamera()
        else if (!shouldShowRequestPermissionRationale(Manifest.permission.CAMERA)) {
            // 영구 거부(두 번 거부 또는 '다시 묻지 않음') → 설정으로 안내
            showGoToSettingsDialog()
        } else {
            showRationale()                     // 다음 시도 전에 필요성 설명
        }
    }

    fun onTakePhotoClicked() {
        when {
            ContextCompat.checkSelfPermission(requireContext(), Manifest.permission.CAMERA)
                == PackageManager.PERMISSION_GRANTED -> openCamera()
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> showRationale()
            else -> requestCamera.launch(Manifest.permission.CAMERA)
        }
    }
}
```

**버전별 변화 (요청 로직에 영향을 주는 것)**

- **Android 10(API 29)**: 백그라운드 위치는 별도 권한 `ACCESS_BACKGROUND_LOCATION`으로 분리, 포그라운드 위치를 먼저 받은 뒤 **단계적 요청**
- **Android 11(API 30)**: 위치·카메라·마이크에 "**이번만 허용**" 옵션 추가, 같은 권한을 **두 번 거부하면 시스템이 자동으로 다시 묻지 않음**, 장기간 미사용 앱의 권한 자동 초기화
- **Android 13(API 33)**: 알림에 런타임 권한 `POST_NOTIFICATIONS` 도입, 저장소 권한이 `READ_MEDIA_IMAGES`·`VIDEO`·`AUDIO`로 세분화
- **Android 14(API 34)**: 사진·동영상에 "**선택한 항목만 허용**"(부분 접근) 추가

> ⚠️ 권한이 없는데 API를 호출하면 `SecurityException`으로 즉시 크래시한다. 권한은 사용자가 설정에서 **언제든 회수**할 수 있으므로, "앱 시작 때 받았으니 됐다"가 아니라 **매 사용 시점마다 `checkSelfPermission`**을 확인해야 한다. 회수 시 시스템은 프로세스를 종료하고 재시작시킨다.

<br>

### 2-1. 권한 요청 UX 원칙

- **필요할 때, 맥락 안에서** 요청한다. 앱 첫 실행에 권한 5개를 한꺼번에 묻는 것은 거부율을 높인다
- 요청 전에 **왜 필요한지**를 앱 자체 UI로 먼저 설명하면 시스템 대화상자의 허용률이 올라간다
- 거부되어도 앱이 **동작 가능한 대체 경로**를 제공한다 (사진 대신 갤러리 선택, 위치 대신 수동 입력)
- 권한이 필요 없는 대안 API를 우선 검토한다: **사진 선택기(Photo Picker)**는 미디어 권한 없이 사진을 고를 수 있고, `ACTION_IMAGE_CAPTURE` 인텐트는 카메라 권한 없이 촬영을 위임한다

<br>

### 3. 민감 데이터 저장

앱 샌드박스는 **루팅되지 않은 기기의 다른 앱**으로부터 데이터를 지킬 뿐, 루팅 기기·백업 추출·기기 분실 상황에서는 평문 파일이 그대로 읽힌다. 인증 토큰·개인 식별 정보·결제 관련 데이터는 추가 보호가 필요하다.

| **데이터**                      | **저장 방식**                                              | **비고**                                                  |
| ------------------------------- | ---------------------------------------------------------- | --------------------------------------------------------- |
| **비밀번호**                    | **저장하지 않음**                                          | 서버가 발급한 토큰으로 대체                               |
| **액세스·리프레시 토큰**        | Keystore 키로 암호화한 뒤 내부 저장소(DataStore·파일)      | 만료 짧게, 리프레시는 서버에서 회전                       |
| **API 키(클라이언트 측)**       | 가능하면 서버 프록시로 이동                                | APK에 넣으면 난독화해도 **추출 가능**                     |
| **개인정보 캐시**               | 최소 수집, 내부 저장소, 필요 시 암호화                     | 외부 저장소·로그에 남기지 않음                            |
| **생체 인증 보호 데이터**       | `BiometricPrompt` + Keystore 키(사용자 인증 필수 옵션)     | 키 사용 자체를 생체 인증에 묶음                           |

<br>

### 3-1. Android Keystore

**Android Keystore**는 암호 키를 **앱 프로세스 밖(하드웨어 보안 모듈 또는 TEE)**에 보관하고, 앱은 키 자체를 꺼내지 못한 채 **암호화·복호화 연산만 요청**할 수 있게 한다. 키가 메모리에 노출되지 않으므로 힙 덤프나 루팅으로도 키를 빼내기 어렵다.

```kotlin
object TokenCipher {
    private const val ALIAS = "auth_token_key"

    private fun key(): SecretKey {
        val ks = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        (ks.getEntry(ALIAS, null) as? KeyStore.SecretKeyEntry)?.let { return it.secretKey }
        val gen = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        gen.init(
            KeyGenParameterSpec.Builder(ALIAS, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .build()
        )
        return gen.generateKey()
    }

    fun encrypt(plain: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply { init(Cipher.ENCRYPT_MODE, key()) }
        return cipher.iv + cipher.doFinal(plain)          // IV(12바이트)를 앞에 붙여 저장
    }

    fun decrypt(stored: ByteArray): ByteArray {
        val iv = stored.copyOfRange(0, 12)
        val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply {
            init(Cipher.DECRYPT_MODE, key(), GCMParameterSpec(128, iv))
        }
        return cipher.doFinal(stored.copyOfRange(12, stored.size))
    }
}
```

- 암호화된 바이트는 DataStore(Proto 또는 Base64 문자열)나 내부 파일에 저장한다 (unit07 참고)
- 과거 권장이던 Jetpack Security의 **`EncryptedSharedPreferences`는 `security-crypto` 1.1.0-alpha07에서 폐기(deprecated)**되었다. 특정 제조사 기기에서의 키셋 손상과 메인 스레드 I/O 문제가 이유이며, 현재 권장은 위처럼 **Keystore를 직접 사용**하거나 Tink 같은 검증된 라이브러리와 DataStore를 조합하는 것이다

> 💡 "토큰을 암호화해 저장했으니 안전하다"는 절반만 맞다. 앱이 실행 중이라면 복호화된 토큰이 메모리에 있고, 루팅 기기에서는 훅킹으로 가로챌 수 있다. 저장소 암호화는 **분실·백업·다른 앱 접근**에 대한 방어이고, 근본적으로는 **토큰 수명을 짧게** 하고 서버에서 이상 징후 시 폐기하는 설계가 함께 필요하다.

<br>

### 3-2. 네트워크와 로그에서의 노출 방지

- **평문 HTTP 차단**: Android 9(API 28)부터 기본으로 `cleartext` 트래픽이 금지된다. 개발 서버용 예외는 `network_security_config.xml`에 도메인 단위로만 열고, 릴리스에는 포함하지 않는다
- **인증서 고정(Certificate Pinning)**: `network_security_config`의 `<pin-set>` 또는 OkHttp `CertificatePinner`로 중간자 공격을 막을 수 있으나, 인증서 교체 시 **앱 업데이트 없이는 통신 불가**가 되므로 백업 핀과 만료 관리가 필요하다 (network unit19·web-security unit10 참고)
- **로그**: 릴리스 빌드에서 토큰·개인정보가 `Log`나 로깅 인터셉터로 출력되지 않도록 `BuildConfig.DEBUG`로 분기한다 (unit08 참고). 로그는 `adb logcat`으로 다른 개발자 도구에서 읽을 수 있다
- **백업 제외**: `android:allowBackup`·`dataExtractionRules`로 민감 파일을 클라우드 백업·기기 이전 대상에서 제외한다

<br>

### 4. 난독화와 코드 보호 — R8

APK·AAB 안의 DEX 바이트코드는 `jadx` 같은 도구로 **거의 원본에 가깝게 디컴파일**된다. **R8**은 안드로이드 빌드 도구의 기본 축소기(shrinker)로, 세 가지 일을 한다.

| **기능**                    | **동작**                                                | **효과**                                   |
| --------------------------- | ------------------------------------------------------- | ------------------------------------------ |
| **코드 축소(Shrinking)**    | 도달 불가능한 클래스·메서드·필드 제거                   | APK 크기 감소, 공격 표면 축소              |
| **난독화(Obfuscation)**     | 클래스·멤버 이름을 `a`, `b`, `c`처럼 의미 없는 이름으로 변경 | 디컴파일 결과의 **가독성 저하**            |
| **최적화(Optimization)**    | 인라인, 죽은 코드 제거, 클래스 병합                     | 실행 성능 향상, 크기 감소                  |

```kotlin
// build.gradle.kts (app 모듈)
android {
    buildTypes {
        release {
            isMinifyEnabled = true             // R8 축소·난독화·최적화
            isShrinkResources = true           // 미사용 리소스 제거
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

```
proguard-rules.pro 에 흔히 필요한 규칙
-keep class com.example.app.data.remote.dto.** { *; }   # 리플렉션 기반 직렬화(Gson 등)가 필드명을 쓰는 경우
-keepattributes *Annotation*, Signature                  # 어노테이션·제네릭 시그니처 유지
```

**흔한 함정**

- **리플렉션·직렬화**: Gson처럼 필드명을 런타임에 읽는 라이브러리는 난독화되면 필드가 `a`로 바뀌어 파싱이 깨진다. `@Keep`·`-keep` 규칙 또는 코드 생성 기반 라이브러리(Kotlinx Serialization, Moshi codegen)를 쓴다
- **크래시 스택 해독**: 난독화된 스택 트레이스는 읽을 수 없으므로 빌드마다 생성되는 **`mapping.txt`**를 보관하고 Crashlytics·Play Console에 업로드해야 한다
- **AGP 8.0부터 R8 전체 모드(full mode)가 기본**이 되어 이전에 통과하던 코드가 더 공격적으로 제거·병합될 수 있다 (버전에 따라 다를 수 있음). 릴리스 빌드에서 반드시 실제 기기 테스트를 거친다

> ⚠️ 난독화는 **지연 수단이지 방어가 아니다**. 이름을 바꿔도 문자열 상수(API 키, 서버 URL)와 로직 흐름은 그대로 남는다. 클라이언트에 비밀을 두지 않는 설계가 우선이고, 결제·치트 방지처럼 위조 여부가 중요한 경우에는 **Play Integrity API** 같은 서버 검증을 추가한다.

<br>

### 5. 정리 — 면접·실무 체크포인트

| **질문**                                                  | **핵심 답변**                                                                                       |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **런타임 권한 요청 흐름은?**                              | `checkSelfPermission` → 필요 시 근거 설명 → `requestPermission` → 결과에 따라 실행·대체 경로·설정 안내 |
| **"다시 묻지 않음" 상태는 어떻게 판별하는가?**            | 거부 결과 후 `shouldShowRequestPermissionRationale`이 **false**면 영구 거부 → 설정 화면 안내         |
| **권한이 있어도 매번 확인해야 하는 이유는?**              | 사용자가 설정에서 **언제든 회수**할 수 있고, 미사용 앱은 자동 초기화되기 때문                        |
| **토큰은 어떻게 저장하는가?**                             | **Keystore** 키로 암호화해 내부 저장소에. `EncryptedSharedPreferences`는 폐기됨                       |
| **Keystore가 안전한 이유는?**                             | 키가 앱 프로세스 밖(TEE·HSM)에 있어 **키 자체를 꺼낼 수 없고** 연산만 요청 가능                     |
| **R8 난독화의 한계는?**                                   | 이름만 바꿀 뿐 문자열·로직은 남음. **클라이언트에 비밀을 두지 않는 설계**가 우선                     |
| **난독화 후 크래시 로그가 안 읽힐 때는?**                 | 빌드별 **`mapping.txt`**로 디옵스큐어(deobfuscate)                                                   |

- 권한은 **맥락 안에서 최소한으로** 요청하고, 거부 경로와 대체 경로를 설계에 포함한다
- 민감 데이터는 Keystore로 암호화하되 **토큰 수명·서버 검증**과 함께 설계한다
- 인텐트·컴포넌트 노출은 **unit01**, 저장소는 **unit07**, 네트워크 로깅은 **unit08**을 참고할 것
