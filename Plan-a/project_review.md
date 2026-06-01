# 📱 Project Review: PLAN-A

**프로젝트명:** PLAN-A: AI-Powered Phone-Based Scheduling Assistant  
**수업:** AI 핵심기술 기반 실무 프로젝트 (GIST, 2024)  
**기간:** Mar. 2024 – Jun. 2024  
**팀원:** 권흥찬, 김두영, 김명진, 김지윤, 김태욱  
**공식 레포:** https://github.com/samkwon1122/Plan-A  
**나의 역할:** PPT 제작 및 발표, 테스트용 데이터셋 대본 작성 및 녹음, 아이디어 탐색, 자료조사  
**리뷰 작성:** Myoungjin Kim | 2026

---

## 1. 프로젝트 개요

통화 녹음 파일을 자동으로 분석해서 일정을 추출하고 기기 캘린더에 등록해주는 Flutter 기반 Android 앱이다.

**핵심 파이프라인:**
```
통화 녹음 파일 (오늘자 자동 감지)
        ↓
Whisper API (음성 → 텍스트)
        ↓
GPT-4o (텍스트 → 구조화된 일정 JSON, few-shot prompting)
        ↓
Device Calendar (확정/미확정 캘린더에 각각 저장)
```

**타겟층:**
1. 일정이 매우 많아 등록에 번거로움이 있는 사람 (교수님, 강사 등)
2. 갑작스러운 일정 대처가 어렵거나 자주 잊어버리는 사람 (운전 중인 택배기사 등)
3. 일정이 빡빡한 대학생이나 일반인

---

## 2. 아이디어 발전 과정

처음부터 이 아이디어가 나온 건 아니었다. 여러 차례 피봇팅을 거쳤다.

```
1단계: 최초 후보 5개
  → 전화 상담 LLM, 이메일 요약, Zero-shot TTS, 학습 도우미 챗봇, 리뷰 감정분석
  → 1, 3, 5번 채택하여 발표

2단계: 교수님 피드백 "세 아이디어를 융합해보라"
  → 음성으로 하루 일과 말하고 AI가 일기로 정리 + 그림으로 묘사

3단계: 피드백 "일기를 안 쓰는 사람은 어차피 안 쓴다"
  → 일기 → 캘린더로 방향 전환
  → 개인 비서처럼 전화를 참고해서 일정을 자동 정리해주는 앱으로 확정
```

이 피봇팅 과정에서 배운 것: **나에게 유용한 것이 남들에게 항상 유용한 건 아니다.**  
그림 일기 아이디어가 팀 내에서도 "굳이 쓸까?"라는 반응이 나왔고, 타겟층의 니즈를 정확히 파악하는 것이 얼마나 중요한지 깨달았다.

---

## 3. 나의 기여

### ① 테스트용 데이터셋 대본 작성 및 녹음

단순히 "전화 통화"를 만드는 게 아니라, 서비스에서 오류가 생길 만한 케이스를 고민하며 대본을 작성했다.

예를 들어:
- 일정이 **변경**되는 경우 ("금요일 → 토요일로 바꿔요")
- 일정이 **미확정**인 경우 ("언제가 좋을지 나중에 다시 연락드릴게요")
- **상대적 날짜 표현**이 있는 경우 ("이번 주 금요일", "다음 주 월요일")

이 과정에서 서비스를 개발자 시각이 아닌 **비판적 시각**으로 바라보는 연습이 됐다.

### ② PPT 제작 및 발표

IR피칭 강사님 강의를 듣고 발표를 준비했다. 배운 것:
- 청중의 관심을 끄는 요소와 프로젝트의 필요성을 **유기적으로 연결**하는 구성
- 발표를 녹화해서 스스로 피드백하는 방법 → 실제로 가만히 있지 못하는 습관 발견 후 개선

### ③ 아이디어 탐색 및 자료조사

확정/미확정 일정 구분 시스템 아이디어에 기여했다.  
"일정이 아직 확정되지 않은 경우를 어떻게 처리할 것인가"라는 문제를 제기했고,  
이것이 `decided` 필드와 별도 캘린더(빨간색 표시)로 구현됐다.

---

## 4. 기술적 분석 (리뷰 당시 새롭게 이해한 것들)  
이후 코드를 다시 분석하며 이해한 기술적 내용들이다.

### ① Whisper API 호출 구조

```dart
var request = http.MultipartRequest(
  'POST',
  Uri.parse('https://api.openai.com/v1/audio/transcriptions')
);
request.files.add(await http.MultipartFile.fromPath('file', filePath));
request.fields['model'] = 'whisper-1';
```

오디오 파일을 `multipart/form-data` 형식으로 전송한다.  
"Whisper API가 음성을 텍스트로 변환한다"는 것만 알았는데, 실제로는 파일을 HTTP multipart 요청으로 전송하고 응답에서 `text` 필드를 추출하는 구조였다.

### ② GPT API few-shot prompting 구조

```
System: 일정 추출 역할 부여 + 오늘 날짜 주입
User (예시 1): 금요일 미팅 통화 대본
Assistant (예시 1): [{"date": "2024-06-07", "time": "11:00", ..., "decided": "1"}]
User (예시 2): 회식 일정 미확정 통화 대본
Assistant (예시 2): [{"date": "...", ..., "decided": "0"}]
User (실제 입력): ${text}
```

대화 형식으로 예시를 2개 넣어 GPT가 출력 형식을 학습하도록 유도했다.  
`temperature=0.5`, `max_tokens=256`으로 설정.

내가 제안한 확정/미확정 구분이 프롬프트의 `decided` 필드와 예시 2번(미확정 케이스)으로 구현됐다는 걸 코드를 보며 확인했다.

### ③ 오늘자 파일 자동 감지

```dart
DateTime today = DateTime.now();
DateTime startOfToday = DateTime(today.year, today.month, today.day);

for (var file in files) {
  DateTime modificationDate = file.lastModifiedSync();
  if (modificationDate.isAfter(startOfToday) && 
      modificationDate.isBefore(endOfToday)) {
    // 오늘자 파일만 처리
  }
}
```

파일의 수정 시간을 기준으로 오늘자 파일만 필터링한다.

### ④ 확정/미확정 분기 처리

```dart
if (schedule['decided'] == "1") {
  event = Event(
    _defaultCalendar!.id,       // Plan-A 캘린더 (초록색)
    status: EventStatus.Confirmed,
  );
} else {
  event = Event(
    _undecidedCalendar!.id,     // Plan-A (미확정) 캘린더 (빨간색)
    title: schedule['task'] + " (미확정)",
    status: EventStatus.Tentative,
  );
}
```

---

## 5. 최종 발표 피드백 및 개선 방향

발표 후 교수님 피드백:
1. 기업 회의에서도 사용 가능하도록 방향 확장
2. 일정 추가 전 사용자 확인 피드백 과정 추가
3. 운전 중 실시간 일정 감지 기능

팀이 정리한 추가 개선 방향:
- **카톡/문자/이메일**로 입력 소스 확장
- 전화 종료 즉시 자동 분석 → 임시 저장 → 사용자 확인 후 등록
- 기업 기밀 보호를 위한 로컬 모델 개발
- 베타 테스트 후 앱스토어 배포

---


---

## 참고

- 공식 레포: https://github.com/samkwon1122/Plan-A
