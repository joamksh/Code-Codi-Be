# Code-Codi 백엔드와 joamksh 작업 이력 분석

분석일: 2026-09-28. 기준: 로컬 develop, HEAD `04df469` (2025-06-19).

현재 소스의 도메인·서비스·엔티티·API 구조와 로컬 Git 이력을 대조했다. 원격 서버를 새로 fetch하지 않았으며, 로컬에 저장된 모든 브랜치의 커밋은 HEAD 이력에 포함되어 있었다. 실제 서버 실행이나 DB 연동 테스트는 수행하지 않았다. 따라서 아래의 구현 설명은 코드에 존재하는 기능을 뜻하며 운영 검증을 뜻하지 않는다.

## 1. 전체 프로젝트

Code-Codi는 **수업에 참여하는 학생 팀과 교수자의 프로젝트 진행·과제 제출을 지원하는 협업 서비스의 백엔드**로 해석된다. 학생/교수자 역할, 수업별 팀, 팀별 회의록·일정·칸반, 교수자 과제 안내와 학생 제출 데이터가 이 해석의 근거다. 저장소에 README는 없어 제품 목적은 코드에서 복원했다.

| 영역 | 코드에서 확인한 기능 |
| --- | --- |
| user | 회원가입, 비밀번호 암호화·확인, 세션 로그인, 내 정보 확인, 로그아웃, 탈퇴, 학생/교수자 역할 |
| course | 수업 엔티티와 수업 목록 조회 |
| team | 수업에 연결된 팀 생성·수정, 이메일로 팀원 연결, 팀 탈퇴, 사용자별 팀 조회 |
| project | 팀별 칸반 항목 CRUD, TODO/INPROGRESS/COMPLETED 상태, 담당자·우선순위·기한 |
| schedule | 팀 일정 CRUD, 연도·월·팀 조건의 일정 조회 |
| meeting | 팀 회의록 CRUD, 안건·세부 내용·결정사항 개별 관리, 회의 참석자 관리 |
| taskGuide | 수업별 과제 안내, 세부 문항, 마감일, 팀별 제출용 과제 생성, 배포 상태 |
| task | 팀별 과제 답안 저장·수정, 제출/미제출 전환, 교수자 확인용 목록 |
| board | 공유/가이드 게시판, 댓글, 좋아요 수, 조회 수, 인기글, 이미지 업로드 |

기술 구성은 Java 17, Spring Boot 3.4.1, Spring Web, Spring Data JPA, Oracle JDBC, Gradle, Lombok, Bean Validation, Spring Security, springdoc OpenAPI다. 별도 서비스로 나뉜 구조가 아니라 하나의 Spring Boot 애플리케이션 안에 기능별 패키지를 둔 형태다.

일반적인 요청 흐름은 `Controller → Service → Repository → DB`이며 DTO와 Converter로 요청·응답 데이터를 변환한다. meeting/task/taskGuide 등은 Command 서비스와 Query 서비스를 분리한다. 공통 ApiResponse와 예외 처리 체계가 있지만 모든 API의 응답 형식이 완전히 동일한 것은 아니다. 로그인은 HttpSession에 사용자 응답 정보를 보관한다.

핵심 데이터 관계:

```text
Course(수업) ─ Team(팀) ─ UserTeam ─ User(회원)
                ├─ Meeting ─ Agenda ─ AgendaDetail
                │     ├─ Decision
                │     └─ MeetingAttendee ─ UserTeam
                ├─ Schedule
                └─ Project(teamId로 연결되는 칸반 항목)

Course ─ TaskGuide(교수자 안내) ─ TaskGuideDetail(문항)
            └─ Task(팀별 제출본) ─ TaskDetail(답안)
                  │                    └─ TaskGuideDetail 참조
                  └─ Team 참조
```

