# 🔍 Code Analysis: PLAN-A

> 본 문서는 팀 프로젝트 PLAN-A의 핵심 코드를 기술적 관점에서 분석한 개인 리뷰 노트입니다.  
> 공식 레포: https://github.com/samkwon1122/Plan-A

---

## 1. 전체 코드 구조

모든 로직이 `lib/main.dart` 하나에 작성되어 있다.  
Flutter 앱의 전형적인 StatefulWidget 구조를 따른다.

```
main.dart
  ├── main()                      앱 진입점, 타임존 초기화
  ├── MyApp                       MaterialApp 설정
  └── VoiceToTextScreen           핵심 화면 (StatefulWidget)
        ├── initState()           캘린더 초기화
        ├── _retrieveCalendars()  Plan-A / Plan-A(미확정) 캘린더 생성
        ├── _requestPermissions() 파일 접근 권한 요청
        ├── _pickFileAndTranscribe() 오늘자 녹음 파일 감지 및 처리
        ├── _transcribeFile()     Whisper API 호출
        ├── _extractSchedules()   GPT-4o API 호출
        ├── _addScheduleToCalendar() 캘린더 등록
        └── build()               UI 렌더링
```

---

## 2. 핵심 모듈별 분석

### ① 앱 초기화 — 타임존 설정

```dart
void main() {
  tz.initializeTimeZones();
  runApp(MyApp());
}
```

앱 시작 시 타임존 데이터를 초기화한다.  
캘린더에 일정을 등록할 때 `Asia/Seoul` 타임존을 명시적으로 지정하기 위해 필요하다.  
타임존 처리 없이 저장하면 기기 설정에 따라 시간이 달라질 수 있다.

---

### ② 캘린더 초기화 — `_retrieveCalendars()`

```dart
var planACalendar = calendarsResult.data!.firstWhere(
  (calendar) => calendar.name == 'Plan-A',
  orElse: () => Calendar(id: ''),
);
if (planACalendar.name == 'Plan-A') {
  // 이미 존재하면 그대로 사용
} else {
  // 없으면 새로 생성
  await _deviceCalendarPlugin.createCalendar('Plan-A', 
    calendarColor: Color(0xFF105625));
}
```

앱 실행 시 `Plan-A` (초록색)와 `Plan-A (미확정)` (빨간색) 두 캘린더가 있는지 확인하고, 없으면 자동으로 생성한다.  
매번 실행해도 중복 생성되지 않도록 이름으로 먼저 확인하는 구조다.

---

### ③ 오늘자 파일 감지 — `_pickFileAndTranscribe()`

```dart
Directory recordDir = await Directory('/storage/emulated/0/Call');
List<FileSystemEntity> files = recordDir.listSync();

DateTime today = DateTime.now();
DateTime startOfToday = DateTime(today.year, today.month, today.day);
DateTime endOfToday = DateTime(today.year, today.month, today.day + 1)
    .subtract(Duration(seconds: 1));

for (var file in files) {
  if (file is File) {
    DateTime modificationDate = file.lastModifiedSync();
    if (modificationDate.isAfter(startOfToday) && 
        modificationDate.isBefore(endOfToday)) {
      // 오늘자 파일만 처리
    }
  }
}
```

통화 녹음 파일이 저장되는 경로(`/storage/emulated/0/Call`)를 직접 스캔한다.  
파일의 **수정 시간**을 기준으로 오늘 하루 동안 생성된 파일만 필터링해서 하나씩 처리한다.

**한계:** 경로가 하드코딩되어 있어 기기마다 다를 수 있다. 실제 배포 시 다양한 안드로이드 기기에서 동작하지 않을 가능성이 있다.

---

### ④ Whisper API 호출 — `_transcribeFile()`

```dart
var request = http.MultipartRequest(
  'POST', 
  Uri.parse('https://api.openai.com/v1/audio/transcriptions')
);
request.headers['Authorization'] = 'Bearer $apiKey';
request.files.add(await http.MultipartFile.fromPath('file', filePath));
request.fields['model'] = 'whisper-1';

var response = await request.send();
var responseBody = await http.Response.fromStream(response);

if (responseBody.statusCode == 200) {
  var responseData = json.decode(utf8.decode(responseBody.bodyBytes));
  return responseData['text'];
}
```

오디오 파일을 `multipart/form-data` 형식으로 OpenAI 서버에 전송한다.  
`utf8.decode(responseBody.bodyBytes)`로 한국어 텍스트가 깨지지 않도록 처리한 부분이 눈에 띈다.  
응답에서 `text` 필드만 추출해서 반환한다.

