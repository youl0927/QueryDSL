### 20240724
- QueryDSL 프로젝트 생성
- DB 커넥션 연결 확인 -> h2
- 테스트 엔티티 생성 후 테스트코드 작성
---
- 개발환경 세팅
   - 디펜던시 설정 (3.X)
```
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.2.0'
  id 'io.spring.dependency-management' version '1.1.4'
}

  group = 'study'
  version = '0.0.1-SNAPSHOT'
  sourceCompatibility = '17'

configurations {
  compileOnly {
    extendsFrom annotationProcessor
  }
}

repositories {
  mavenCentral()
}
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0
  compileOnly 'org.projectlombok:lombok'
  runtimeOnly 'com.h2database:h2'
  annotationProcessor 'org.projectlombok:lombok'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'

  //test 롬복 사용
  testCompileOnly 'org.projectlombok:lombok'
  testAnnotationProcessor 'org.projectlombok:lombok'

  //Querydsl 추가
  implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
  annotationProcessor "com.querydsl:querydsl-apt:$
  {dependencyManagement.importedProperties['querydsl.version']}:jakarta"
  annotationProcessor "jakarta.annotation:jakarta.annotation-api"
  annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}
tasks.named('test') {
  useJUnitPlatform()
}
clean {                                //추가해줘야 여기에 Q클래스가 생성됨
  delete file('src/main/generated')
}
```
- application.yml 파일 생팅
```
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/querydsl
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        # show_sql: true
        format_sql: true

logging.level:
  org.hibernate.SQL: debug
  # org.hibernate.type: trace
```
- JPAQueryFactory 필드일 경우
```
@Test
public void startQuerydsl2() {
   //member1을 찾아라.
   QMember m = new QMember("m");

   Member findMember = queryFactory
                              .select(m)
                              .from(m)
                              .where(m.username.eq("member1"))
                              .fetchOne();

   assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- 기본 Q-Type 활용하기
     - 이런식을 가능하지만, 보통 static import를 활용
```
QMember qMember = new QMember("m"); //별칭 직접 지정
QMember qMember = QMember.member; //기본 인스턴스 사용

