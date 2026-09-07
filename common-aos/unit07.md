## 데이터 저장소 선택

앱 데이터는 **크기·구조·수명·민감도**에 따라 저장 방식이 달라진다. 설정값 몇 개를 저장하는 데 관계형 DB를 쓰거나, 수천 건의 목록을 키-값 저장소에 JSON 문자열로 밀어 넣는 것은 모두 잘못된 선택이다. 이 유닛은 **SharedPreferences → DataStore**로 이어지는 키-값 저장소의 변화와, **Room**을 도입해야 하는 기준, 그리고 파일·외부 저장소의 위치를 정리한다.

<br>

### 1. 저장소 선택의 기준

| **질문**                                | **예**                                 | **아니오**                    |
| --------------------------------------- | -------------------------------------- | ----------------------------- |
| **데이터가 구조화된 다건 레코드인가?**  | **Room**(SQLite)                       | 키-값 저장소 또는 파일        |
| **조회·정렬·조인·부분 갱신이 필요한가?** | **Room**                              | 키-값 저장소 또는 파일        |
| **소수의 설정·플래그·토큰인가?**        | **DataStore**(신규) / SharedPreferences(레거시) | 파일 또는 DB          |
| **바이너리(이미지·오디오)인가?**        | **파일**(내부 저장소 또는 캐시)        | DB에 넣지 않음                |
| **다른 앱과 공유해야 하는가?**          | MediaStore·SAF·ContentProvider         | 앱 전용 내부 저장소           |
| **앱을 삭제해도 남아야 하는가?**        | MediaStore(공유 미디어)                | 내부 저장소(삭제 시 함께 제거) |

```
                       데이터 성격
                           │
        ┌──────────────────┼──────────────────┐
     소량 키-값          구조화 다건            바이너리·대용량
        │                   │                     │
   DataStore             Room                파일 시스템
 (Preferences/Proto)   (SQLite + DAO)      (filesDir / cacheDir)
        │                   │                     │
   설정, 토큰,         오프라인 캐시,          이미지, 다운로드,
   온보딩 완료 여부     로컬 우선 데이터        로그 파일
```

> 💡 저장소를 고를 때는 "지금 몇 개인가"가 아니라 "**어떻게 읽을 것인가**"를 먼저 묻는다. 키로 통째로 꺼내면 키-값, 조건으로 걸러 일부만 꺼내면 DB다. 나중에 "즐겨찾기 목록에서 최근 것 10개만"이 필요해지는 순간 키-값 저장소는 한계에 부딪힌다.

<br>

### 2. SharedPreferences — 레거시 키-값 저장소

`SharedPreferences`는 XML 파일 하나에 키-값을 저장하는 가장 오래된 API다. 단순하지만 설계상 한계가 명확하다.

```kotlin
// 흔한 사용 — 문제가 되는 지점이 세 군데 있다
val prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE)

prefs.edit().putBoolean("dark_mode", true).commit()   // ① commit(): 호출 스레드에서 동기 디스크 쓰기
val dark = prefs.getBoolean("dark_mode", false)        // ② 첫 접근 시 파일 전체를 메인 스레드에서 로드할 수 있음
prefs.edit().putString("token", jwt).apply()           // ③ apply()는 비동기지만 오류를 알 수 없음
```

**한계**

- **동기 I/O**: `commit()`은 호출 스레드를 막고, `apply()`도 프로세스 종료 직전 `onStop()` 등에서 **디스크 쓰기가 끝날 때까지 대기**해 ANR의 원인이 되어 왔다 (unit04 참고)
- **오류 신호 없음**: `apply()`는 실패해도 예외를 던지지 않는다
- **타입 안전성 없음**: 키는 문자열, 값은 런타임 캐스팅. 오타·타입 불일치를 컴파일 시점에 잡지 못한다
- **트랜잭션·일관성 보장 부족**: 다중 프로세스 접근이 안전하지 않다 (`MODE_MULTI_PROCESS`는 폐기됨)

<br>

### 3. DataStore — 코루틴 기반 키-값 저장소

**Jetpack DataStore**는 SharedPreferences의 대체제로, **코루틴과 Flow 위에서 비동기·트랜잭션 방식**으로 동작한다. 두 가지 구현이 있다.

| **항목**             | **Preferences DataStore**              | **Proto DataStore**                             |
| -------------------- | -------------------------------------- | ----------------------------------------------- |
| **스키마**           | 없음 (키-값)                           | **Protocol Buffers로 정의** (타입 안전)         |
| **마이그레이션 난이도** | SharedPreferences에서 쉬움           | .proto 정의와 빌드 설정 필요                    |
| **적합한 경우**      | 단순 설정, 빠른 전환                   | 구조화된 설정 객체, 타입 안전이 중요한 경우      |