예를 들어 교수가 특정 수업에 문항 3개짜리 과제를 작성하고 배포하면, 배포 시점에 그 수업에 속한 각 팀에 과제와 빈 답안 3개가 생성된다. 학생 팀은 답안을 저장하고 제출 상태를 바꾸며, 교수자는 수업·팀·상태 조건으로 제출 목록을 확인한다. 회의록·칸반·일정은 그 팀의 협업을 지원한다.

## 2. joamksh의 기여 범위

저장소의 HEAD 도달 가능 커밋은 총 150개이며, 전체 이력의 날짜 범위는 2025-04-29~2025-06-19다. `joamksh <joamksh@naver.com>`의 기록은 2025-05-14~2025-06-19에 걸쳐 **77개: 비병합 61개, 병합 16개**다. 비병합에도 이름 변경·되돌리기·초기 설정이 포함되므로 61개의 독립 기능을 만들었다는 의미는 아니다. 커밋 수를 기여율로 환산하지 않았다.

현재 패키지 경로에 대한 비병합 이력에서 meeting 변경 21개, task 변경 18개, taskGuide 변경 14개의 작성자는 모두 joamksh였다. 여러 영역을 함께 변경한 커밋은 중복 집계되며, 과거 brief/login 등의 경로 변경은 이 패키지별 수치에 모두 포함되지 않는다. 따라서 이 수치는 핵심 담당 영역을 확인하는 보조 근거다.

### A. 프로젝트 초기 구동 기반과 패키지 구조 정리

- `fdef94b` (05-14): 기존 `com.example.codi` 코드 묶음을 정리하고 `com.codiapp.codi` 기반 Spring Boot/Gradle 실행 구조, Oracle JDBC 의존성, 간단한 저장소·실행 확인용 코드를 추가했다. 기존 업로드 이력이 있으므로 저장소 최초 작성자라는 뜻은 아니다.
- `5106894` (05-15): controller/converter/dto/entity/repository/service 예시 패키지와 global 패키지를 추가했다. 팀이 기능별로 코드를 넣을 기본 형태를 마련한 작업이다.
- `0505011` (05-18): 회의 관련 ID 생성에 Oracle 시퀀스를 적용하고 TEXT 매핑을 LOB로 바꾸는 호환성 수정을 했다. 이후 현재 코드의 ID 전략은 다시 IDENTITY이므로 현재 전체 시스템이 시퀀스 기반이라고 설명하면 안 된다.

### B. 회의록 도메인의 초기 설계부터 참석자 관리까지

1. `Meeting`, `Agenda`, `AgendaDetail`, `Decision` 엔티티와 관계를 만들었다 (`0743ac8`, `601d94b`). 제목·일시·장소, 안건, 안건별 상세 내용, 결정사항을 구조적으로 저장했다.
2. Repository, DTO, Converter, Command/Query Service, Controller를 순차적으로 만들고 생성·조회·수정·삭제 API를 구현했다 (`fe6bd93`, `74b8928`, `c875f5b`, `dc5b1fb`, `3cb4a04`, `aad1b65`, `46309c6`).
3. 회의 전체를 한 번에 수정하던 구조에서 안건·상세 내용·결정사항의 생성/수정/삭제를 개별 API로 분리했다 (`ab8497a`). 회의록 편집 중 특정 항목만 고칠 수 있게 한 변화다.
4. 상세 응답에 하위 항목 ID를 추가했다 (`87aeddf`). 조회한 화면에서 해당 항목을 식별해 수정·삭제할 수 있게 하는 변경이다.
5. 회의록 목록과 페이지 처리를 추가하고 이후 teamId로 조회 범위를 제한했다 (`18229e7`, `0c9c1b9`).
6. 수정 API가 엔티티 대신 상세 DTO를 반환하도록 변경하고, 날짜 문자열 파싱과 잘못된 날짜 예외를 추가했다 (`2e28922`). 메시지에는 로그인 연동이라고 적혀 있지만 실제 diff의 핵심은 요청·응답 및 날짜 처리다.
7. `MeetingAttendee`로 Meeting과 UserTeam을 연결해 참석자를 저장했다 (`90c0180`). 이어 팀원 후보 조회, 저장된 참석자 조회, 참석자 전체 교체, 회의 목록의 참석자 정보 반환을 구현했다 (`f2e51c7`, `09e6cc8`, `4749da7`, `b46b934`).

