## 백그라운드 작업

사용자가 화면을 보고 있지 않을 때 실행되는 **백그라운드 작업**은 배터리와 메모리를 소모하기 때문에, 안드로이드는 버전이 올라갈수록 제약을 강화해 왔다. 오늘날 "백그라운드에서 뭔가를 하려면" 작업의 성격에 따라 **WorkManager**, **포그라운드 서비스(Foreground Service)**, **AlarmManager** 중 하나를 골라야 하며, 그 선택 기준과 배터리 최적화(Doze·앱 대기 버킷)가 어떤 제약을 거는지 이해하는 것이 이 유닛의 목표다.

<br>

### 1. 백그라운드 제약의 역사

| **버전**              | **주요 변경**                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------------------------------------- |
| **6.0 (API 23)**      | **Doze 모드**·**앱 대기(App Standby)** 도입 — 기기 방치 시 네트워크·작업 지연                                  |
| **8.0 (API 26)**      | **백그라운드 실행 제한** — 백그라운드 앱의 `startService()` 금지, 암시적 브로드캐스트 대부분 수신 불가       |
| **9.0 (API 28)**      | **앱 대기 버킷(App Standby Buckets)** — 사용 빈도에 따라 작업·알람 허용량 차등                                 |
| **12 (API 31)**       | 백그라운드에서 **포그라운드 서비스 시작 금지**(예외 있음), 정확한 알람 권한 `SCHEDULE_EXACT_ALARM` 필요       |
| **13 (API 33)**       | 알림 런타임 권한 `POST_NOTIFICATIONS` — 포그라운드 서비스 알림도 사용자가 숨길 수 있음                          |
| **14 (API 34)**       | 포그라운드 서비스 **타입 선언 필수**(`foregroundServiceType`), 타입별 권한 요구                                |
| **15 (API 35)**       | `dataSync`·`mediaProcessing` 타입 포그라운드 서비스 **24시간 내 총 6시간 제한**, 초과 시 `onTimeout()` 호출     |

흐름은 일관된다. "**사용자가 인지하지 못하는 작업은 시스템이 일괄 관리하고, 인지하는 작업만 앱이 직접 실행하라**"는 방향이다.

> 💡 면접에서 "백그라운드에서 서비스를 돌리면 되지 않나요?"라는 답은 API 26 이전 지식으로 취급된다. 현재 기준 답변은 "**미룰 수 있으면 WorkManager, 사용자가 보고 있어야 하면 포그라운드 서비스, 정확한 시각이면 AlarmManager**"다.

<br>

### 2. 선택 기준 — 어떤 도구를 쓸 것인가

```
            작업이 지금 당장, 사용자가 인지하는 상태로 계속 실행돼야 하는가?
                    │
        ┌───────────┴───────────┐
       예                       아니오
        │                        │
 포그라운드 서비스          특정 시각에 정확히 실행돼야 하는가?
 (음악·내비·통화·             │
  진행 중 업로드 표시)  ┌──────┴──────┐
                       예             아니오
                        │              │
                  AlarmManager     WorkManager
                  (알람·리마인더)   (동기화·로그 업로드·
                                    백업·주기 작업)
```

| **항목**            | **WorkManager**                             | **포그라운드 서비스**                    | **AlarmManager**                    |
| ------------------- | ------------------------------------------- | ---------------------------------------- | ----------------------------------- |
| **실행 보장**       | **프로세스 종료·재부팅 후에도 보장**        | 실행 중에만 (프로세스 종료 시 소멸)      | 예약 시각에 앱 깨움                 |
| **실행 시점**       | 시스템이 조건·배터리 고려해 **지연 가능**   | 즉시                                     | 정확(exact) 또는 유연(inexact)      |
| **사용자 노출**     | 없음 (Expedited 시 임시 알림 가능)          | **알림 필수**                            | 없음                                |
| **제약 조건**       | 네트워크·충전·저장 공간 등 선언 가능        | 없음                                     | 없음                                |
| **적합한 작업**     | 지연 가능·반드시 완료돼야 하는 작업          | 사용자가 인지하는 장기 작업              | 시각 기반 알림·리마인더             |
| **부적합한 작업**   | 즉시 응답이 필요한 UI 작업, 정확한 시각      | 미룰 수 있는 동기화                      | 주기적 데이터 동기화                |

