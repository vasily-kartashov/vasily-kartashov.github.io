---
layout: post
title: Getting started with Spring Data Envers
tags: java spring jpa
---

At some point of every company's adult life it starts reassessing the sanity and quality of its data set. One of the ways to regain the trust is to introduce auditing and versioning of database entities, which is exactly what Hibernate Envers is for.

As far as Envers is concerned all you need to do is to sparkle a couple of annotations around

```java
@Entity
@Table(name = "users")
@Audited
@AuditTable("users_audit")
public class User {
    ...
}
```

The majority of interesting features are hidden behind the scenes, including creation of the revision tracking table and `users_audit` table as well as storing the audit records when you write the data to the database. In a certain way Envers' unique selling proposition is to be as transparent as possible while adding audit storing capabilities. While adding auditing you're not required to change your database schema.

The hiding complex stuff behind a facade has implications. It's quite painful to fetch the objects back efficiently, and may require legwork in worst case scenario. On a positive side the default simple setup is quite trivial.

Project setup
---

Let's start with a simple application that has a set of users and these users can send invitations. When an invitation is being sent we want to capture the name, email of the inviter and the invitation email. If the user at some point changes her name or the email we still want to be able to get the original inviter data.

We'll need the following dependencies for our spring boot project

```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-envers</artifactId>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
</dependency>
```

_For some odd reason, you'll need to add `org.aspectj:aspectjweaver` dependency to
the project._

We'll start with the User entity

```java
@Entity
@Table(name = "users")
@Audited
@AuditTable("users_audit")
public class User {
    @Id
    @GeneratedValue
    private Long id;

    private String firstName, lastName, email;
}
```

For events we would like to be a tiny bit more flexible anticipating a huge set of different events ahead of us.

```java
@Entity
@Table(name = "events")
@Inheritance(strategy = InheritanceType.JOINED)
@Audited
public abstract class Event {
    @Id
    @GeneratedValue
    private Long id;
}

@Entity
@Table(name = "invitation_events")
@Audited
@AuditTable("invitation_events_audit")
public class InvitationEvent extends Event {
    @ManyToOne
    private User inviter;
    private String invitationEmail;
}
```

Envers didn't like `InheritanceType.TABLE_PER_CLASS`

Now to the `spring-data-envers` magic. Create repositories extending `RevisionRepository<E, ID, N>` like following

```java
public interface UserRepository extends
        RevisionRepository<User, Long, Integer>,
        CrudRepository<User, Long> {
}

public interface EventRepository extends
        RevisionRepository<Event, Long, Integer>,
        CrudRepository<User, Long> {
}
```

It's also important to change `repositoryFactoryBeanClass` while enabling JPA repositories

```java
@SpringBootApplication
@EnableJpaRepositories(repositoryFactoryBeanClass = EnversRevisionRepositoryFactoryBean.class)
public class Application {
    ...
}
```

You're all set. Let's run a simple query and see what Envers does for us:

```java
User user = new User();
user.setEmail("old-email@tradition.test");
userRepository.save(user);

InvitationEvent event = new InvitationEvent();
event.setInviter(user);
event.setInvitationEmail("invitation-email@kartashov.com");
eventRepository.save(event);
Long eventId = event.getId();

user.setEmail("completely-new-email@tradition.test");
userRepository.save(user);
```

At this point if we decide just to fetch the event by ID we'll get the link to the updated user row in table `users` and won't see the email the invitation email was sent from. This is where `RevisionRepository` comes into play. Let's fetch the latest revision of our event and see how the inviter's data looked like at the time the invitation was sent.

```java
Event event = eventRepository
        .findLastChangeRevision(eventId).getEntity();
assert event.getInviter().getEmail().equals("old-email@tradition.test");
```

Behind the scene
---

Let's see what goes behind the scenes. For `ddl-auto` set to `create`, `create-drop` or `update` Envers would create all the auxiliary tables on application start. For our case we will end up with the following selection

![Envers database schema](/public/images/envers.svg)

Let's see what are we asking our database to do for us when we're after the latest revision on an event

In the first query we're fetching all the revision IDs for the specified `:event_id`

```sql
    SELECT events_audit.revision_id
      FROM events_audit
CROSS JOIN revinfo
     WHERE events_audit.id = :event_id
       AND events_audit.revision_id = revinfo.rev
  ORDER BY events_audit.revision_id ASC
```

In the next step we're fetching all the revisions for the IDs that we just fetched, which certainly seems like a not such a smart thing to do considering that we could have done it in the first query. One possible explanation is that Envers lets you extend the `revinfo` entities and add fields to it, therefore prepares for worst case scenario.

```sql
SELECT revinfo.rev,
       revinfo.revtstmp
  FROM revinfo
 WHERE rev IN (:revision_ids)
```

In the next step we're dealing with this beauty, and whoever feels it's not a masterpiece needs an urgent appointment with a doctor

```sql
         SELECT events_audit.id AS id1_1_,
                events_audit.revision_id AS revision2_1_,
                events_audit.revision_type AS revision3_1_,
                invitation_events_audit.invitation_email,
                invitation_events_audit.user_id,
                CASE
                    WHEN invitation_events_audit.id IS NOT NULL THEN 1
                    WHEN events_audit.id IS NOT NULL THEN 0
                END AS clazz
           FROM events_audit
LEFT OUTER JOIN invitation_events_audit invitation_events_audit
             ON events_audit.id = invitation_events_audit.id
            AND events_audit.revision_id = invitation_events_audit.revision_id
          WHERE events_audit.revision_id = (
                SELECT MAX(ea.revision_id)
                  FROM events_audit ea
                 WHERE ea.revision_id <= ?
                   AND events_audit.id = ea.id)
            AND events_audit.revision_type <> :revision_type
            AND events_audit.id = :event_id
```

A lot of the clutter is caused by joined table inheritance strategy.

The last query is the lazily loaded user revision which pretty much mimics the last query only in this case we don't have to deal with inheritance hierarchies:

```sql
SELECT users_audit.id,
       users_audit.revision_id,
       users_audit.revision_type,
       users_audit.email,
       users_audit.first_name,
       users_audit.last_name
  FROM users_audit
 WHERE users_audit.revision_id = (
       SELECT max(ua.revision_id)
         FROM users_audit ua
        WHERE ua.revision_id <= :revision_id
          AND user_audit.id = ua.id)
   AND user_audit.revision_type <> :revision_type
   AND user_audit.id = :user_id
```

We are looking for a revision (create or update), based on `:user_id`. Our target `:revision_id` might not have affected the user row, therefore we're looking for `max(revision_id)` for this specific user, which is less than or equal to our target `:revision_id`.

Querying data
---

For querying the data you have a Criteria API which is similar to JPA Criteria API.
Now let's say we want to find all invitations that where sent from a specific email address, no matter who owns this email now and whether it's currently used by any user in the database. The best up-to-date documentation can be found in [Hibernate User Guide](http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#envers)

In our case unfortunately a surprise is waiting for us. Innocently looking query throws `UnsupportedOperationException`

```java
return AuditReaderFactory(entityManager)
        .createQuery()
        .forRevisionsOfEntity(InvitationEvent.class, true, false)
        .traverseRelation("inviter", JoinType.LEFT)
        .add(AuditEntry.property("email").eq(email))
        .getResultList();
```

Let's just hope that this will be addressed soon.