---

### ⑤ GPT-4o API 호출 — `_extractSchedules()`

few-shot prompting으로 GPT-4o가 일정 정보를 JSON 형식으로 출력하도록 유도한다.

**프롬프트 구조:**
```
System: 일정 추출 역할 부여
        + 오늘 날짜 주입 (상대적 날짜 표현 처리용)
        + decided 필드 설명 ("0" = 미확정)

User   (예시 1): 금요일 미팅 변경 통화 대본
Assistant (예시 1): [{"date":"2024-06-07","time":"11:00",...,"decided":"1"}]

User   (예시 2): 회식 일정 미확정 통화 대본
Assistant (예시 2): [{"date":"...","decided":"0",...}]

User   (실제 입력): ${text}
```

**주요 설정:**
```dart
"temperature": 0.5,   // 일관성과 유연성의 균형
"max_tokens": 256     // 출력 길이 제한
```

**예시 2번의 역할이 핵심이다.**  
"일정이 미확정인 경우 `decided`를 `0`으로" 라는 규칙을 예시로 보여줌으로써,  
GPT가 실제 입력에서도 확정/미확정을 스스로 판단하도록 유도한다.

---

### ⑥ 캘린더 등록 — `_addScheduleToCalendar()`

```dart
final location = tz.getLocation('Asia/Seoul');
final TZDateTime startDateTime = tz.TZDateTime.from(
  DateTime.parse(schedule['date'] + ' ' + schedule['time']),
  location,
);

if (schedule['decided'] == "1") {
  event = Event(
    _defaultCalendar!.id,          // Plan-A (초록색)
    title: schedule['task'],
    start: startDateTime,
    end: endDateTime,
    reminders: [Reminder(minutes: 30)],
    status: EventStatus.Confirmed,
  );
} else {
  event = Event(
    _undecidedCalendar!.id,        // Plan-A (미확정) (빨간색)
    title: schedule['task'] + " (미확정)",
    status: EventStatus.Tentative,
  );
}
```

`decided` 값에 따라 두 캘린더 중 하나에 등록한다.  
확정 일정은 30분 전 리마인더가 자동으로 추가된다.  
타임존을 `Asia/Seoul`로 명시해서 국가/기기 설정과 무관하게 시간이 정확히 저장된다.

---

## 3. 전체 데이터 흐름 정리

```
앱 실행
  └→ _retrieveCalendars()
       캘린더 존재 확인 및 생성

버튼 클릭
  └→ _requestPermissions()
       파일 접근 권한 확인
  └→ _pickFileAndTranscribe()
       /storage/emulated/0/Call 스캔
       오늘자 파일 필터링
       파일마다 반복:
         └→ _transcribeFile()
              Whisper API → 텍스트 반환
         └→ _extractSchedules()
              GPT-4o API → JSON 반환
              예: [{"task":"미팅","date":"2024-06-07","decided":"1"}]
         └→ _addScheduleToCalendar()
              decided == "1" → Plan-A 캘린더
              decided == "0" → Plan-A(미확정) 캘린더
              setState() → 앱 화면에 요약 표시
```

---

## 4. 코드에서 배운 점

**① multipart 요청**  
단순 JSON POST가 아니라 파일을 전송할 때는 `multipart/form-data` 형식을 써야 한다는 걸 처음 알았다.

**② few-shot prompting 설계 방식**  
예시를 단순히 "정상 케이스"만 넣는 게 아니라, **예외 케이스(미확정)** 를 예시에 포함시켜 GPT가 스스로 판단하도록 유도하는 설계가 인상적이었다. LINGO-Space의 semantic parser도 같은 패턴을 사용한다.

**③ 타임존 처리의 중요성**  
일정 앱처럼 시간이 핵심인 서비스에서는 타임존 처리가 필수라는 걸 코드를 보며 이해했다.

---

## 5. 개선 가능한 부분

| 항목 | 현재 | 개선 방향 |
|------|------|-----------|
| 녹음 파일 경로 | 하드코딩 | 기기별 동적 경로 탐색 |
| 에러 처리 | 실패 시 메시지만 출력 | 실패 원인 구분 (네트워크/API/파일 형식) |
| 입력 소스 | 통화 녹음만 | 카톡, 문자, 이메일으로 확장 |
| 일정 확인 | 자동 등록 | 등록 전 사용자 확인 단계 추가 |
| 모델 | GPT API 의존 | 개인정보 보호를 위한 로컬 모델 |

---

## 참고

- 공식 레포: https://github.com/samkwon1122/Plan-A
- 프로젝트 리뷰: ./project_review.md