<br>

### 3. WorkManager

**WorkManager**는 Jetpack 라이브러리로, 작업을 **내부 DB에 영속화**해 두고 시스템 조건(JobScheduler 등)에 맞춰 실행한다. 앱이 죽거나 기기가 재부팅돼도 작업은 남아 있다가 실행된다.

### 3-1. 기본 사용

```kotlin
class UploadLogsWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val path = inputData.getString("path") ?: return Result.failure()
        return try {
            LogUploader.upload(File(path))
            Result.success()
        } catch (e: IOException) {
            // 일시적 네트워크 오류 → 백오프 후 재시도 (runAttemptCount로 상한 제어 가능)
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

val request = OneTimeWorkRequestBuilder<UploadLogsWorker>()
    .setInputData(workDataOf("path" to logFile.path))
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)   // Wi-Fi에서만
            .setRequiresBatteryNotLow(true)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context)
    .enqueueUniqueWork("upload-logs", ExistingWorkPolicy.KEEP, request)   // 중복 예약 방지
```

- **`Result.retry()`**: 백오프 정책에 따라 재시도. **`Result.failure()`**: 종료. **`Result.success()`**: 완료
- **`enqueueUniqueWork`**: 같은 이름의 작업이 있으면 `KEEP`(무시)·`REPLACE`(교체)·`APPEND`(연결)로 중복을 제어한다. 앱 시작마다 `enqueue`를 호출하는 코드에서 필수다
- **`PeriodicWorkRequest`**: 최소 간격 **15분**. 정확한 주기가 아니라 "그 창(window) 안 어딘가"에서 실행된다
- **Expedited Work**(`setExpedited`): 사용자 상호작용에 가까운 중요 작업을 우선 실행. 시스템 할당량이 소진되면 일반 작업으로 강등되거나 짧은 포그라운드 서비스로 실행됨 (WorkManager 2.7 이상)

> ⚠️ WorkManager는 "**언젠가 반드시**"를 보장하지, "**지금 당장**"을 보장하지 않는다. 앱이 Doze 상태이거나 대기 버킷이 낮으면 몇 시간 뒤 실행될 수 있다. 사용자가 버튼을 눌러 결과를 기다리는 작업에는 코루틴(`viewModelScope`)을 쓰고, WorkManager는 화면을 떠나도 완료돼야 하는 작업에 쓴다.

<br>

### 4. 포그라운드 서비스

**포그라운드 서비스**는 사용자에게 **알림으로 실행 사실을 노출**하는 대신, 시스템이 포그라운드 프로세스 수준의 우선순위를 부여해 종료를 거의 하지 않는 서비스다 (unit04 참고).

```kotlin
class MusicService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("재생 중")
            .setSmallIcon(R.drawable.ic_play)
            .setOngoing(true)
            .build()

        // startForegroundService() 호출 후 짧은 제한 시간 안에 반드시 호출해야 함 (미호출 시 ANR)
        ServiceCompat.startForeground(
            this, NOTIFICATION_ID, notification,
            ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK   // API 34+: 타입 필수
        )
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

```xml
<!-- 매니페스트: 공통 권한 + 타입별 권한 + 서비스 타입 선언 (API 34+) -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<service
    android:name=".MusicService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```

**주요 제약**

- **백그라운드에서 시작 금지**(API 31+): 앱이 화면에 없을 때 `startForegroundService()`를 호출하면 `ForegroundServiceStartNotAllowedException`. 예외는 알림 탭, 정확한 알람, 고우선순위 FCM 메시지 등 사용자 행동에 준하는 트리거뿐
- **타입 선언 필수**(API 34+): `mediaPlayback`·`location`·`dataSync`·`camera`·`microphone` 등. 타입에 맞지 않는 작업을 하면 Play 정책 위반이 될 수 있고, 일부 타입은 해당 런타임 권한이 있어야 시작된다
- **시간 제한**(API 35+): `dataSync`·`mediaProcessing`은 24시간 중 총 6시간까지만 실행되고 `onTimeout()`이 호출된다. 데이터 동기화 용도는 WorkManager로 옮기라는 신호다
- 서비스도 **메인 스레드에서 콜백**을 받으므로 실제 작업은 코루틴·스레드로 분리해야 한다 (unit01 참고)