```kotlin
// Preferences DataStore: 파일 하나당 인스턴스 하나 (최상위 확장 프로퍼티로 싱글턴 보장)
private val Context.settingsStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

class SettingsRepository(private val context: Context) {
    private val darkModeKey = booleanPreferencesKey("dark_mode")

    // 읽기: Flow → 값이 바뀔 때마다 자동 방출, IO 디스패처에서 처리됨
    val darkMode: Flow<Boolean> = context.settingsStore.data
        .catch { e -> if (e is IOException) emit(emptyPreferences()) else throw e }
        .map { prefs -> prefs[darkModeKey] ?: false }

    // 쓰기: suspend + 트랜잭션 (읽기-수정-쓰기가 원자적)
    suspend fun setDarkMode(enabled: Boolean) {
        context.settingsStore.edit { prefs -> prefs[darkModeKey] = enabled }
    }
}
```

**SharedPreferences 대비 개선점**

- 모든 I/O가 `Dispatchers.IO`에서 수행되어 **메인 스레드를 막지 않는다**
- `edit {}`는 **원자적 읽기-수정-쓰기**를 보장하며, 실패 시 예외로 알려준다
- 읽기가 `Flow`라서 설정 변경이 UI에 **자동 반영**된다 (unit09의 단방향 데이터 흐름과 잘 맞음)
- `SharedPreferencesMigration`으로 기존 데이터를 첫 접근 시 자동 이전할 수 있다

> ⚠️ 같은 파일에 대해 DataStore 인스턴스를 **두 개 이상 만들면** 예외가 발생한다. 반드시 싱글턴(최상위 프로퍼티 또는 DI 컨테이너)으로 관리한다. 또한 DataStore는 소량 설정용이므로 **수백 KB 이상의 데이터나 목록**을 넣으면 매 변경마다 전체 파일을 다시 쓰게 되어 느려진다.

<br>

### 4. Room — 구조화 데이터를 위한 SQLite 추상화

**Room**은 SQLite 위에 **컴파일 타임 SQL 검증·객체 매핑·Flow 통합**을 얹은 Jetpack 라이브러리다. 직접 `SQLiteOpenHelper`를 쓰는 것에 비해 보일러플레이트와 런타임 오류가 크게 줄어든다.

### 4-1. 도입 기준

다음 중 하나라도 해당하면 Room을 고려한다.

- 레코드가 **수십 건 이상**이고 계속 늘어난다 (채팅 메시지, 게시글 캐시, 검색 기록)
- **조건 조회·정렬·페이징**이 필요하다 (최근순 20개, 특정 사용자의 글만)
- 여러 엔티티 사이에 **관계**가 있다 (사용자-게시글-댓글)
- **오프라인 우선(Offline-first)** 전략으로 서버 데이터의 로컬 사본을 유지한다
- 데이터 변경을 UI가 **관찰**해야 한다 (`Flow<List<T>>` 반환)

반대로 키 몇 개의 설정, 한 번 읽고 마는 임시 데이터, 바이너리 파일은 Room이 과하다.

<br>

### 4-2. 구성 요소와 사용

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: Long,
    val title: String,
    val savedAt: Long,
    @ColumnInfo(name = "is_read") val isRead: Boolean = false
)

@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY savedAt DESC LIMIT :limit")
    fun recent(limit: Int): Flow<List<ArticleEntity>>          // 테이블 변경 시 자동 재방출

    @Upsert
    suspend fun upsertAll(items: List<ArticleEntity>)           // 있으면 갱신, 없으면 삽입

    @Query("UPDATE articles SET is_read = 1 WHERE id = :id")
    suspend fun markRead(id: Long)
}

