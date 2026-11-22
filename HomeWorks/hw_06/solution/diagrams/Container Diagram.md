```mermaid
C4Container
    title Диаграмма контейнеров ЕИС "Реестр граждан и льгот"

    Person(operator, "Сотрудник ОИВ", "Работает с витринами данных и отчетами")
    
    System_Ext(foiv, "ФОИВ / ОИВ", "Внешние системы (ЕРН, ФИАС, мЕРН, НСУД, ДСЗН, ЕМИАС)")

    System_Boundary(eis_boundary, "ЕИС РГиЛ") {
        Container(gateway, "API Gateway", "Spring Cloud Gateway", "Маршрутизация, Rate Limiting, JWT")
        
        Container_Boundary(integration, "Интеграционный слой") {
            Container(int_ingestion, "Internal Ingestion Service", "Spring Boot", "Прием из внутреннего Artemis")
        }
        
        Container_Boundary(core, "Бизнес-логика") {
            Container(profile, "Citizen Profile Service", "Spring Boot", "Мастер-данные граждан")
            Container(msp, "MSP Service", "Spring Boot", "Меры поддержки")
            Container(lk, "LK Service", "Spring Boot", "Льготные категории")
            Container(tx, "Transaction Service", "Spring Boot", "Транзакции")
        }
        
        Container_Boundary(analytics, "Аналитика и отчетность") {
            Container(airflow, "Apache Airflow", "Python", "Оркестратор для витрин")
            Container(report, "Reporting Service", "Spring Boot + PHP(Laravel)", "Фронт для отчетов")
        }
        
        ContainerQueue(artemis_ext, "Artemis External queues", "Apache Artemis", "Изоляция внешних интеграций")
        ContainerQueue(artemis_int, "Artemis Internal queues", "Apache Artemis", "Изоляция внутренних взаимодействий")
        
        Container_Boundary(db, "БД и кэш") {
            ContainerDb(pg_master, "PostgreSQL 19 (Master)", "PostgreSQL", "Мастер-данные (OLTP)")
            ContainerDb(pg_replica, "PostgreSQL 19 (Replica)", "PostgreSQL", "Реплика мастер-базы")
            ContainerDb(pg_dm, "PostgreSQL 19 (Data Marts)", "PostgreSQL", "Отдельная БД для витрин и отчетов")
            ContainerDb(redis, "Redis", "Redis", "Кэш профилей")
        }
    }

    %% Взаимодействия с внешними системами
    Rel(operator, gateway, "Запросы", "HTTPS/REST")
    Rel(foiv, gateway, "SOAP / REST", "HTTPS")
    Rel(foiv, artemis_ext, "JMS", "JMS")
    
    %% Шлюз
    Rel(gateway, artemis_ext, "Публикация событий", "JMS")
    Rel(gateway, report, "Маршрутизация", "REST")
    Rel(gateway, airflow, "Триггер DAG", "REST")
    
    %% Обработка очередей
    Rel(artemis_ext, int_ingestion, "Вычитка", "JMS")
    Rel(int_ingestion, artemis_int, "Публикация", "JMS")
    
    Rel(artemis_int, profile, "События", "JMS")
    Rel(artemis_int, msp, "События", "JMS")
    Rel(artemis_int, lk, "События", "JMS")
    Rel(artemis_int, tx, "События", "JMS")
    
    %% gRPC связи
    Rel(tx, profile, "Проверка гражданина", "gRPC")
    Rel(tx, msp, "Проверка меры поддержки", "gRPC")
    Rel(tx, lk, "Проверка льготной категории", "gRPC")
    
    Rel(msp, profile, "Данные профиля", "gRPC")
    Rel(lk, profile, "Данные профиля", "gRPC")
    
    %% Запись в мастер-базу
    Rel(profile, pg_master, "Запись", "JDBC")
    Rel(msp, pg_master, "Запись", "JDBC")
    Rel(lk, pg_master, "Запись", "JDBC")
    Rel(tx, pg_master, "Запись", "JDBC")
    Rel(profile, redis, "Кэш", "Redis Protocol")
    
    %% Репликация и витрины
    Rel(pg_master, pg_replica, "Репликация", "WAL Streaming")
    Rel(airflow, pg_replica, "Чтение", "JDBC")
    Rel(airflow, pg_dm, "Запись витрин", "JDBC")
    Rel(report, pg_dm, "Чтение отчетов", "JDBC")
```