현재 참석자 수정은 기존 연결을 모두 삭제한 뒤 요청받은 팀원 연결을 새로 저장하는 방식이다. 증분 비교 방식은 아니다.

주요 코드: `domain/meeting/controller/MeetingController.java`, `MeetingSubItemController.java`, `service/MeetingCommandServiceImpl.java`, `MeetingAttendeeCommandServiceImpl.java`, `converter/MeetingConverter.java`.

### C. 학생 과제 관리와 제출 흐름

1. `Task`, `TaskDetail`, `TaskStatus`와 DTO를 설계하고 CRUD 계층을 구현했다 (`1f0877a`, `2ea002d`, `925d994`).
2. 과제의 세부 답안을 개별 생성·수정·삭제하는 API와 목록 조회를 보강했다 (`d9d8154`, `5fca495`, `18d7462`).
3. 수정 로직을 Converter에서 Task 엔티티의 `applyUpdate`로 옮겼다 (`758d03b`). 서비스는 대상 조회 후 엔티티에 변경을 위임하고 트랜잭션 안에서 변경 사항을 저장한다.
4. 목록 조회에 teamId를 전달하도록 Controller부터 Repository까지 연결했다 (`318f9aa`).
5. 교수자 안내(TaskGuide) 및 문항(TaskGuideDetail)을 학생 제출 데이터와 연결했다 (`9a1e6c1`). 최종 상세 응답에서 과제 안내·마감일·문항 설명과 학생 답안을 함께 내려준다.
6. 답안 ID와 본문 목록으로 여러 답안을 수정하는 API를 추가했다 (`72dd9d9`).
7. `IN_PROGRESS ↔ COMPLETE` 제출 상태 전환을 구현했다. COMPLETE 전환 시 현재 날짜를 기록하고 되돌리면 날짜를 비운다 (`b3a0e61`).
8. 교수자 확인용 목록에 수업·팀·상태 필터를 추가하고 페이지 처리 및 팀/수업 ID 응답을 보강했다 (`7f053d4`, `0add6a8`). 코드상 status 파라미터를 받으므로 완료 상태 외의 조회도 가능하다.

주요 코드: `domain/task/controller/TaskController.java`, `service/TaskCommandServiceImpl.java`, `TaskQueryServiceImpl.java`, `converter/TaskConverter.java`.

### D. 교수자 과제 안내와 수업별 팀 배포

1. 과제 안내와 문항 엔티티, 단건 조회, 생성 API를 구현했다 (`41e6204`, `bfd24df`, `342c8ea`).
2. 개발 중 TaskGuide → Brief → TaskGuide로 이름을 정리했다 (`07ab09c`, `4903a1f`). 별개 기능 세 개가 아니라 같은 기능의 명칭 변경이다.
3. 안내와 문항의 수정·삭제를 추가했다 (`9fa2c60`, `e6c6ef7`).
4. 안내의 연결 대상을 User에서 Course로 변경했다 (`05f9ab0` → 되돌리기 `764e1b5` → 재적용 `6853491`). 수업별 과제 배포가 가능한 모델로 바뀌었다. 되돌린 이유는 Git만으로 알 수 없다.
5. 문항을 ID 오름차순으로 조회하도록 정렬 기준을 추가하고 수업별 안내 목록을 구현했다 (`e89d705`, `144e805`). 사용자가 자유롭게 문항 순서를 지정하는 기능은 아니다.
6. **같은 수업의 모든 팀에 Task와 빈 TaskDetail을 생성하는 배포 로직을 구현했다** (`747814e`). 단순 CRUD를 넘어 교수자 안내와 학생 팀의 작업을 연결하는 핵심 변경이다.
7. 안내의 배포 상태와 응답 필드를 추가하고, 이미 생성된 안내의 재배포 요청을 예외로 처리했다 (`b1794c3`, `8458b91`, `5949899`). 이 상태 검사만으로 동시 요청의 중복 생성까지 방지된다고 단정할 수는 없다.

