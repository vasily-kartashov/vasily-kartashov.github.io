---
layout: post
title: Spring data JPA and inheritance
tags: java spring jpa
---

Let's start adding user management to our application. There are two kind of personal information that we want to capture at the moment: existing users and invitations sent out for new potential customers.
Invitations and Users have a lot in common so let's start with entities for our Spring Boot application

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Person {

    private @Id @GeneratedValue(strategy = GenerationType.TABLE) Long id;
    private String firstName;
    private String lastName;
    private String email;

    ...
}

@Entity
public class Invitation extends Person {

    ...

}

@Entity
public class User extends Person {

    ...

}
```

Let's start with keeping invitations and users in completely separate tables by using `InheritanceType.TABLE_PER_CLASS`

To work with the set of data create a simple repository

```java
public interface PersonRepository extends JpaRepository<Person, Long> {

    Page<Person> findAll(Pageable pageable);
}
```

If we now issue a simple Pageable request

```java
Pageable pageable = new PageRequest(0, 10, Sort.Direction.ASC, "firstName", "lastName");
Page<Person> firstPage = personRepository.findAll(pageable);
for (Person person : firstPage) {
    System.out.println(person);
}
```

So let's see what kind of queries the database issues in the background by placing the following `application.properties` in `/src/test/resources`

```properties
spring.jpa.show-sql = true
spring.jpa.properties.hibernate.format_sql = true

logging.level.org.hibernate.SQL = DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder = TRACE
```

If we stay with `TABLE_PER_CLASS` strategy the hibernate issues the following query

```sql
SELECT person0_.id AS id1_1_,
       person0_.email AS email2_1_,
       person0_.first_name AS first_na3_1_,
       person0_.last_name AS last_nam4_1_,
       person0_.date AS date1_0_,
       person0_.clazz_ AS clazz_
  FROM ( SELECT id,
                email,
                first_name,
                last_name,
                null AS date,
                1 AS clazz_
           FROM user UNION ALL
         SELECT id,
                email,
                first_name,
                last_name,
                date,
                2 AS clazz_
           FROM invitation
       ) person0_
 ORDER BY person0_.first_name ASC,
          person0_.last_name ASC
 LIMIT ?
```

For `SINGLE_TABLE` strategy we get

```sql
SELECT person0_.id AS id2_0_,
       person0_.email AS email3_0_,
       person0_.first_name AS first_na4_0_,
       person0_.last_name AS last_nam5_0_,
       person0_.date AS date6_0_,
       person0_.dtype AS dtype1_0_
  FROM person person0_
 ORDER BY person0_.first_name ASC,
          person0_.last_name ASC
 LIMIT ?
```

And for the `JOINED` strategy we get

```sql
SELECT person0_.id AS id1_1_,
       person0_.email AS email2_1_,
       person0_.first_name AS first_na3_1_,
       person0_.last_name AS last_nam4_1_,
       person0_2_.date AS date1_0_,
       CASE
           WHEN person0_1_.id IS NOT NULL THEN 1
           WHEN person0_2_.id IS NOT NULL THEN 2
           WHEN person0_.id IS NOT NULL THEN 0
       END as clazz_
  FROM person person0_
  LEFT OUTER JOIN user person0_1_
               ON person0_.id=person0_1_.id
  LEFT OUTER JOIN invitation person0_2_
               ON person0_.id=person0_2_.id
 ORDER BY person0_.first_name ASC,
          person0_.last_name ASC
 LIMIT ?
```

That's it for now