@Test
public void startQuerydsl3() {
   //member1을 찾아라.

   Member findMember = queryFactory
                              .select(member)
                              .from(member)
                              .where(member.username.eq("member1"))
                              .fetchOne();

   assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- 월래는 쿼리 안에 QMember.member 이런식으로 엔티티를 넣어야 되지만 static import를 활용하면 QmMember 클래스 안에 name이 member인걸 확인 할수 있음
```
spring.jpa.properties.hibernate.use_sql_comments: true   //이걸 설정하면 QueryDSL -> JPQL로 변환된 쿼리를 확인 할 수 있
```

- 검색 조건 쿼리
```
@Test
public void search() {
   Member findMember = queryFactory
                              .selectFrom(member)
                              .where(member.username.eq("member1")
                              .and(member.age.eq(10)))
                              .fetchOne();

   assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- 다음은 QueryDSL 에서 활용 할 수 있는 검색조건이다
```
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'
member.username.isNotNull() //이름이 is not null
member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30
member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30
member.username.like("member%") //like 검색
member.username.contains("member") // like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색
```

- AND 조건을 파라미터로 처리 할 경우
```
@Test
public void searchAndParam() {
   List<Member> result1 = queryFactory
                              .selectFrom(member)
                              .where(member.username.eq("member1"),
                              member.age.eq(10))
                              .fetch();

   assertThat(result1.size()).isEqualTo(1);
}
```

- 다음에는 fetch 관련 결과 조회할때 사용하는 것이다
```
//List
List<Member> fetch = queryFactory
                           .selectFrom(member)
                           .fetch();

//단 건
Member findMember1 = queryFactory
                           .selectFrom(member)
                           .fetchOne();

//처음 한 건 조회
Member findMember2 = queryFactory
                           .selectFrom(member)
                           .fetchFirst();

//페이징에서 사용
QueryResults<Member> results = queryFactory
                                    .selectFrom(member)
                                    .fetchResults();

//count 쿼리로 변경
long count = queryFactory
                  .selectFrom(member)
                  .fetchCount();
```

- 정렬
   - desc()` , `asc()` : 일반 정렬
   - nullsLast()` , `nullsFirst()` : null 데이터 순서 부여
```
/**
* 회원 정렬 순서
* 1. 회원 나이 내림차순(desc)
* 2. 회원 이름 올림차순(asc)
* 단 2에서 회원 이름이 없으면 마지막에 출력(nulls last)
*/
@Test
public void sort() {
   em.persist(new Member(null, 100));
   em.persist(new Member("member5", 100));
   em.persist(new Member("member6", 100));

   List<Member> result = queryFactory
                              .selectFrom(member)
                              .where(member.age.eq(100))
                              .orderBy(member.age.desc(), member.username.asc().nullsLast())
                              .fetch();

   Member member5 = result.get(0);
   Member member6 = result.get(1);
   Member memberNull = result.get(2);

   assertThat(member5.getUsername()).isEqualTo("member5");
   assertThat(member6.getUsername()).isEqualTo("member6");
   assertThat(memberNull.getUsername()).isNull();
}
```

- 페이징
```
@Test
public void paging1() {
   List<Member> result = queryFactory
                              .selectFrom(member)
                              .orderBy(member.username.desc())
                              .offset(1) //0부터 시작(zero index)
                              .limit(2) //최대 2건 조회
                              .fetch();

   assertThat(result.size()).isEqualTo(2);
}
```

- 전체 조회 수가 필요하다면?
   - 이렇게 한다면 count 쿼리가 실행된다.
```
@Test
public void paging2() {
   QueryResults<Member> queryResults = queryFactory
                                             .selectFrom(member)
                                             .orderBy(member.username.desc())
                                             .offset(1)
                                             .limit(2)
                                             .fetchResults();

   assertThat(queryResults.getTotal()).isEqualTo(4);
   assertThat(queryResults.getLimit()).isEqualTo(2);
   assertThat(queryResults.getOffset()).isEqualTo(1);
   assertThat(queryResults.getResults().size()).isEqualTo(2);
}
```

- 집합
```
```java
/**
* JPQL
* select
* COUNT(m), //회원수
* SUM(m.age), //나이 합
* AVG(m.age), //평균 나이
* MAX(m.age), //최대 나이
* MIN(m.age) //최소 나이
* from Member m
*/
@Test
public void aggregation() throws Exception {
   List<Tuple> result = queryFactory
                              .select(member.count(),
                              member.age.sum(),
                              member.age.avg(),
                              member.age.max(),
                              member.age.min())
                              .from(member)
                              .fetch();
   
   Tuple tuple = result.get(0);
   
   assertThat(tuple.get(member.count())).isEqualTo(4);
   assertThat(tuple.get(member.age.sum())).isEqualTo(100);
   assertThat(tuple.get(member.age.avg())).isEqualTo(25);
   assertThat(tuple.get(member.age.max())).isEqualTo(40
   assertThat(tuple.get(member.age.min())).isEqualTo(10);
}
```

- GroupBy 사용
   - groupBy` , 그룹화된 결과를 제한하려면 `having`
```
/**
* 팀의 이름과 각 팀의 평균 연령을 구해라.
*/
@Test
public void group() throws Exception {
   List<Tuple> result = queryFactory
                           .select(team.name, member.age.avg())
                           .from(member)
                           .join(member.team, team)
                           .groupBy(team.name)
                           .fetch();

   Tuple teamA = result.get(0);
   Tuple teamB = result.get(1);

   assertThat(teamA.get(team.name)).isEqualTo("teamA");
   assertThat(teamA.get(member.age.avg())).isEqualTo(15);
   assertThat(teamB.get(team.name)).isEqualTo("teamB");
   assertThat(teamB.get(member.age.avg())).isEqualTo(35);
}

//groupBy(), having() 예시
.groupBy(item.price)
.having(item.price.gt(1000))
```

- 조인
   - join()` , `innerJoin()` : 내부 조인(inner join)
   - leftJoin()` : left 외부 조인(left outer join)
   - rightJoin()` : rigth 외부 조인(rigth outer join)
```
/**
* 팀 A에 소속된 모든 회원
*/
@Test
public void join() throws Exception {
   QMember member = QMember.member;
   QTeam team = QTeam.team;

   List<Member> result = queryFactory
                              .selectFrom(member)
                              .join(member.team, team)
                              .where(team.name.eq("teamA"))
                              .fetch();

   assertThat(result)
   .extracting("username")
   .containsExactly("member1", "member2");
}
```

- 세타 조인
   - from 절에 여러 엔티티를 선택해서 세타 조인
   - 외부 조인 불가능 다음에 설명할 조인 on을 사용하면 외부 조인 가능
```
/**
* 세타 조인(연관관계가 없는 필드로 조인)
* 회원의 이름이 팀 이름과 같은 회원 조회
*/
@Test
public void theta_join() throws Exception {
   em.persist(new Member("teamA"));
   em.persist(new Member("teamB"));

   List<Member> result = queryFactory
                              .select(member)
                              .from(member, team)
                              .where(member.username.eq(team.name))
                              .fetch();

   assertThat(result)
   .extracting("username")
   .containsExactly("teamA", "teamB");
}
```
---

- 조인 on
   - 연관관계 없는 엔티티를 조인할때 주로 사용함
```
1, 조인 대상 필터링
/**
* 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
* JPQL: SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'teamA'
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and
 t.name='teamA'
  */
 @Test
 public void join_on_filtering() throws Exception {
     List<Tuple> result = queryFactory
             .select(member, team)
             .from(member)
             .leftJoin(member.team, team).on(team.name.eq("teamA"))
             .fetch();
     for (Tuple tuple : result) {
         System.out.println("tuple = " + tuple);
   }
}

2, 연관관계 없는 엔티티 외부 조인
/**
* 2. 연관관계 없는 엔티티 외부 조인
* 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
* JPQL: SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name */
 @Test
 public void join_on_no_relation() throws Exception {
     em.persist(new Member("teamA"));
     em.persist(new Member("teamB"));
     List<Tuple> result = queryFactory
             .select(member, team)
             .from(member)
             .leftJoin(team).on(member.username.eq(team.name))
             .fetch();
     for (Tuple tuple : result) {
         System.out.println("t=" + tuple);
   }
}
```

- 페치 조인
   - `join(), leftJoin()` 등 조인 기능 뒤에 `fetchJoin()` 이라고 추가하면 된다.
```
@PersistenceUnit
EntityManagerFactory emf;

@Test
 public void fetchJoinNo() throws Exception {         //페치조인 미적용
     em.flush();
     em.clear();
     Member findMember = queryFactory
             .selectFrom(member)
             .where(member.username.eq("member1"))
             .fetchOne();
     boolean loaded = emf.getPersistenceUnitUtil().isLoaded(findMember.getTeam());   // 이미 로딩된 entity인지 초기화 안된 entity인지 확인해주는 기능
      assertThat(loaded).as("페치 조인 미적용").isFalse();
}
--------------------------------------------------------------------------------------

@Test
 public void fetchJoinUse() throws Exception {
     em.flush();
     em.clear();
     Member findMember = queryFactory
             .selectFrom(member)
             .join(member.team, team).fetchJoin()
             .where(member.username.eq("member1"))
             .fetchOne();
     boolean loaded = emf.getPersistenceUnitUtil().isLoaded(findMember.getTeam());
assertThat(loaded).as("페치 조인 적용").isTrue(); }
}
```

- 서브 쿼리 사용
   - `com.querydsl.jpa.JPAExpressions` 사용 (static import로 대채 가능)
```
서브 쿼리 eq 사용
/**
* 나이가 가장 많은 회원 조회 */
@Test
public void subQuery() throws Exception {
   QMember memberSub = new QMember("memberSub");

   List<Member> result = queryFactory
                  .selectFrom(member)
                  .where(member.age.eq(
                     JPAExpressions                    //static import 가능
                        .select(memberSub.age.max())
                        .from(memberSub)
                  ))
                  .fetch();

   assertThat(result).extracting("age").containsExactly(40);
}

--------------------------------------------------------------------------------------

서브쿼리 goe 사용
/**
* 나이가 평균 나이 이상인 회원 */
@Test
public void subQueryGoe() throws Exception {
   QMember memberSub = new QMember("memberSub");

   List<Member> result = queryFactory
                  .selectFrom(member)
                  .where(member.age.goe(
                     JPAExpressions
                           .select(memberSub.age.avg())
                           .from(memberSub)
                  ))
                  .fetch();

   assertThat(result).extracting("age").containsExactly(30,40);
}

--------------------------------------------------------------------------------------

서브쿼리 여러 건 사용 in 사용
/**
* 서브쿼리 여러 건 처리, in 사용 */
@Test
public void subQueryIn() throws Exception {
   QMember memberSub = new QMember("memberSub");

   List<Member> result = queryFactory
                  .selectFrom(member)
                  .where(member.age.in(
                           JPAExpressions
                                 .select(memberSub.age)
                                 .from(memberSub)
                                 .where(memberSub.age.gt(10))
                  ))
                  .fetch();

   assertThat(result).extracting("age").containsExactly(20, 30, 40);
}

--------------------------------------------------------------------------------------

select 절에 subquery사용
 List<Tuple> fetch = queryFactory
         .select(member.username,
                 JPAExpressions
                         .select(memberSub.age.avg())
                         .from(memberSub)
         ).from(member)
         .fetch();

 for (Tuple tuple : fetch) {
     System.out.println("username = " + tuple.get(member.username));
     System.out.println("age = " + tuple.get(JPAExpressions.select(memberSub.age.avg()).from(memberSub)));
}

--------------------------------------------------------------------------------------

static import 활용
 import static com.querydsl.jpa.JPAExpressions.select;

 List<Member> result = queryFactory
         .selectFrom(member)
         .where(member.age.eq(
                 select(memberSub.age.max())
.from(memberSub)
)) .fetch();
```

- Case문 활용
   - select, 조건절(where), order by에서 사용 가능
```
단순조건
List<String> result = queryFactory
         .select(member.age
            .when(10).then("열살")
            .when(20).then("스무살")
            .otherwise("기타"))
         .from(member)
         .fetch();

--------------------------------------------------------------------------------------

복잡한 조건
List<String> result = queryFactory
        .select(new CaseBuilder()
               .when(member.age.between(0, 20)).then("0~20살")
               .when(member.age.between(21, 30)).then("21~30살")
               .otherwise("기타"))
        .from(member)
        .fetch();

--------------------------------------------------------------------------------------

orderBy에서 Case문 함께 사용하기 예제
예를 들어서 다음과 같은 임의의 순서로 회원을 출력하고 싶다면? 1. 0~30살이아닌회원을가장먼저출력
2. 0~20살회원출력
3. 21~30살회원출력
 NumberExpression<Integer> rankPath = new CaseBuilder()
         .when(member.age.between(0, 20)).then(2)
         .when(member.age.between(21, 30)).then(1)
         .otherwise(3);

 List<Tuple> result = queryFactory
         .select(member.username, member.age, rankPath)
         .from(member)
         .orderBy(rankPath.desc())
         .fetch();

 for (Tuple tuple : result) {
     String username = tuple.get(member.username);
     Integer age = tuple.get(member.age);
     Integer rank = tuple.get(rankPath);
     System.out.println("username = " + username + " age = " + age + " rank = " +rank);
}

--------------------------------------------------------------------------------------

상수, 문자 더하기
상수가 필요하면 `Expressions.constant(xxx)` 사용
Tuple result = queryFactory
         .select(member.username, Expressions.constant("A"))
         .from(member)
         .fetchFirst();

--------------------------------------------------------------------------------------

문자 더하기 concat
 String result = queryFactory
         .select(member.username.concat("_").concat(member.age.stringValue()))
         .from(member)
         .where(member.username.eq("member1"))
         .fetchOne();
```
---

- 프로젝션 대상이 하나일 경우
```
List<String> result = queryFactory
   .select(member.username)
   .from(member)
   .fetch();
```

- 튜플 조회
```
List<Tuple> result = queryFactory
      .select(member.username, member.age)
      .from(member)
      .fetch();

for (Tuple tuple : result) {
   String username = tuple.get(member.username);
   Integer age = tuple.get(member.age);
   System.out.println("username=" + username);
   System.out.println("age=" + age);
}
```

- 프로젝션 결과 반환 - DTO
```
DTO 생성
@Data
public class MemberDto {
   private String username;
   private int age;

   public MemberDto() {
   }

   public MemberDto(String username, int age) {
      this.username = username;
      this.age = age;
   }
}

조회 (JPQL)
List<MemberDto> result = em.createQuery(
                  "select new study.querydsl.dto.MemberDto(m.username, m.age) from Member m", MemberDto.class)
                  .getResultList();

조회(query DSL)
프로퍼티 접근 - Setter
List<MemberDto> result = queryFactory
         .select(Projections.bean(MemberDto.class,
                  member.username,
                  member.age))
         .from(member)
         .fetch();

필드 직접 접
List<MemberDto> result = queryFactory
         .select(Projections.bean(MemberDto.class,
                  member.username,
                  member.age))
         .from(member)
         .fetch();

별칭이 다를때
@Data
public class UserDto {
   private String name;
   private int age;
}

List<UserDto> fetch = queryFactory
         .select(Projections.fields(UserDto.class,
                  member.username.as("name"),                     
                  ExpressionUtils.as(
                              JPAExpressions
                                    .select(memberSub.age.max())
                                    .from(memberSub), "age")
                  )
      ).from(member)
      .fetch();

생성자 사용
List<MemberDto> result = queryFactory
         .select(Projections.constructor(MemberDto.class,
                  member.username,
                  member.age))
         .from(member)
         .fetch();
}
```

- 프로젝션 결과 반환 - @QueryProjection
```
@Data
public class MemberDto {
   private String username;
   private int age;

   public MemberDto() {
   }

   @QueryProjection                              //이 어노테이션을 써줘야 QDto가 클래스가 생성담
   public MemberDto(String username, int age) {
   this.username = username;
   this.age = age;
   }

}


List<MemberDto> result = queryFactory
      .select(new QMemberDto(member.username, member.age))
      .from(member)
```

- distinct
```
List<String> result = queryFactory
      .select(member.username).distinct()
      .from(member)
      .fetch();
```

- 동적쿼리
```
@Test
public void 동적쿼리_BooleanBuilder() throws Exception {
   String usernameParam = "member1";
   Integer ageParam = 10;

   List<Member> result = searchMember1(usernameParam, ageParam);
   Assertions.assertThat(result.size()).isEqualTo(1);
   }

   private List<Member> searchMember1(String usernameCond, Integer ageCond) {
   BooleanBuilder builder = new BooleanBuilder();

   if (usernameCond != null) {
      builder.and(member.username.eq(usernameCond));
   }

   if (ageCond != null) {
      builder.and(member.age.eq(ageCond));
   }

   return queryFactory
            .selectFrom(member)
            .where(builder)
            .fetch();
}
```

- 동적쿼리 - where 다중 파라미터 사용
   - where 조건에 null 값은 무시된다.
   - 메서드를 다른 쿼리에서도 재활용 할 수 있다.
   - 쿼리 자체의 가독성이 높아진다.
```
@Test
public void 동적쿼리_WhereParam() throws Exception {
String usernameParam = "member1";
Integer ageParam = 10;

List<Member> result = searchMember2(usernameParam, ageParam);
Assertions.assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember2(String usernameCond, Integer ageCond) {
   return queryFactory
               .selectFrom(member)
               .where(usernameEq(usernameCond), ageEq(ageCond))
               .fetch();
}

private BooleanExpression usernameEq(String usernameCond) {
   return usernameCond != null ? member.username.eq(usernameCond) : null;
}

private BooleanExpression ageEq(Integer ageCond) {
   return ageCond != null ? member.age.eq(ageCond) : null;
}

-----------------------------------------------------------------------------------------
private BooleanExpression allEq(String usernameCond, Integer ageCond) {
   return usernameEq(usernameCond).and(ageEq(ageCond));
}
이런식으로 조합도 가능하지만 null 체크를 해줘야
```

- 수정, 삭제 벌크 연산
```
쿼리 한번으로 대량 데이터 수정할 경우
long count = queryFactory
      .update(member)
      .set(member.username, "비회원")
      .where(member.age.lt(28))
      .execute();

기존 숫자에 1 더하기
long count = queryFactory
      .update(member)
      .set(member.age, member.age.add(1))
      .execute();

쿼리 한번으로 대량 데이터 삭제
long count = queryFactory
      .delete(member)
      .where(member.age.gt(18))
      .execute();
```

- SQL function 호출 하는 방법
```
member -> M으로 변경하는 replace 함수 사용
String result = queryFactory
         .select(Expressions.stringTemplate("function('replace', {0}, {1}, {2})", member.username, "member", "M"))
         .from(member)
         .fetchFirst();

소문자로 변경해서 비교해라
String result = queryFactory
         .select(member.username)
         .from(member)
         .where(member.username.eq(Expressions.stringTemplate("function('lower', {0})", member.username)))
         .fetchFirst();

lower 같은 ansi 표준 함수들은 querydsl이 상당부분 내장하고 있다. 따라서 다음과 같이 처리해도 결과는 같다.
String result = queryFactory
         .select(member.username)
         .from(member)
         .where(member.username.eq(member.username.lower()))
         .fetchFirst();
```

---

- 순수 JPA레파짓토리와 QueryDSL 레파짓 토리의 차이점
   - 순수 JPA 레파짓토리 및 테스트
```
package study.querydsl.repository;
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.dsl.BooleanExpression;
import com.querydsl.jpa.impl.JPAQueryFactory;
import org.springframework.stereotype.Repository;
import study.querydsl.dto.MemberSearchCondition;
import study.querydsl.dto.MemberTeamDto;
import study.querydsl.dto.QMemberTeamDto;
import study.querydsl.entity.Member;
import javax.persistence.EntityManager;
import java.util.List;
import java.util.Optional;
import static org.springframework.util.StringUtils.hasText;
import static org.springframework.util.StringUtils.isEmpty;
import static study.querydsl.entity.QMember.member;
import static study.querydsl.entity.QTeam.team;
@Repository
public class MemberJpaRepository {

   private final EntityManager em;
   private final JPAQueryFactory queryFactory;

   public MemberJpaRepository(EntityManager em) {
      this.em = em;
      this.queryFactory = new JPAQueryFactory(em);
   }

   public void save(Member member) {
      em.persist(member);
   }

   public Optional<Member> findById(Long id) {
      Member findMember = em.find(Member.class, id);
      return Optional.ofNullable(findMember);
   }

   public List<Member> findAll() {
      return em.createQuery("select m from Member m", Member.class)
                        .getResultList();
   }

   public List<Member> findByUsername(String username) {
      return em.createQuery("select m from Member m where m.username = :username", Member.class)
                  .setParameter("username", username)
                  .getResultList();
   }
}

-----------------------------------------------------------------------------------------
테스트

package study.querydsl.repository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;
import study.querydsl.dto.MemberSearchCondition;
import study.querydsl.dto.MemberTeamDto;
import study.querydsl.entity.Member;
import study.querydsl.entity.Team;
import javax.persistence.EntityManager;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Transactional
class MemberJpaRepositoryTest {
   @Autowired
   EntityManager em;

   @Autowired
   MemberJpaRepository memberJpaRepository;

   @Test
   public void basicTest() {
      Member member = new Member("member1", 10);
      memberJpaRepository.save(member);

      Member findMember = memberJpaRepository.findById(member.getId()).get();
      assertThat(findMember).isEqualTo(member);

      List<Member> result1 = memberJpaRepository.findAll();
      assertThat(result1).containsExactly(member);

      List<Member> result2 = memberJpaRepository.findByUsername("member1");
      assertThat(result2).containsExactly(member);
   }
}
```
   - QueryDSL 레파짓토리
```
public List<Member> findAll_Querydsl() {
   return queryFactory
            .selectFrom(member).fetch();
}

public List<Member> findByUsername_Querydsl(String username) {
   return queryFactory
            .selectFrom(member)
            .where(member.username.eq(username))
            .fetch();
}

-----------------------------------------------------------------------------------------
테스트

@Test
public void basicQuerydslTest() {
   Member member = new Member("member1", 10);
   memberJpaRepository.save(member);

   Member findMember = memberJpaRepository.findById(member.getId()).get();
   assertThat(findMember).isEqualTo(member);

   List<Member> result1 = memberJpaRepository.findAll_Querydsl();
   assertThat(result1).containsExactly(member);

   List<Member> result2 = memberJpaRepository.findByUsername_Querydsl("member1");
   assertThat(result2).containsExactly(member);
}
```

      - JPAQueryFactory 스프링 빈 등록
```
@Bean
JPAQueryFactory jpaQueryFactory(EntityManager em) {
   return new JPAQueryFactory(em);
}
```

- QueryDSL 용 DTO 추가
   - 꼭 QueryProjection를 붙여줘야됨 그럼 Q클래스가 생성
```
package study.querydsl.dto;
import com.querydsl.core.annotations.QueryProjection;
import lombok.Data;
@Data
public class MemberTeamDto {
   private Long memberId;
   private String username;
   private int age;
   private Long teamId;
   private String teamName;

   @QueryProjection
   public MemberTeamDto(Long memberId, String username, int age, Long teamId,
      String teamName) {
      this.memberId = memberId;
      this.username = username;
      this.age = age;
      this.teamId = teamId;
      this.teamName = teamName;
   }
}
```

- builder 를 사용한 동적쿼리 예제
```
//Builder 사용
//회원명, 팀명, 나이(ageGoe, ageLoe)
public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {

   BooleanBuilder builder = new BooleanBuilder();

   if (hasText(condition.getUsername())) {
      builder.and(member.username.eq(condition.getUsername()));
   }

   if (hasText(condition.getTeamName())) {
      builder.and(team.name.eq(condition.getTeamName()));
   }

   if (condition.getAgeGoe() != null) {
      builder.and(member.age.goe(condition.getAgeGoe()));
   }

   if (condition.getAgeLoe() != null) {
      builder.and(member.age.loe(condition.getAgeLoe()));
   }
   return queryFactory
      .select(new QMemberTeamDto(
               member.id,
               member.username,
               member.age,
               team.id,
               team.name))
      .from(member)
      .leftJoin(member.team, team)
      .where(builder)
      .fetch();
}

-----------------------------------------------------------------------------------------
테스트
@Test
public void searchTest() {
   Team teamA = new Team("teamA");
   Team teamB = new Team("teamB");
   em.persist(teamA);
   em.persist(teamB);

   Member member1 = new Member("member1", 10, teamA);
   Member member2 = new Member("member2", 20, teamA);
   Member member3 = new Member("member3", 30, teamB);
   Member member4 = new Member("member4", 40, teamB);

   em.persist(member1);
   em.persist(member2);
   em.persist(member3);
   em.persist(member4);

   MemberSearchCondition condition = new MemberSearchCondition();
   condition.setAgeGoe(35);
   condition.setAgeLoe(40);
   condition.setTeamName("teamB");

   List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);
   assertThat(result).extracting("username").containsExactly("member4");
}
```

- 동적쿼리를 활용한 where 절 파라미터 사용
```
//회원명, 팀명, 나이(ageGoe, ageLoe)
public List<MemberTeamDto> search(MemberSearchCondition condition) {
   return queryFactory
         .select(new QMemberTeamDto(
                  member.id,
                  member.username,
                  member.age,
                  team.id,
                  team.name))
         .from(member)
         .leftJoin(member.team, team)
         .where(usernameEq(condition.getUsername()),
                  teamNameEq(condition.getTeamName()),
                  ageGoe(condition.getAgeGoe()),
                  ageLoe(condition.getAgeLoe()))
         .fetch();
}

private BooleanExpression usernameEq(String username) {
   return isEmpty(username) ? null : member.username.eq(username);
}

private BooleanExpression teamNameEq(String teamName) {
   return isEmpty(teamName) ? null : team.name.eq(teamName);
}

private BooleanExpression ageGoe(Integer ageGoe) {
   return ageGoe == null ? null : member.age.goe(ageGoe);
}

private BooleanExpression ageLoe(Integer ageLoe) {
   return ageLoe == null ? null : member.age.loe(ageLoe);
}

-----------------------------------------------------------------------------------------
참고로 where 절에 파라미터 방식을 사용하면 조건 재사용 가능
//where 파라미터 방식은 이런식으로 재사용이 가능하다.
public List<Member> findMember(MemberSearchCondition condition) {
   return queryFactory
         .selectFrom(member)
         .leftJoin(member.team, team)
         .where(usernameEq(condition.getUsername()),
               teamNameEq(condition.getTeamName()),
               ageGoe(condition.getAgeGoe()),
               ageLoe(condition.getAgeLoe()))
         .fetch();
}
```

- 예제 api 개발
   - 컨트롤러 
```
package study.querydsl.controller;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import study.querydsl.dto.MemberSearchCondition;
import study.querydsl.dto.MemberTeamDto;
import study.querydsl.repository.MemberJpaRepository;
import java.util.List;

@RestController
@RequiredArgsConstructor
public class MemberController {

   private final MemberJpaRepository memberJpaRepository;
   
   @GetMapping("/v1/members")
   public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {
      return memberJpaRepository.search(condition);
   }
}
```