주요 코드: `domain/taskGuide/service/TaskGuideCommandServiceImpl.java`의 `generateTasks`, `createEmptyTasksForTeams`, `entity/TaskGuide.java`.

### E. 다른 도메인과 연결하고 API를 정리한 작업

- 학생/교수자 역할을 `UserRole` Enum으로 추가하고 회원가입 입력·저장에 연결했다 (`1a46a5f`). 역할 데이터 모델을 추가한 것이며 로그인 전체나 완전한 역할별 접근 제어를 구현했다는 의미는 아니다.
- 교수자 화면을 위한 courseId별 팀 목록을 추가했다 (`5ed7a7b`).
- 회의 참석자 선택을 위한 userTeamId·사용자 이름 조회를 추가했다 (`f2e51c7`).
- Swagger 설명을 보강했다 (`0154d51`). Swagger 최초 설정은 다른 작성자의 커밋이다.
- DTO 오타·중복 코드, URL, 에러 코드 이름을 정리했다 (`72e41f3`, `0ff48d3`, `5c85522`, `5949899`).
- PR 병합과 develop 반영 기록이 있다. `f925a6f`는 다른 작성자의 팀 워크스페이스 작업 병합이다. 병합 참여를 해당 기능의 직접 구현으로 계산하지 않았다. 병합 기록만으로 리뷰 내용이나 팀장 역할을 확정할 수는 없다.

## 3. 시기별로 복원한 작업 흐름

| 시기 | 진행한 일 |
| --- | --- |
| 05-14~05-15 | Spring Boot/Gradle 기반과 도메인 패키지 틀 정리 |
| 05-18 | 회의록과 과제 엔티티·DTO·저장소·서비스·API 집중 구현, Oracle 호환성 수정 |
| 05-21~05-25 | 회의 하위 항목과 과제 세부 항목의 독립 편집, 목록 조회, 응답 ID, Swagger 설명, 엔티티 변경 로직 정리 |
| 06-06~06-14 | 팀별 조회 연동, 회의 수정 요청·응답 보완, 학생/교수자 역할 추가 |
| 06-18 | 교수자 안내 모델 구현·명칭 정리·수업 연결, 학생 과제와 안내 연결, 수업 내 팀별 과제 생성 |
| 06-19 | 답안 수정·제출 상태·교수자 조회·배포 상태 보강, 팀 조회와 회의 참석자 기능 마무리 |

이력에서는 초기 데이터 모델과 CRUD를 먼저 만든 뒤, 개별 항목 편집을 가능하게 구조를 나누고, 마지막에 수업·팀·사용자와 연결해 실제 화면 흐름에 맞추는 작업 방식이 나타난다. 이는 변경 순서에 근거한 해석이며 당시 회의나 요구사항 문서를 확인한 것은 아니다.

## 4. 다른 팀원 작업과 구분

| 영역 | 이력상 주된 작성자/joamksh 기여 |
| --- | --- |
| 회의록·과제·과제 안내 | joamksh의 핵심 구현 영역 |
| 칸반·팀 워크스페이스 초기 기능 | hisemsem 계열 작성자 중심. joamksh는 초기 패키지 예시 및 후속 팀 조회 연결 |
| 일정·수업 | mk-star/MinGyeong Kim 계열 중심 |
| 로그인·회원가입·게시판 | kimSR0916 중심, 다른 작성자의 후속 수정도 존재. joamksh는 역할 Enum 추가 |
| 공통 응답·예외 처리 최초 기반 | `efc8e8c`, mk-star. joamksh는 담당 도메인의 핸들러·에러 코드 확장 |
| Swagger 최초 설정 | `3c06c53`, mk-star. joamksh는 개별 API 설명 보강 |

