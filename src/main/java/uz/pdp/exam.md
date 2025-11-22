/*
--------------------------- Questions ---------------------------

1. Database - Ma’lumotlarni tartibli saqlash, boshqarish va ulardan foydalanish uchun mo'ljallangan tizimli to'plam.

2. RDBMS - Relational DBMS — ma’lumotlar jadval ko'rinishida saqlanadi va jadvallar o'zaro bog'langan bo'ladi

4. Sql - Structured Query Language — ma’lumotlar bazasida so'rovlar yozish tili.

5. Primary key — jadvaldagi unikal va NULL bo'lmaydigan ustun.
   Foreign key — boshqa jadvalning primary key siga bog'lanadi.
   Farqi: Primary key identifikatsiya qiladi, foriegn esa bog'laydi.

6. PL/pgSQL - PostgreSQL’ning procedurali dasturlash tili (function, trigger, loop).

7. Assert - Shartni tekshiradi; false bo'lsa xatolik chiqaradi.

8. Trigger - INSERT, UPDATE, DELETE bo'lganda avtomatik ishga tushadigan funksiya

9. Tranzaksiya — bajarilishi kerak bo'lgan amallar ketma-ketligi.
   ACID: Atomicity, Consistency, Isolation, Durability — ishonchlilik tamoyillari.

10. Indexing - Qidiruvni tezlashtiruvchi ma’lumot tuzilmasi

*/

create table roles
(
id       serial primary key,
roleName varchar unique not null
);

insert into roles(roleName)
values ('Student'),
('Mentor');

create table users
(
id       serial primary key,
fullName varchar check ( length(fullName >= 5) ),
phone    varchar unique check (length(phone = 9)),
role_id  int,
foreign key (role_id) references roles (id)
);

create table groups
(
id        serial primary key,
groupName varchar check ( length(groupName >= 3) ),
mentor_id int,
foreign key (mentor_id) references users (id),
createdAt date not null default now()
);

create table students
(
id        serial primary key,
user_id   int,
foreign key (user_id) references users (id),
group_id  int,
foreign key (group_id) references groups (id),
createdAt date    not null default now(),
active    boolean not null default false
);

-- Standart view yaratish

create view getStudentsInfoView as
select s.id        as student_id,
u.fullName  as student_fullName,
s.createdAt as student_createdAt,
g.groupName as groupName
from students s
inner join users u on u.id = s.user_id
inner join groups g on g.id = s.group_id and g.mentor_id = s.user_id;

select * from getstudentsinfoview;

-- Materialized view yaratish

create materialized view getStudentsMaterializedView as
select g.id        as group_id,
g.groupName,
g.createdAt as group_createdAt,
count(s.id) as studentCount
from groups g
left join students s on g.id = s.group_id
group by g.id, g.groupName, g.createdAt
order by studentCount desc
limit 3;

select * from getstudentsmaterializedview;

-- Funksiya yaratish

create or replace function groupOfMentor(p_mentor_id int)
returns table(
mentor_id int,
mentor_fullName varchar,
groupName varchar,
group_createdAt date,
studentCount int
)
language sql
as
$$
select
g.mentor_id,
u.fullName as mentor_fullName,
g.groupName,
g.createdAt as group_createdAt,
count(s.id) as studentCount
from groups g
inner join users u on u.id = g.mentor_id
left join students s on s.group_id = g.id
where g.mentor_id = p_mentor_id
group by g.mentor_id, u.fullName, g.groupName, g.createdAt;
$$;

select groupOfMentor(1);

-- Procedure yaratish

create or replace procedure studentActivitor(p_group_id int)
as
$$
begin
update students
set active = true
where group_id = p_group_id
and active = false;

    raise notice 'Result = %', p_group_id;
end;
$$ language plpgsql;

call studentActivitor(1);

-- Trigger yaratish

create table update_logs
(
id       serial primary key,
old_data jsonb,
new_data jsonb
);

create or replace function log_updates()
returns trigger
as
$$
begin
insert into update_logs(old_data, new_data)
values (to_jsonb(old),
to_jsonb(new)
);
return new;
end;
$$ language plpgsql;

select * from update_logs;