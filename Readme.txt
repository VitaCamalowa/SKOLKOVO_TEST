Созданная структура включает в себя таблицы трех типов: таблица персон, справочные таблицы и таблицы записей. Первый тип представлен единственной таблицей FIOS, в которой хранятся имя и фамилия, id, информация о статусе персоны и дата внесения/вынесения.
Справочники – таблицы APPOINTS и ROOMS и DEPS служат для хранения постоянного набора стабильных записей и содержат информацию, которую после можно будет соотносить с персоной.
APPOINTID, ROOMID, DEPID, PHONES и EMAILS – таблицы служащие для соотнесения данных(по id), а так же хранения постоянно пополняемой информации, которую бесполезно создавать в виде справочника. Так же здесь есть информация о статусе, авторе изменений и их времени. В этих таблицах с помощью статусов осуществляются все действия с персоной. Например, создается запись о человеке N, работающем в департаменте ИТ-технологий в кабинете 1 на должности «стажер». Активный статус записи – 0. После перевода человека на должность «младший специалист» запись становится неактуальной – меняет статус на 99, а так же создается новая запись со статусом 0 и актуальным соотнесением информации. Таким образом карьерный путь этого человека можно будет отследить.

База разработана на PgAdmin 4 и сохранена в формате sql 

Запросы для тестирования
Поиск всех людей из комнаты 2599

select f.firstname, f.lastname from roomed r left join fios f on r.fio_id=f.id
where r.room_id=2599
and r.status=0
and f.status=0


Добавление нового телефона сотруднику 'Pedro Davidson'
insert into phones
(phone, fio_id, id, status, in_date)
select '+79162640803',(select id from fios where lastname='Davidson'
					  and firstname='Pedro') fio_id, (select max(id)+1 from phones) ph,0,current_timestamp
Проверка добавления телефона сотруднику 'Pedro Davidson'
select ph.phone from fios f left join phones ph on f.id=ph.fio_id
where f.lastname='Davidson' and f.firstname='Pedro'
and f.status=0 
and ph.status=0

получение списка телефонов всех сотрудников определенной должности в одном из отделов
select phone from phones where fio_id in
(select fio_id from deped 
 where dep_id in (select distinct id from deps where dep_name='Sales' and status=0)
and status=0)
and fio_id in
(select fio_id from appointed where ap_id in (select distinct id from appoints where ap_name='Manager' and status=0)
and status=0)

Добавление нового сотрудника
Функция, если не подгрузилась при импорте базы
Create or replace FUNCTION ADD_NEW_AP (dep varchar(50), email varchar(50), firstn varchar(50), lastn varchar(50), phon varchar(50), app varchar(50), fr integer)
RETURNS integer AS $$
Declare
ivan_id integer;
begin
select max(id)+1 into ivan_id from fios;
-- Внесем его ФИ
insert into fios (id, firstname, lastname, status, in_date)
select ivan_id, firstn,lastn,0,current_timestamp;
--Внесем его отдел
insert into deped (id, dep_id, fio_id,who_did, status, in_date)
select ( select max(id)+1 from deped), (select id from deps where dep_name=dep and status=0),ivan_id,0,0,current_timestamp;
-- Внесем его email
insert into emails (id, fio_id,email, status, in_date)
select ( select max(id)+1 from emails),ivan_id,email,0,current_timestamp;
-- Внесем его телефон, дополнительные в случае чего добавим позже
insert into phones (id, fio_id,phone, status, in_date)
select ( select max(id)+1 from phones),ivan_id,phon,0,current_timestamp;
--Внесем его должность
insert into appointed (id, ap_id, fio_id,who_did, status, in_date)
select ( select max(id)+1 from appointed), (select id from appoints where ap_name=app and status=0),ivan_id,0,0,current_timestamp;
--Внесем его комнату
insert into roomed (id, room_id, fio_id, who_did, status, in_date)
select ( select max(id)+1 from roomed), (select r.id from rooms r where r.room=fr and status=0),ivan_id,0,0,current_timestamp;
return 0;
end;
$$ LANGUAGE plpgsql;
запрос на запись
select add_new_ap('Sales','ivanov@mail.ru', 'Ivan','Ivanov', '+77777777777', 'Manager',2599)

Запрос для получения последнего сотрудника (для проверки предыдущего запроса)

select dp.dep_name, e.email, f.firstname, f.lastname, ph.phone, ap.ap_name, rs.room from fios f left join appointed a on f.id=a.fio_id 
left join appoints ap on a.ap_id=ap.id 
left join deped d on f.id=d.fio_id 
left join deps dp on d.dep_id=dp.id 
left join roomed r on f.id=r.fio_id 
left join rooms rs on r.room_id=rs.id 
left join emails e on e.fio_id=f.id
left join phones ph on ph.fio_id=f.id
where f.id=(select max(id) from fios )

Функция и запрос для перевода сотрудника в другой отдел и комнату (пример - перевод Betty Davis из IT в Sales из 1069 в 243 комнату. Заметим, что запрос не обрабатывает отделы и комнаты, не использованные ранее, они должны быть сперва созданы)

Create or replace FUNCTION MOVE_W (dep1 varchar(50), dep2 varchar(50), firstn varchar(50), lastn varchar(50), room1 integer, room2 integer)
RETURNS integer AS $$
Declare
ivan_id integer;
begin
select id into ivan_id from fios where firstname=firstn and lastname=lastn; -- вычислим id
-- Если отдел, из которого надо вывести человека - не указан, выведем из всех, иначе - только из указанного
if(dep1 is null or dep1='') then
update deped 
SET status = 99 WHERE fio_id=ivan_id;
else 
update deped 
SET status = 99 WHERE fio_id=ivan_id and dep_id=(select id from deps where status=0 and dep_name=dep1);
end if;
insert into deped (id, fio_id, dep_id, status, who_did, in_date)
select ( select max(id)+1 from deped),ivan_id,(select id from deps where dep_name=dep2 and status =0),0,0,current_timestamp;
-- если комната не указана, то выведем из всех, иначе - только из указанной
if(room1 is null or room1=0) then
update roomed 
SET status = 99 WHERE fio_id=ivan_id;
else 
update roomed  
SET status = 99 WHERE fio_id=ivan_id and room_id=(select id from rooms where status=0 and room=room1);
end if;
insert into roomed  (id, fio_id, room_id, status, who_did, in_date)
select ( select max(id)+1 from roomed),ivan_id,(select id from rooms where room=room2 and status =0),0,0,current_timestamp;

return 0;
end;
$$ LANGUAGE plpgsql;

select MOVE_W('IT', 'Sales','Betty', 'Davis', 1069, 243)

Запрос для проверки результатов работы запроса по переводу

select dp.dep_name, e.email, f.firstname, f.lastname, ph.phone, ap.ap_name, rs.room from fios f left join appointed a on f.id=a.fio_id 
left join appoints ap on a.ap_id=ap.id 
left join deped d on f.id=d.fio_id 
left join deps dp on d.dep_id=dp.id 
left join roomed r on f.id=r.fio_id 
left join rooms rs on r.room_id=rs.id 
left join emails e on e.fio_id=f.id
left join phones ph on ph.fio_id=f.id
where f.firstname='Betty' and f.lastname='Davis'
and d.status=0
and r.status=0