작성자 표시 이름이 달라도 이메일이 동일한 경우가 있다. 이 문서는 Git 작성자 기록을 근거로 하며 실제 공동 작업이나 페어 프로그래밍 여부는 알 수 없다.

## 5. 당시 작업을 설명할 때 사용할 수 있는 문장

> 수업 기반 팀 협업 서비스 Code-Codi의 Spring Boot 백엔드에서 회의록, 학생 과제 제출, 교수자 과제 안내 기능을 담당했습니다. 회의록을 안건·세부 내용·결정사항으로 나누어 개별 편집할 수 있도록 구현하고, 팀원 정보와 연결된 참석자 관리 기능을 개발했습니다. 교수자 과제 안내와 팀별 제출 데이터를 분리해 수업 내 각 팀에 빈 제출 양식을 생성하고, 답안 수정·제출 상태 전환·교수자 확인 목록까지 연결했습니다. 또한 초기 프로젝트 구조 정리와 팀별 조회, DTO 응답, 예외 처리 및 API 문서 보강에 참여했습니다.

구체적인 성과 수치, 성능 향상, 운영 배포, 이용자 수, 자동화 테스트 범위는 이 저장소만으로 확인되지 않는다. 테스트 소스에는 기본 `contextLoads` 하나가 확인된다. 현재 SecurityConfig는 전체 경로 허용 설정을 포함하므로 역할 Enum 추가를 보안 권한 체계 완성으로 표현하면 안 된다.

## 6. 재확인 명령

```powershell
git log --author=joamksh --no-merges --date=short --format="%h %ad %s"
git show ab8497a
git show 758d03b
git show 747814e
git show b3a0e61
git show 90c0180
git log --author=joamksh --merges --oneline
```

아래 부록은 분석 기준 HEAD에서 직접 추출한 비병합 커밋 전체 목록이다.

## 부록: joamksh 비병합 커밋 61개

