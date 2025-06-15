## Spring / MyBatis 환경에서 소스 클린징 사례 정리

---

## 📘 SCM이란 무엇인가?

**SCM(Software Configuration Management)**는 소스코드, 설정, 외부 의존성, 환경 정보 등을 일관되게 관리하여 **소프트웨어 시스템의 신뢰성과 재현 가능성을 높이기 위한 환경 관리 체계**입니다.

### ✅ 왜 SCM이 중요한가?

* 개발자의 로컬 환경에서는 동작하지만, CI/CD 또는 Full Build 환경에서는 실패하는 문제 방지
* 형상 일관성 확보 (예: 사용하지 않는 import, 누락된 Bean 등록 등)
* 협업 시 코드 기준을 통일하여 **팀 생산성** 향상
* 배포 전 정적 품질 확보 → 고객사 이슈 최소화

### 🧩 주요 적용 대상

* Spring 기반 DI 환경 설정 (Bean 등록, Transaction 범위)
* MyBatis Mapper의 SQL 매핑, 파라미터 바인딩
* Spring Batch, Listener, Job 파라미터 흐름
* 전체 Build 수행 시 오류 유발 가능성이 있는 잠재적 결함

---

## ✅ 사례별 실전 오류 클린징 정리

### 🔸 Case 1. @Transactional(REQUIRES\_NEW) AOP 적용 오류

**현상:** REQUIRES\_NEW 사용에도 불구하고 트랜잭션이 분리되지 않음

```java
// 오류: private 메서드는 AOP가 적용되지 않음
@Transactional(propagation = Propagation.REQUIRES_NEW)
private void someTxMethod() {...} 
```

```java
// 개선: public 접근자로 선언 시 정상 분리됨
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void someTxMethod() {...}
```

**원인 설명:** Spring의 AOP는 Proxy 기반이며 `public` 메서드에만 적용 가능 → `private` 또는 `final`에서는 트랜잭션 분리 미작동

---

### 🔸 Case 2. MyBatis SQL 매핑 오류 (jdbcType 누락)

**현상:** 쿼리 실행 시 ORA 오류 또는 결과 없음

```xml
<!-- 오류: jdbcType 누락 또는 $ 사용 -->
where a.user_id = ${userId}
```

```xml
<!-- 개선: #{} + jdbcType 지정 -->
where a.user_id = #{userId, jdbcType=VARCHAR}
```

**원인 설명:**

* `${}` 사용 시 SQL 인젝션 위험 + PreparedStatement가 아닌 문자 치환으로 동작
* `jdbcType`이 없을 경우 null 처리나 형변환 오류 발생 가능

---

### 🔸 Case 3. Spring Batch Listener의 NPE 발생

**현상:** Listener 내부에서 선언되지 않은 값 사용 → NPE 발생

```java
@Override
public void beforeStep(StepExecution stepExecution) {
    this.jobDate = stepExecution.getExecutionContext().getString("jobYmd"); // NPE 발생 가능
}
```

```java
// 개선
if (ctx.containsKey("jobYmd")) {
    this.jobDate = ctx.getString("jobYmd");
} else {
    this.jobDate = getToday();
}
```

**원인 설명:** Listener는 Step 실행 전후 Hook이며, `ExecutionContext`에서 참조할 key가 미리 저장되어 있지 않으면 NPE 발생

---

## 📋 SCM 정합성 확보를 위한 클린징 체크리스트

| 체크 항목       | 설명                                                    |
| ----------- | ----------------------------------------------------- |
| 트랜잭션 접근자    | `@Transactional` 사용 시 public 접근자 확인 (AOP 적용 여부 확인)    |
| SQL 바인딩     | MyBatis의 `#{}` 내부 `jdbcType` 명시 여부, `${}` 오용 방지       |
| Bean 누락     | 스프링 XML/Java Config에서 누락된 Bean 등록 여부 확인               |
| Listener 흐름 | beforeStep, getStepExecutionContext 사용 시 key 존재 여부 확인 |
| 불필요한 Import | 사용하지 않는 import, 미사용 메서드 제거                            |
| 로컬 전용 코드 제거 | 개발자 PC에서만 동작하는 경로, 파일 참조 제거                           |

---

## 🔚 마무리 및 향후 계획

이 문서는 사내 시스템의 Full Build 환경 구축 과정에서 발견된 **SCM 정합성 오류 사례**를 기반으로 정리되었으며, 유사한 Spring/MyBatis 환경에서 **빌드 신뢰도 향상과 유지보수성 개선**을 위해 반복 활용될 수 있습니다.