<br>

### 5. 배터리 최적화가 거는 제약

### 5-1. Doze와 앱 대기 버킷

- **Doze**: 화면이 꺼지고 기기가 정지 상태로 방치되면 진입. **유지 관리 창(Maintenance Window)**에만 네트워크·작업·알람이 일괄 실행되고, 시간이 갈수록 창의 간격이 길어진다
- **앱 대기 버킷**: 사용 빈도에 따라 `Active → Working set → Frequent → Rare → Restricted`로 분류. 버킷이 낮을수록 WorkManager 실행 횟수와 알람 허용량이 줄어든다
- **제한(Restricted) 모드**: 사용자가 설정에서 앱의 백그라운드 사용을 제한하면 포그라운드 서비스도 사실상 시작할 수 없다

```
사용 빈도 ↑                                                        사용 빈도 ↓
Active ──▶ Working set ──▶ Frequent ──▶ Rare ──▶ Restricted
 제한 없음     작업 소폭 지연    작업 하루 몇 회    하루 1회 수준    거의 실행 불가
```

<br>

### 5-2. 정확한 알람과 예외 요청

- **`setExactAndAllowWhileIdle()`**: Doze 중에도 알람을 깨우지만, 시스템이 **9분에 한 번** 수준으로 빈도를 제한한다 (버전에 따라 다를 수 있음)
- **`SCHEDULE_EXACT_ALARM`**(API 31+): 정확한 알람에 필요한 특수 권한. API 34부터는 알람·시계 앱이 아니면 **기본 거부**되며 사용자가 설정에서 켜야 한다. 캘린더·알람 성격이 아니면 `setWindow()` 같은 유연한 알람을 쓴다
- **배터리 최적화 제외 요청**(`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`): Play 스토어 정책상 핵심 기능이 백그라운드 실행에 의존하는 경우(메신저·건강 추적 등)에만 허용되며, 남용하면 심사에서 거부된다

> 💡 푸시 알림 기반 앱이라면 FCM의 **고우선순위 메시지**가 Doze를 일시적으로 깨울 수 있어 폴링이 필요 없다. "주기적으로 서버를 확인하는 서비스"를 설계하기 전에 서버 푸시로 뒤집을 수 없는지 먼저 검토한다.

<br>

### 6. 정리 — 면접·실무 체크포인트

| **질문**                                                 | **핵심 답변**                                                                                       |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **WorkManager와 포그라운드 서비스 선택 기준은?**         | **지연 가능·완료 보장**이면 WorkManager, **사용자가 인지하는 즉시·장기 작업**이면 포그라운드 서비스 |
| **WorkManager가 보장하는 것과 못 하는 것은?**            | 프로세스 종료·재부팅 후 실행은 보장, **실행 시각은 보장 안 함**                                     |
| **API 26 이후 백그라운드 서비스가 안 되는 이유는?**      | 백그라운드 실행 제한 — `startService()` 금지, 포그라운드 서비스나 WorkManager로 대체               |
| **포그라운드 서비스의 필수 요소는?**                     | 알림, `FOREGROUND_SERVICE` 권한, **API 34+ 타입 선언과 타입별 권한**                                |
| **Doze 모드가 앱에 미치는 영향은?**                      | 네트워크·작업·알람이 **유지 관리 창**으로 묶여 지연됨                                               |
| **PeriodicWorkRequest의 최소 주기는?**                   | 15분, 정확한 주기가 아닌 실행 창 안에서 실행                                                        |

- 백그라운드 작업은 "사용자 인지 여부"와 "지연 가능 여부"로 도구를 고르고, 시스템의 배터리 정책에 **협조**하는 방향으로 설계한다
- 프로세스 우선순위와 ANR은 **unit04**, 백그라운드 작업 결과를 저장하는 방법은 **unit07**을 참고할 것