| 커밋 | 날짜 | 메시지 |
| --- | --- | --- |
| fdef94b | 2025-05-14 | init : db 연결 및 기본 setting |
| 5106894 | 2025-05-15 | init : 도메인 구조 통일 |
| 0743ac8 | 2025-05-18 | feat  : 회의록 관련 엔티티추가 팀정보 제외 |
| 8ea6574 | 2025-05-18 | ref : 속성명 Id로 통일 |
| fe6bd93 | 2025-05-18 | feat : 회의록 repository 추가 |
| 601d94b | 2025-05-18 | ref : team 정보 추가 |
| 74b8928 | 2025-05-18 | feat : dto 생성 |
| c875f5b | 2025-05-18 | feat : converter 추가 |
| dc5b1fb | 2025-05-18 | feat : MeetingCommandService 생성 |
| 0505011 | 2025-05-18 | ref : 자동생성 mysql->오라클에 맞게 시퀀스 생성 및 text 대체 |
| 3cb4a04 | 2025-05-18 | feat : 생성, 단일 조회 |
| aad1b65 | 2025-05-18 | feat : 삭제 기능 추가 |
| 1f0877a | 2025-05-18 | feat : 엔티티와 dto 생성 |
| 2ea002d | 2025-05-18 | feat : converter 추가 |
| 925d994 | 2025-05-18 | feat : service, repository, controller 생성 CRD |
| 46309c6 | 2025-05-21 | ref : 응답 및 update |
| ab8497a | 2025-05-21 | ref : meeting, agenda, agendaDetail, Decision 도메인 cud 분리, 상세 수정 목적 |
| d9d8154 | 2025-05-23 | ref: 엔티티 수정 |
| 5fca495 | 2025-05-23 | ref: dto 및 수정 |
| 18d7462 | 2025-05-23 | ref: controller, service 수정 |
| 87aeddf | 2025-05-23 | ref: GetMapping 해당 속성 아이디 추가 |
| 18229e7 | 2025-05-23 | feat: MeetingList GetMapping |
| 0154d51 | 2025-05-25 | refactor: swagger 부가설명 |
| 758d03b | 2025-05-25 | refactor: 도메인 변경 로직 엔티티로 이동 |
| 0c9c1b9 | 2025-06-06 | feat: 회의록 리스트 teamId 반영 조회 |
| 2e28922 | 2025-06-13 | refactor: update 로그인 연동 수정 |
| 318f9aa | 2025-06-13 | refactor: teamId 전달 |
| 1a46a5f | 2025-06-14 | feat: 유저 Enum으로 역할구분 |
| 41e6204 | 2025-06-18 | feat: 엔티티 등록 |
| bfd24df | 2025-06-18 | feat: taskGuide 단일 조회 |
| b774259 | 2025-06-18 | refactor: detailTitle에서 title로 속성이름 변경 |
| 342c8ea | 2025-06-18 | feat: taskGuide, detail Create |
| 07ab09c | 2025-06-18 | refactor: 기능명 변경 taskGuide-> brief |
| a96f477 | 2025-06-18 | refactor: handler관련 코드 brief로 이름 변경 |
| 9fa2c60 | 2025-06-18 | feat: brief,detail Update 추가 |
| b0286c5 | 2025-06-18 | refactor: Task이름으로 복원 |
| 72e41f3 | 2025-06-18 | refactor: requestDto 오타 수정 및 중복코드 삭제 |
| e6c6ef7 | 2025-06-18 | feat: brief 삭제 |
| 05f9ab0 | 2025-06-18 | refactor: user에서 courser로 연결 변경 |
| 764e1b5 | 2025-06-18 | Revert "refactor: user에서 courser로 연결 변경" |
| 6853491 | 2025-06-18 | refactor: user에서 course로 연결 변경 |
| 4903a1f | 2025-06-18 | refactor: brief에서 taskGuide로 이름 수정 |
| e89d705 | 2025-06-18 | refactor: 수정 단계에서 새롭게 추가해도 정렬 유지 |
| 144e805 | 2025-06-18 | feat: 과제 생성 리스트 조회 |
| 9a1e6c1 | 2025-06-18 | refactor: 학생 과제 단일 조회 |
| 747814e | 2025-06-18 | feat: taskGuide기준 같은 course인 모든 team에 task학생 제출용 생성 |
| 72dd9d9 | 2025-06-19 | refactor: 최종 과제에서 수정 |
| b3a0e61 | 2025-06-19 | refactor: 학생입장 과제 리스트 변경 및 제출 상태 변경 |
| 7f053d4 | 2025-06-19 | feat: 교수자의 완료 과제 리스트 조회 |
| 0add6a8 | 2025-06-19 | refactor: 교수자 확인 리스트 페이지 설정 |
| 5ed7a7b | 2025-06-19 | feat: courseId로 team정보 조회 |
| 0ff48d3 | 2025-06-19 | refactor: 공지 url 중복글자 수정 |
| 5949899 | 2025-06-19 | refactor: errorStatus 대문자로 통일 |
| b1794c3 | 2025-06-19 | feat: TaskGuide에 status추가 공지여부 확인용 |
| 8458b91 | 2025-06-19 | feat: status 응답에 추가 |
| 90c0180 | 2025-06-19 | feat: meeting, userTeam사용해서 참여자들 저장 |
| f2e51c7 | 2025-06-19 | feat: 같은 팀 소속 userTeamId, username 리스트 조회 |
| 09e6cc8 | 2025-06-19 | feat: 저장된 meeting별 참석 유저 정보 리스트 조회 |
| 5c85522 | 2025-06-19 | refactor: url경로 통일 |
| 4749da7 | 2025-06-19 | feat: 회의 참석자 수정 |
| b46b934 | 2025-06-19 | refactor: meeting리스트 저장된 참가자 조회추가 |
