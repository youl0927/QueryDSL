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