@Database(entities = [ArticleEntity::class], version = 2)
abstract class AppDatabase : RoomDatabase() {
    abstract fun articleDao(): ArticleDao
}
```

- **Entity**: 테이블. **DAO**: 쿼리 인터페이스 (SQL은 컴파일 시 검증). **Database**: 진입점이자 싱글턴
- `suspend` 함수와 `Flow` 반환을 지원해 메인 스레드 I/O를 원천 차단한다 — 메인 스레드에서 동기 호출하면 기본 설정에서 **예외**를 던진다
- **마이그레이션**: 스키마를 바꾸면 `version`을 올리고 `Migration`을 제공해야 한다. 개발 중 `fallbackToDestructiveMigration()`으로 넘어가면 출시 후 사용자 데이터가 삭제되므로 릴리스에서는 반드시 명시적 마이그레이션을 쓴다
- **TypeConverter**로 `Date`·enum·리스트를 컬럼으로 변환하고, `@Relation`·`@Embedded`로 관계를 표현한다

> 💡 Room은 "DB 계층"이 아니라 **데이터 소스**다. 아키텍처에서 ViewModel이 DAO를 직접 호출하지 않고 **Repository**를 거치게 하면 (unit09 참고) 나중에 네트워크 캐시 전략을 바꾸거나 테스트용 가짜 소스로 교체하기 쉽다.

<br>

### 5. 파일 저장소와 범위 지정 저장소

| **위치**                     | **API**                          | **권한**              | **앱 삭제 시** | **용도**                                |
| ---------------------------- | -------------------------------- | --------------------- | -------------- | --------------------------------------- |
| **내부 저장소**              | `filesDir`                       | 불필요 (앱 전용)      | 삭제           | 앱 전용 파일, 민감 데이터               |
| **내부 캐시**                | `cacheDir`                       | 불필요                | 삭제           | 임시·재생성 가능 데이터, 시스템이 정리 가능 |
| **외부 앱 전용**             | `getExternalFilesDir()`          | 불필요 (API 19+)      | 삭제           | 큰 파일, 사용자가 접근 가능한 앱 파일   |
| **공유 미디어**              | `MediaStore`                     | 읽기 시 미디어 권한   | **유지**       | 사진·동영상·음악                        |
| **문서·임의 파일**           | Storage Access Framework(SAF)    | 사용자 선택으로 대체  | 유지           | 사용자가 고른 문서 가져오기·내보내기    |

- Android 10(API 29)부터 **범위 지정 저장소(Scoped Storage)**가 적용되어 외부 저장소를 자유롭게 읽고 쓸 수 없다. 공유 데이터는 MediaStore·SAF를 통해 접근한다
- `cacheDir`는 저장 공간이 부족하면 시스템이 삭제할 수 있으므로, 여기 둔 데이터는 **없어도 동작해야** 한다

<br>

### 6. 저장소 비교 요약

| **항목**              | **SharedPreferences** | **DataStore**              | **Room**                     | **파일**             |
| --------------------- | --------------------- | -------------------------- | ---------------------------- | -------------------- |
| **데이터 형태**       | 키-값                 | 키-값 / Proto 객체         | **관계형 레코드**            | 바이트 스트림        |
| **비동기**            | 부분적 (`apply`)      | **완전 비동기(코루틴)**    | suspend·Flow                 | 직접 처리            |
| **타입 안전**         | 없음                  | Proto는 있음               | **컴파일 타임 SQL 검증**     | 없음                 |
| **트랜잭션**          | 없음                  | 원자적 edit                | **ACID**                     | 없음                 |
| **관찰 가능성**       | 리스너 (누수 주의)    | **Flow**                   | **Flow**                     | 없음                 |
| **적합 규모**         | 소량                  | 소량                       | **대량·구조화**              | 대용량 바이너리      |
| **현재 권장 여부**    | 신규 코드 비권장      | **권장**                   | 권장                         | 용도에 따라          |

<br>

### 7. 정리 — 면접·실무 체크포인트

| **질문**                                                | **핵심 답변**                                                                             |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **SharedPreferences 대신 DataStore를 쓰는 이유는?**     | 코루틴 기반 **비동기 I/O**, 원자적 트랜잭션, 오류 전달, Flow 관찰 — ANR·타입 문제 해결   |
| **Preferences vs Proto DataStore 차이는?**              | 스키마 유무. Proto는 **타입 안전**하지만 .proto 정의가 필요                               |
| **Room을 도입해야 하는 기준은?**                        | 다건 레코드·조건 조회·관계·오프라인 캐시·변경 관찰이 필요할 때                            |
| **Room 마이그레이션을 빼먹으면?**                       | 버전 불일치 예외 또는 `fallbackToDestructiveMigration`으로 **사용자 데이터 삭제**         |
| **이미지를 DB에 넣으면 안 되는 이유는?**                | DB 파일 비대화·쿼리 성능 저하. 파일로 저장하고 DB에는 **경로**만 보관                     |
| **토큰·비밀번호는 어디에 저장하는가?**                  | 평문 SharedPreferences 금지. Keystore 기반 암호화 (unit10 참고)                            |

- 저장소는 **읽는 방식과 규모**로 고른다: 키로 통째로 → DataStore, 조건 조회 → Room, 바이너리 → 파일
- 저장소 접근은 Repository 뒤에 숨겨 UI가 구현체를 모르게 한다 (**unit09** 참고)
