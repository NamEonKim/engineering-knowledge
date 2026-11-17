# Cleaning with Dee AI 도입 및 CRM 아키텍처

이 가이드는 Cleaning with Dee를 단순한 청소 서비스 웹사이트에서 AI 기반 고객·파트너십 관리 플랫폼(CRM + AI Agent)으로 발전시키는 실용적인 경로를 제안합니다. Flutter 관리자 대시보드를 중심으로 고객 및 이해관계자 커뮤니케이션을 사람이 통제하며, AI 결과를 자율적인 사업 결정으로 취급하지 않습니다.

> **범위와 가정:** 실제 운영 중인 시스템이나 법률 자문을 설명하는 문서가 아니라 제안 아키텍처입니다. 아래 Firestore 컬렉션과 필드는 예시 시작점이며, 실제 업무, 권한, 보존 요건에 맞게 조정해야 합니다. 일반적인 AI 도입, 모델 선택, RAG, 프롬프팅, 평가 개념은 [AI 솔루션 도입 및 LLM 엔지니어링 가이드](../llm/ai-solution-adoption-and-llm-engineering-guide.ko.md)를 참고하세요.

## 목차

- [1. 제품 목표와 제안 스택](#1-제품-목표와-제안-스택)
- [2. 시스템 아키텍처](#2-시스템-아키텍처)
- [3. CRM 에이전트와 경계](#3-crm-에이전트와-경계)
- [4. Firestore 컬렉션 및 필드 예시](#4-firestore-컬렉션-및-필드-예시)
- [5. 사람이 검토하는 이메일 워크플로](#5-사람이-검토하는-이메일-워크플로)
- [6. 영국 개인정보 및 전자 마케팅 보호 조치](#6-영국-개인정보-및-전자-마케팅-보호-조치)
- [7. Flutter 호출 예시](#7-flutter-호출-예시)
- [8. 단계별 MVP 우선순위](#8-단계별-mvp-우선순위)
- [9. 이후 확장](#9-이후-확장)
- [공식 참고 자료](#공식-참고-자료)

## 1. 제품 목표와 제안 스택

제안하는 제품은 고객, 파트너십, 커뮤니티, 리크루터, 경영진 관계 관리 영역을 갖춘 Flutter 관리자 대시보드입니다. 권한이 있는 직원이 레코드를 검토하고 유용한 요약과 초안을 준비하며 Cleaning with Dee의 사업 관계 업무를 조율하도록 돕습니다. 대시보드에서는 검증된 정보, AI가 생성한 콘텐츠, 사람의 검토가 필요한 항목을 명확히 구분해야 합니다.

| 계층 | 제안 시작점 | 책임 |
| --- | --- | --- |
| 관리자 인터페이스 | Flutter | 인증된 직원 업무, 레코드 검토, 초안 편집, 승인, 상태 확인 |
| 신원 확인 | Firebase Authentication | 로그인과 신원 토큰 발급. 애플리케이션 역할은 서버에서 별도로 확인해야 함 |
| CRM 데이터 | Cloud Firestore | 운영 레코드, 레코드 간 참조, 워크플로 상태, 범위가 제한된 감사 메타데이터 |
| 신뢰할 수 있는 애플리케이션 백엔드 | Firebase Cloud Functions 또는 인증된 다른 서버 백엔드 | 신원과 역할 검증, 업무 규칙 적용, 권한이 있는 레코드 읽기·쓰기, 모델 호출, 이메일 발송 통제 |
| 모델 연동 | 현재 지원되는 서버 측 API를 통한 Gemini. 지원되는 앱 매개 사용 사례에서는 Firebase AI Logic 검토 가능 | 선택적 요약, 분류, 초안 생성. 현재 모델, 리전, 계정, 개인정보 및 서비스 제한에 따름 |
| 이메일 제공업체 | 조직이 승인한 SMTP 또는 기타 트랜잭션/마케팅 이메일 서비스 | 승인된 메시지만 발송하고 백엔드에 발송 및 상태 결과 반환 |
| 보조 자료 | 작업 문서를 위한 Google Drive, 운영 절차를 위한 Notion, 제한된 내보내기·분석을 위한 Sheets | Firestore를 CRM 원본으로 유지하고 같은 연락처 데이터를 여러 도구에 복제하지 않음 |

이는 제안 스택이지 필수 요건이 아닙니다. Firebase 클라이언트 SDK/API 제공 여부, 할당량, 모델 접근, 가격, 무료 사용 조건, 제공업체의 데이터 처리 방식은 바뀔 수 있습니다. 구현 전에 현재 제품 문서, 지역별 제공 여부, 서비스 약관, 데이터 처리 약관을 확인하고 어떤 구성 요소나 사용량도 계속 무료라고 가정하지 마세요.

## 2. 시스템 아키텍처

Flutter 클라이언트는 인증된 화면 표시와 사용자 동작에 집중합니다. 백엔드는 정책을 적용하는 신뢰 경계입니다. Firebase 신원을 확인하고 직원 역할과 레코드별 접근 권한을 검사하며 요청을 검증하고, 권한이 높은 인증정보와 이메일 발송 권한을 클라이언트에서 분리합니다.

```text
권한 있는 직원
        |
        v
Flutter 관리자 대시보드
  - Firebase Authentication 로그인
  - CRM 화면 및 편집 가능한 이메일 초안
  - 명시적인 사람의 승인 동작
        |
        | HTTPS/Callable 요청 + Firebase ID 토큰
        v
Firebase Functions (신뢰할 수 있는 백엔드)
  - 신원, 역할, 레코드 접근, 요청 검증
  - 허용된 Firestore 레코드만 읽기
  - 모델 요청 전 맥락 최소화/가림 처리
  - 서버 측 인증정보로 승인된 모델 호출
  - 생성 콘텐츠를 자동 발송하지 않고 초안으로 저장
  - 발송 직전에 권한, 수신 거부, 억제 목록 재확인
        |                         |
        v                         v
Cloud Firestore             승인된 이메일 서비스
  - CRM 레코드                 - 승인 뒤에만 발송
  - 초안 및 상태               - 발송 결과 반환
  - 억제 상태
```

Firebase Cloud Functions(또는 이에 준하는 신뢰할 수 있는 서버)는 AI 결과 검증, 승인된 이메일 발송, 후속 작업 예약, 발송 결과 기록을 수행할 수 있습니다. Firestore Security Rules는 클라이언트의 무단 읽기 및 쓰기를 차단해야 하지만, 백엔드 권한 검사를 대체하지는 않습니다. 권한이 높은 서버 SDK 접근은 Firestore Security Rules를 우회할 수 있으므로 각 Function이 자체적으로 동일한 권한 및 입력 검증 정책을 적용해야 합니다. 모델 제공업체 비밀정보, 서비스 계정 인증정보, 권한이 높은 이메일 인증정보를 Flutter 앱 번들이나 소스 저장소에 넣지 마세요.

모델 호출에는 선택한 배포 방식에서 현재 지원되는 연동을 사용하세요. Gemini와 Firebase AI Logic은 Firebase 생태계의 예시이지 모든 계정, 리전, 업무, 백엔드에서 사용 가능하다는 보장이 아닙니다. Firebase AI Logic의 지원 클라이언트 흐름과 보호 기능은 최신 문서에서 확인하세요. 비공개 CRM 맥락, 권한이 필요한 도구, 대외 발송 결정은 신뢰할 수 있는 백엔드에서 처리해야 합니다. 실사용 모델 ID를 코드에 하드코딩하지 말고 현재 지원 모델을 설정으로 선택한 뒤 제공업체 최신 문서를 확인하세요.

## 3. CRM 에이전트와 경계

다음 항목은 독립적인 직원 계정이 아니라 범위가 제한된 워크플로 보조 기능 또는 작업 유형으로 다룹니다. 오케스트레이터는 허용된 기능에 요청을 전달할 뿐, 사용자가 요청했다는 이유로 에이전트에 접근 권한을 부여하지 않습니다. 각 작업에는 권한이 확인된 목적에 필요한 데이터만 제공합니다.

| 에이전트 | 가능한 지원 | 경계 |
| --- | --- | --- |
| **Customer Agent** | 고객 문의나 견적 요청 분류, 권한이 확인된 고객 이력 요약, 서비스 응답 또는 견적 초안 작성 | 서비스 가능 여부, 가격, 약속, 고객 사실을 지어내지 않으며 직원이 응답을 검증 |
| **Partnership Agent** | 파트너십 레코드 정리, 출처가 확인된 제휴 기회 평가, 후속 연락 초안 작성 | 발견한 업체·연락처를 검증하고 약속을 추론하거나 메시지를 발송하지 않으며 기록된 사실과 제안을 구분 |
| **Community Agent** | 승인된 지역 커뮤니티·단체 관계 정리 및 홍보·협업 초안 준비 | 게시하거나 회원에게 연락하거나 다른 사람의 정보를 노출하지 않음 |
| **Recruiter Agent** | 채용 업무 메모, 후보자 관계 정리 또는 커뮤니케이션 초안 및 후속 업무 지원 | 고용 결정은 사람이 책임지고, 승인되고 필요한 후보자 정보만 사용 |
| **Executive Agent** | 요청자가 볼 권한이 있는 레코드에서 간결한 업무 요약 준비 | 집계 또는 최소화된 정보를 우선하고 역할 기반 접근을 적용하며 불확실성을 표시 |
| **Email & Follow-up Agent** | 이메일 작성·수정·분류 및 후속 연락 알림 초안 또는 대기열 등록 | 초안과 대기열 관리만 수행. 별도 서버 측 발송 작업은 권한이 있는 사람의 승인과 발송 직전 적격성 재확인이 필요 |
| **Orchestrator Agent** | 사용자 요청을 특정 승인 워크플로에 전달하고 결과 수집 | 제한 없는 도구 접근, 역할 권한 상승, 검토 및 발송 통제 우회 금지 |

에이전트마다 별도의 모델을 구매할 필요는 없습니다. 현재 지원되는 모델 하나가 여러 워크플로를 처리하더라도 프롬프트, 데이터 접근, 권한을 분리할 수 있습니다. 에이전트 지침과 출력 스키마는 버전 관리하세요. 각 워크플로를 대표 레코드와 접근 거부 사례로 시험합니다. 결정과 외부 커뮤니케이션의 책임은 사람에게 둡니다.

파트너십 업무 예시는 다음과 같습니다. 직원 또는 승인된 조사 단계가 잠재 파트너를 찾고 출처를 기록합니다. Partnership Agent가 출처가 표시된 기회 요약을 준비하고 Email & Follow-up Agent가 맞춤 초안을 작성합니다. Orchestrator는 검토 대기열에 넣고 권한이 있는 관리자가 검토·승인합니다. 서버가 발송 적격성을 재확인한 뒤 발송하고 결과를 기록합니다. 조사 결과는 검증할 잠재 고객이지 확인된 관계나 연락 허가가 아닙니다.

## 4. Firestore 컬렉션 및 필드 예시

아래 스키마는 제안 예시이며 기존 Cleaning with Dee 데이터베이스를 설명하는 사실이 아닙니다. 승인된 업무에 필요한 최소 필드부터 시작하고, 컬렉션 간 개인 데이터를 복사하는 대신 문서 참조나 ID를 사용하세요.

| 컬렉션 | 필드 예시 | 목적 및 주의 사항 |
| --- | --- | --- |
| `staffProfiles` | `uid`, `role`, `active`, `createdAt`, `updatedAt` | 인증된 직원을 승인된 앱 역할에 연결합니다. 클라이언트가 보낸 역할을 신뢰하지 마세요. |
| `contacts` | `name`, `email`, `organisation`, `contactType`, `source`, `status`, `notes` | 고객, 파트너, 커뮤니티, 리크루터, 경영진 연락처를 통합 관리합니다. 필요한 정보만 저장하고 `source`에 데이터 출처를 기록합니다. |
| `partnerships` | `company`, `opportunity`, `contactId`, `stage`, `nextAction` | 제휴 대상과 협업 진행 현황입니다. 메모는 사실에 기반하고 관련성이 있으며 접근이 제한되어야 합니다. |
| `emailQueue` | `recipient`, `subject`, `body`, `status`, `campaignId` | 초안 및 승인 대기 항목입니다. 클라이언트 쓰기로 발송을 트리거할 수 없어야 하며 대기 초안은 발송 허가가 아닙니다. |
| `emailLogs` | `recipient`, `sentAt`, `status`, `providerMessageId` | 접근이 제한된 발송 기록입니다. 보존 기간을 정하고 불필요하게 메시지 내용을 중복 저장하지 않습니다. |
| `tasks` | `contactId`, `action`, `dueDate`, `result`, `owner` | 후속 연락, 미팅, 전화 업무입니다. |
| `activities` | `contactId`, `activityType`, `description`, `createdAt` | 연락처 활동 이력입니다. 설명은 목적에 필요한 내용으로 제한하고 불필요한 민감 정보를 넣지 않습니다. |
| `consents` | `contactId`, `consentStatus`, `lawfulBasis`, `recordedAt` | 적절한 경우 수신 동의 증거 또는 검토된 처리 근거를 기록합니다. 이 필드 자체가 법적 결론은 아니므로 기록을 뒷받침할 증거·출처와 맥락을 보존합니다. |
| `suppressionList` | `email`, `reason`, `unsubscribeAt` | 모든 발송 전에 확인하는 접근 제한 목록입니다. 이의 제기나 수신 거부를 존중하는 데 필요한 정보만 보존하고 접근을 제한합니다. |

위 이름과 필드는 원본 제안의 예시 스키마를 보존한 것으로 필수 또는 프로덕션 준비 스키마가 아닙니다. 정확한 필드 이름과 권한 또는 마케팅 상태의 법적 의미는 조직이 결정해야 합니다. 적절한 경우 동의 또는 다른 허가 근거를 기록하고 출처와 맥락을 보존하며 발송 시 억제 여부를 최종 확인합니다. 실제 CRM 데이터를 적재하기 전에 문서별 권한, 보존, 정정, 삭제 절차를 정의하세요.

## 5. 사람이 검토하는 이메일 워크플로

1. 인증된 직원이 권한이 있는 고객 또는 이해관계자 레코드를 선택하고 목적을 지정해 초안을 요청합니다.
2. 백엔드가 사용자 역할과 레코드 접근 권한을 검증하고 요청의 허용 여부를 확인한 뒤, 승인된 CRM 필드로 최소화된 프롬프트를 구성합니다. 모델에 불필요한 개인정보를 보내지 마세요.
3. 모델이 초안을 반환합니다. 백엔드는 결과를 검증하고 원본 레코드 및 프롬프트 버전 참조와 함께 `emailQueue`에 검토 대기 상태로 저장합니다.
4. 직원이 대시보드에서 수신자, 제목, 본문, 목적, 근거가 된 사실을 검토하고 수정합니다. 화면에서 생성된 문구와 검증된 CRM 정보를 구분합니다.
5. 명시적 승인 후 백엔드가 승인자의 권한을 다시 확인하고 수신자 적격성, 적용되는 동의/허가 기록, 억제 목록, 기록된 이의 제기 또는 수신 거부 여부를 재검사합니다.
6. 검사를 통과할 때만 백엔드가 승인된 SMTP 또는 이메일 제공업체에 메시지를 제출합니다. Firebase Trigger Email Extension은 지원되는 Firestore 트리거 방식의 선택지일 수 있지만, 발송 가능한 레코드는 신뢰할 수 있는 서버 코드만 만들도록 구성합니다. 이메일 발송을 트리거하는 컬렉션에 클라이언트가 직접 쓰지 못하게 하세요. 불필요한 메시지 데이터를 보존하지 않으면서 승인자와 승인 시각, 발송 시도, 제공업체 결과, 억제 또는 차단 사유를 접근이 제한된 `emailLogs`에 기록합니다.
7. 적격성이 불분명하거나 증거가 누락되거나 레코드가 억제되었거나 발송이 실패하면 발송하지 않습니다. 사람이 수정하거나 해결하도록 항목을 반환합니다.

이러한 검사를 우회하는 클라이언트 측 `sendEmail` 기능을 노출하지 마세요. AI가 만든 초안, 사용자 계정, 과거 적격성 검사는 이후 발송을 승인하기에 충분하지 않습니다.

## 6. 영국 개인정보 및 전자 마케팅 보호 조치

이 가이드는 법률 자문이 아닙니다. 메시지가 직접 마케팅에 해당하는지, 어떤 PECR 규정이 적용되는지, 수신자의 가입자 유형이 무엇인지, 개인정보 처리의 적법한 근거가 있는지, 예외가 적용되는지는 구체적인 사실에 따라 달라집니다. 사람에게 연락하거나 과거 CRM 데이터를 가져오기 전에 조직별 법률 또는 개인정보 자문을 받으세요.

- **전자적 연락 전에 PECR을 확인합니다.** ICO 지침은 법인 가입자와 개인 가입자를 구분합니다. 유한회사 주소는 개인사업자나 일부 파트너십 주소와 다르게 취급될 수 있으며, 후자는 개인 가입자로 간주될 수 있습니다. 개인 가입자 대상 전자 마케팅은 일반적으로 동의가 필요하며 고객에게 적용되는 제한적인 “soft opt-in” 등의 예외가 있을 수 있습니다. 도메인 이름이나 B2B라는 추정에 의존하지 말고 수신자 유형과 현재 조건을 확인하세요. 공개된 이메일 주소나 AI가 작성한 메시지만으로 발송이 허용된다고 가정하지 마세요.
- **UK GDPR을 별도로 고려합니다.** 개인정보를 사용하는 경우 적절한 적법한 근거를 확인·기록하고 투명성, 목적 제한, 최소화, 정확성, 보안, 보존 의무를 충족합니다. PECR상 발송 허용 여부와 UK GDPR상 적법한 처리 근거는 관련이 있지만 서로 다른 점검 사항입니다.
- **이의 제기와 수신 거부를 존중합니다.** 효과적인 억제 절차를 유지하고 발송 직전에 확인합니다. UK GDPR은 직접 마케팅 목적 처리에 대한 개인의 절대적 이의 제기권을 규정합니다. 향후 원치 않는 연락을 방지하는 데 필요한 정보만 접근을 제한해 보관합니다.
- **AI 제공업체에 전달하는 데이터를 최소화합니다.** 업무에 필요한 승인 필드만 사용합니다. 개인정보를 보내기 전에 제공업체의 최신 약관, 보존, 학습/데이터 이용 제어, 국제 이전, 하위 처리업체, 계정 설정을 검토합니다.
- **영향이 큰 결정을 자동화하지 않습니다.** 아웃리치, 채용, 사람에게 영향을 미치는 결정은 사람이 감독합니다. 데이터를 정정하고 개인정보 요청을 처리하는 경로를 마련합니다.
- **문서화하고 재검토합니다.** 의도된 목적, 데이터 원본, 승인, 보존 기간, 접근 역할, 억제 절차, 사고 연락처를 기록합니다. 업무 흐름, 제공업체, 관련 지침이 바뀌면 재평가합니다.

ICO 공식 참고 자료:

- [ICO: PECR 전자우편 마케팅](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guide-to-pecr/electronic-and-telephone-marketing/electronic-mail-marketing/)
- [ICO: 직접 마케팅 지침](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/direct-marketing-guidance/)
- [ICO: 기업 간 마케팅](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/business-to-business-marketing/)
- [ICO: UK GDPR 적법한 근거 안내](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/a-guide-to-lawful-basis/)
- [ICO: 이의 제기권](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-object/)

## 7. Flutter 호출 예시

선택한 경로에 필요한 패키지만 추가하세요. Firebase AI Logic 클라이언트 설정 예시에는 `firebase_ai`, `firebase_app_check`를 사용할 수 있습니다. 아래 CRM 초안 워크플로는 비공개 CRM 맥락과 권한이 높은 모델 접근을 백엔드에 두기 위해 `cloud_functions`를 사용합니다. 사용 전에 최신 Flutter 설정 문서, 지원 모델 설정, App Check 요구사항을 확인하세요.

```powershell
flutter pub add cloud_functions
# 선택 사항: 지원되는 비민감 Firebase AI Logic 클라이언트 흐름
flutter pub add firebase_ai firebase_app_check
```

이 간단한 예시는 Flutter `cloud_functions` 패키지로 인증된 백엔드에 초안 생성을 요청합니다. 이름, 이메일 주소, 자유 형식 개인정보 대신 레코드 참조를 보내도록 의도했습니다. 실제 앱에서는 Firebase 프로젝트, Callable Function, 오류 처리, 권한 검증을 구성해야 하며 이 코드는 설명용일 뿐입니다.

```dart
import 'package:cloud_functions/cloud_functions.dart';

Future<Map<String, dynamic>> createOutreachDraft(String contactId) async {
  final callable = FirebaseFunctions.instance.httpsCallable('createOutreachDraft');
  final result = await callable.call(<String, Object?>{
    'contactId': contactId,
    'purpose': 'customer_follow_up',
  });

  final data = result.data;
  if (data is! Map) {
    throw const FormatException('Unexpected draft response.');
  }
  return Map<String, dynamic>.from(data);
}
```

Function은 인증, 역할, `contactId` 접근 권한을 검증하고 승인된 맥락만 불러오며 요청 및 호출 수를 제한하고 검토용 초안을 저장해야 합니다. 클라이언트가 보낸 목적은 검증 대상 입력이지 권한이 아닙니다. 제공업체 및 이메일 인증정보는 서버 측 비밀정보 저장소에 보관하세요. 프로덕션에서는 타입이 지정된 응답 모델을 사용하고 민감한 백엔드 세부 정보가 노출되지 않도록 Callable 오류를 처리합니다. Firebase AI Logic을 지원되는 적격 흐름에서 Flutter 클라이언트로 직접 사용하는 경우 App Check를 활성화하고 비공개 고객 데이터를 보내거나 클라이언트 응답을 이메일 발송 권한으로 취급하지 마세요.

Firebase AI Logic은 요건을 충족하는 클라이언트 측 AI 흐름에도 사용할 수 있습니다. 아래 설명용 예시는 민감하지 않은 텍스트에만 사용하고, 비공개 CRM 레코드와 아웃리치 초안은 신뢰할 수 있는 백엔드에서 처리하세요. 지원 모델 문서와 현재 설정에서 선택한 모델 ID를 전달하며 코드에 하드코딩하지 않습니다.

```dart
import 'package:firebase_ai/firebase_ai.dart';

Future<String> summarizeNonSensitiveText({
  required String supportedModelId,
  required String publicText,
}) async {
  final model = FirebaseAI.googleAI().generativeModel(
    model: supportedModelId,
  );
  final response = await model.generateContent([
    Content.text('Summarize this non-sensitive text: $publicText'),
  ]);
  final text = response.text;
  if (text == null) {
    throw StateError('The model returned no text.');
  }
  return text;
}
```

App Check를 구성하고 현재 Firebase AI Logic 설정, 계정, 모델, 개인정보, 할당량, 과금 요건을 확인하세요. 클라이언트 SDK를 사용해도 비공개 Firestore 레코드 접근이나 이메일 발송 권한이 생기지는 않습니다.

## 8. 단계별 MVP 우선순위

| 단계 | 결과물 | 다음 단계 전 확인 |
| --- | --- | --- |
| **0. 업무 및 개인정보 준비** | 사용자 역할, 연락처 출처, 기존 이메일 관행, PECR/적법 근거 검토, 데이터 보존, 성공 지표 정리 | 책임 있는 사업 담당자와 개인정보/보안 검토자가 파일럿 범위와 테스트 데이터를 승인 |
| **1. 안전한 CRM 기반** | Flutter 관리자 대시보드 로그인, 역할 기반 화면, 최소 Firestore 레코드, 서버 측 권한 검증, 감사 메타데이터 | 역할 및 레코드 접근 테스트에서 권한 없는 사용자가 레코드를 읽거나 변경할 수 없음 |
| **2. 읽기 전용 AI 지원** | 승인되고 최소화된 데이터를 사용하는 범위가 제한된 요약 또는 분류 워크플로 하나 | 대표 사례로 품질을 검토하고 대외 메시지나 영향이 큰 결정을 자동화하지 않음 |
| **3. 초안 전용 커뮤니케이션** | Customer 및 Partnership 업무를 위한 Gemini 이메일 초안·분류. 다른 팀의 초안은 승인 후 추가 | 모든 초안에 근거와 생성 여부가 표시되고 모델 또는 클라이언트 경로로 발송할 수 없음 |
| **4. 사람이 승인하는 발송 통제** | 관리자 승인, 서버 측 발송, 후속 일정, 억제/적격성 확인, 발송 로그 | 거부, 수신 거부, 억제, 미승인 메시지가 발송되지 않음을 엔드투엔드 테스트로 확인 |
| **5. 파트너십 및 커뮤니티 업무** | Partnership과 Community 지원을 추가하고 작업 및 후속 조치 추적 | 출처와 접근을 검증하고 감독 없는 아웃리치 금지 |
| **6. 채용 및 경영진 업무** | 개인정보 및 접근 검토 후 Recruiter 및 Executive 요약·알림 추가 | 고용 결정은 사람이 담당하고 경영진 데이터는 역할별 제한 |
| **7. 오케스트레이션 및 보고** | 승인된 에이전트 간 범위가 제한된 업무 전달, 성과 보고 및 분석은 이후 추가 | 각 확장 기능에 담당자, 문서화된 접근 경계, 측정 가능한 이점, 대체 경로, 롤백 계획이 있음 |

안전한 신원 확인, 데이터 접근, 통합 CRM 연락처부터 시작합니다. 그다음 AI 분류·초안과 승인 통제 발송 흐름을 추가하고, 이후 후속 일정, Partnership/Community, Recruiter/Executive, 오케스트레이션, 보고 순서로 확장합니다. 기존 절차와 결과를 비교하고 품질, 준수, 운영 비용이 허용되지 않으면 해당 단계를 중단하거나 수정합니다.

## 9. 이후 확장

CRM과 검토 절차가 안정된 뒤 승인된 조사 절차로 부동산 관리업체, 지역 커뮤니티 단체, 청소 서비스 제휴 후보 등의 잠재 연락처를 수집하고 출처를 기록해 사람이 검증한 다음 CRM에 등록할 수 있습니다. 이후 파이프라인은 검증된 잠재 고객 분류, 파트너십 업무 지원, 이메일 초안 작성, 사람의 승인, 서버 발송, 후속 조치와 분석 추적을 지원할 수 있습니다. 조사 결과는 사람이 확인하기 전까지 미검증 정보이며, 정보 수집은 연락 허가가 아닙니다.

CRM 데이터의 원본은 Firestore로 유지합니다. Google Drive에는 계약서와 제안서를, Notion에는 운영 절차를, Google Sheets에는 범위가 제한된 내보내기나 분석 결과를 둘 수 있습니다. 이 도구들에 연락처 데이터베이스를 중복으로 보관하지 마세요.

## 공식 참고 자료

지원 기능, 제공 여부, 가격, 제한, 데이터 처리는 현재 서비스 문서에서 확인하세요. Firebase 클라이언트 API, 무료 사용 조건, 모델 접근, 제공업체 동작은 바뀔 수 있습니다. 아래 링크는 출발점이지 특정 제품의 추천이 아닙니다.

- [Flutter용 Firebase Authentication](https://firebase.google.com/docs/auth/flutter/start)
- [Cloud Firestore 보안 규칙](https://firebase.google.com/docs/firestore/security/get-started)
- [Firebase Cloud Functions](https://firebase.google.com/docs/functions)
- [Firebase AI Logic](https://firebase.google.com/docs/ai-logic)
- [Firebase AI Logic Flutter 설정](https://firebase.google.com/docs/ai-logic/get-started?api=dev)
- [Gemini API 문서](https://ai.google.dev/gemini-api/docs)
- [Gemini API 가격](https://ai.google.dev/gemini-api/docs/pricing)
- [Flutter `cloud_functions` 패키지](https://pub.dev/packages/cloud_functions)
- [Firebase Trigger Email Extension](https://firebase.google.com/products/extensions/firebase-firestore-send-email)
