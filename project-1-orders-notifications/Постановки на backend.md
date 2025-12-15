> **Примечание:**  
> Все данные, приведённые в данной постановке на разработку, были изменены и приведены к нейтральному виду для соблюдения конфиденциальности. Любые совпадения с реальными данными случайны.  
> Документация отражает логику работы методов и примеры данных в обобщённой форме.


# **1. POST /api/v3/orders**
Эндпоинт предназаначен для создания заказа

## **Входные/выходные параметры**

| №  | Имя параметра    | Тип (Java) | Описание                                 | Обязательное |
| -- | ---------------- | ---------- | ---------------------------------------- | ------------ |
| 1  | orderName        | string     | Наименование заказа                      | да           |
| 2  | orderTypeId      | int        | Идентификатор вида заказа                | да           |
| 3  | actTypeId        | int        | Идентификатор вида работ ТОРО            | да           |
| 4  | plantId          | int        | Идентификатор завода                     | да           |
| 5  | techPlaceId      | int        | Идентификатор технического места         | да           |
| 6  | equipmentId      | int        | Идентификатор единицы оборудования       | нет          |
| 7  | priorityId       | int        | Приоритет заказа                         | да           |
| 8  | workCenterId     | int        | Идентификатор участка                    | да           |
| 9  | planGroupId      | int        | Идентификатор группы планирования        | да           |
| 10 | startDate        | timestamp  | Запланированная дата начала              | да           |
| 11 | endDate          | timestamp  | Запланированная дата окончания           | нет          |
| 12 | defectId         | int        | Идентификатор дефекта                    | нет          |
| 13 | defectIds        | array[int] | Список идентификаторов дефектов          | нет          |
| 14 | techCardId       | int        | Идентификатор технологической карты      | нет          |
| 15 | requiresApproval | boolean    | Признак необходимости утверждения заказа | нет          |
| 16 | orderDescription | string     | Описание заказа                          | нет          |
| 17 | createDate       | timestamp  | Дата создания записи                     | нет          |

### **Логика работы метода создания заказа**
1. Проверка входных параметров

1.1. Проверить заполнение поля `techPlaceId`.  
Если поле не заполнено — вернуть уведомление:  
**«Не заполнен код технического места»**

1.2. Проверить заполнение поля `equipmentId`.  
Если `equipmentId` не заполнено, заказ создаётся по указанному `techPlaceId`.

2. Проверки по связанным сообщениям (defects)

Если во входных параметрах передан массив `defectIds`, необходимо выполнить следующие проверки:

2.1. Проверить, что все переданные идентификаторы сообщений существуют в таблице `DEFECTS` (`id`).  
Если проверка не пройдена — вернуть ошибку:  
**HTTP 409**  
**Сообщение:** «Некоторые сообщения не найдены, создание заказа невозможно»

2.2. Проверить, что все сообщения имеют допустимый статус для создания заказа:  
`DEFECTS.status_id = 13` (в работе)  
Если проверка не пройдена — вернуть ошибку:  
**HTTP 409**  
**Сообщение:** «Некоторые сообщения имеют недопустимый статус для создания заказа»

2.3. Проверить, что все сообщения относятся к одному заводу.  
Значения `DEFECTS.plant_id` у всех переданных сообщений должны совпадать.  
Если проверка не пройдена — вернуть ошибку:  
**HTTP 409**  
**Сообщение:** «Заводы сообщений не совпадают, создание заказа невозможно»

2.4. Проверить, что все сообщения относятся к одной группе плановиков.  
Значения `DEFECTS.plangroup_id` у всех переданных сообщений должны совпадать.  
Если проверка не пройдена — вернуть ошибку:  
**HTTP 409**  
**Сообщение:** «Группа плановиков сообщений не совпадает, создание заказа невозможно»

2.5. Проверить, что все сообщения относятся к одному участку.  
Значения `DEFECTS.work_center_id` у всех переданных сообщений должны совпадать.  
Если проверка не пройдена — вернуть ошибку:  
**HTTP 409**  
**Сообщение:** «Участки сообщений не совпадают, создание заказа невозможно»

3. Создание записи в таблице `ORDER`

3.1. Создать запись в таблице `ORDER` со следующими значениями:

| Поле                | Значение                                                  |
| ------------------- | --------------------------------------------------------- |
| `id`                | Уникальный идентификатор                                  |
| `tp_id`             | `techPlaceId`                                             |
| `equip_id`          | `equipmentId`                                             |
| `order_type_id`     | `orderTypeId`                                             |
| `plant_id`          | `plantId`                                                 |
| `priority_new_id`   | `priorityId`                                              |
| `status_id`         | `1`                                                       |
| `id_job_code`       | `actTypeId`                                               |
| `plangroup_id`      | `planGroupId`                                             |
| `order_number`      | Автоинкремент +1 от последнего заказа (формат `00000001`) |
| `order_name`        | `orderName`                                               |
| `start_date`        | `startDate`                                               |
| `end_date`          | `endDate`                                                 |
| `create_date`       | Текущие дата и время                                      |
| `work_center_id`    | `workCenterId`                                            |
| `creator_id`        | `creatorId`                                               |
| `defect_id`         | `defectId`                                                |
| `tech_card_id`      | `techCardId`                                              |
| `order_description` | `orderDescription`                                        |
| `requires_approval` | `requiresApproval` (`true / false`)                       |

4. Связь заказа с сообщениями

4.1. Для каждого идентификатора из массива defectIds создать запись в таблице ORDER_DEFECTS:

| Поле          | Значение                        |
| ------------- | ------------------------------- |
| `id`          | Уникальный идентификатор        |
| `order_id`    | Идентификатор созданного заказа |
| `defect_id`   | Идентификатор сообщения         |
| `create_date` | Дата создания                   |

5. Обновление статуса сообщений

5.1. Найти все сообщения, связанные с заказом:  
`DEFECTS.id = ORDER_DEFECTS.defect_id`

5.2. Обновить статус найденных сообщений:  
`DEFECTS.status_id = 14`


# **2. PATCH /api/v1/orders/{orderId}**
Эндпоинт предназаначен для редактирования заказа

## **Входные/выходные параметры**
|    № | Имя параметра    | Тип    | Обязательное | Описание                        |
| ---: | ---------------- | ------ | ------------ | ------------------------------- |
|  1.1 | orderId          | int    | да           | Идентификатор заказа            |
|  1.2 | orderName        | string | да           | Наименование заказа             |
|  1.3 | orderTypeId      | int    | да           | ИД вида заказа                  |
|  1.4 | actTypeId        | int    | да           | ИД вида работ ТОРО              |
|  1.5 | plantId          | int    | да           | ИД завода                       |
|  1.6 | techPlaceId      | string | да           | ИД технического места           |
|  1.7 | equipmentId      | int    | да           | Идентификатор оборудования      |
|  1.8 | priopityId       | int    | да           | Приоритет                       |
|  1.9 | planGroupId      | int    | да           | Идентификатор группы плановиков |
| 1.10 | startDate        | date   | да           | Запланированная дата начала     |
| 1.11 | endDate          | date   | да           | Запланированная дата окончания  |
| 1.12 | defectNumber     | string | нет          | Идентификатор дефекта           |
| 1.13 | workCenterId     | int    | да           | Участок                         |
| 1.14 | orderDescription | string | нет          | Описание заказа                 |

|   № | Имя параметра    | Тип    | Обязательное | Описание             |
| --: | ---------------- | ------ | ------------ | -------------------- |
| 1.1 | id               | int    | да           | Идентификатор заказа |
| 1.2 | errorDescription | string | нет          | Описание ошибки      |

### **Логика**
При вызове метода выполняется проверка и последовательность действий:

1. Проверить существование заказа:

1.1. Существует ли в `ORDERS` поле `id == orderId`, переданное на вход.

1.2. Если запись не найдена — возвращать ошибку с текстом:  
**«Заказ с номером n не найден»**

2. Проверить заполнение поля `techPlaceId`:

2.1. Если поле не заполнено, выводить уведомление:  
**«Не заполнен код технического места»**

3. Проверить заполнение поля `equipmentId`:

3.1. Если поле не заполнено, создавать заказ по заполненному `techPlaceId`

4. Обновить запись в таблице `ORDER` (только измененные поля):

4.1. `id_tp` — если получено входное значение `techPlaceId`  
4.2. `eq_id` — если получено входное значение `equipmentId`  
4.3. `plantId` — если получено входное значение `plantId`  
4.4. `priorityId` — если получено входное значение `priorityId`  
4.5. `order_type_id` — если получено входное значение `orderTypeId`  
4.6. `id_job_code` — если получено входное значение `actTypeId`  
4.7. `plangroup` — если получено входное значение `plangroup`  
4.8. `id_status` — записать идентификатор == 1  
4.9. `order_num` — сформировать счетчик +1 от последнего `ORDER/id`, например: 00000001  
4.10. `start_date` — если получено входное значение `startDate`  
4.11. `end_date` — если получено входное значение `endDate`  
4.12. `create_date` — дата и время формирования записи  
4.13. `work_center_id` — если получено входное значение `workCenterId`  
4.14. `order_description` — входное значение `orderDescription`



# **3. GET api/v1/orders - метод получения списка заказов**

Данный эндпоинт позволяет получить список заказов в системе ТОРО (Техническое Обслуживание и Ремонт Оборудования).  
С помощью входных параметров можно фильтровать заказы по различным критериям:  
- по дате начала и окончания плановых работ,  
- по заводу, участку или группе плановиков,  
- по типу заказа и виду работ,  
- по оборудованию или техническому месту,  
- по приоритету, номеру заказа или дефекту.  


   ## **Входные параметры**
   
| №  | Имя параметра        | Тип          | Обязательное | Описание                                         |
|----|--------------------|-------------|--------------|------------------------------------------------|
| 1  | page               | int         | да           | Номер страницы                                  |
| 2  | size               | int         | да           | Кол-во записей на странице                      |
| 3  | sort               | string      | нет          | Параметр сортировки                             |
| 4  | sortType           | string      | нет          | Тип сортировки                                  |
| 5  | startDate          | date        | нет          | Период «С» - плановая дата начала              |
| 6  | endDate            | date        | нет          | Период «По» - плановая дата окончания          |
| 7  | workCenterId       | array(int)  | нет          | Фильтрация по участку                           |
| 8  | planGroupId        | array(int)  | нет          | Фильтрация по группе планирования              |
| 9  | plantId            | array(int)  | да           | Фильтрация по заводу                            |
| 10 | statusId           | array(int)  | нет          | Фильтрация по статусу                           |
| 11 | priorityId         | array(int)  | нет          | Фильтрация по приоритету                        |
| 12 | equipmentId        | array(int)  | нет          | Фильтрация по ЕО                                |
| 13 | techPlaceId        | array(int)  | нет          | Фильтрация по ТМ                                |
| 14 | orderTypeId        | array(int)  | нет          | Фильтрация по виду заказа                       |
| 15 | actTypeId          | array(int)  | нет          | Фильтрация по виду работ                         |
| 16 | equipmentKind      | array(int)  | нет          | Фильтрация по виду оборудования                 |
| 17 | orderNumber        | array(int)  | нет          | Фильтрация по номеру заказа                     |
| 18 | defectNumber       | array(int)  | нет          | Фильтрация по номеру дефекта                     |
| 19 | defectIds          | array(int)  | нет          | Фильтрация по ID дефекта                         |
| 20 | createDateFrom     | date        | нет          | Фильтрация по дате создания «С»                |
| 21 | createDateTo       | date        | нет          | Фильтрация по дате создания «По»               |
| 22 | actualStart        | date        | нет          | Фильтрация по фактической дате начала          |
| 23 | actualEnd          | date        | нет          | Фильтрация по фактической дате окончания       |
| 24 | closeDate          | date        | нет          | Плановая дата окончания выполнения заказа      |


## **Выходные параметры** 

| №  | Имя параметра        | Тип          | Обязательное | Описание                                         |
|----|--------------------|-------------|--------------|------------------------------------------------|
| 1  | errorDescription   | string      | нет          | Описание ошибки (только если метод отличается от 2XX) |
| 2  | id                 | int         | да           | Идентификатор заказа                            |
| 3  | orderNumber        | string      | нет          | Номер заказа                                    |
| 4  | orderName          | string      | нет          | Наименование заказа                             |
| 5  | orderTypeId        | int         | нет          | Идентификатор типа заказа                        |
| 6  | orderTypeName      | string      | нет          | Тип заказа                                      |
| 7  | statusId           | int         | нет          | Идентификатор статуса                            |
| 8  | statusName         | string      | нет          | Наименование статуса                             |
| 9  | equipmentId        | int         | нет          | Идентификатор оборудования                       |
| 10 | equipmentName      | string      | нет          | Наименование оборудования                        |
| 11 | eqCode             | string      | нет          | Код оборудования                                 |
| 12 | equipKindId        | int         | нет          | Ид вида оборудования                             |
| 13 | equipKindName      | string      | нет          | Наименование вида оборудования                   |
| 14 | startDate          | timestamp   | нет          | Дата и время начала (план)                        |
| 15 | endDate            | timestamp   | нет          | Дата и время окончания (план)                     |
| 16 | createDate         | timestamp   | нет          | Дата и время создания                              |
| 17 | actualStart        | timestamp   | нет          | Дата и время начала (факт)                         |
| 18 | actualEnd          | timestamp   | нет          | Дата и время окончания (факт)                      |
| 19 | hasExecutor        | boolean     | нет          | Признак наличия исполнителей                        |
| 20 | planMen            | string      | нет          | Количество человек (человеко-часы)                 |
| 21 | countMen           | string      | нет          | Количество человек, назначенных на заказ           |
| 22 | planTime           | string      | нет          | Количество времени (человеко-часы)                 |
| 23 | priorityDefectId   | int         | нет          | Идентификатор приоритета дефекта                   |
| 24 | priorityDefectName | string      | нет          | Наименование приоритета дефекта                    |
| 25 | planGroupId        | int         | нет          | Идентификатор группы плановиков                     |
| 26 | planGroupName      | string      | нет          | Наименование группы плановиков                      |
| 27 | actTypeId          | int         | нет          | ИД вида работ ТОРО                                   |
| 28 | actTypeName        | string      | нет          | Наименование вида работ ТОРО                         |
| 29 | plantId            | int         | нет          | Номер завода                                         |
| 30 | plantName          | string      | нет          | Наименование завода                                  |
| 31 | workCenterId       | int         | нет          | Id участка                                          |
| 32 | workCenterName     | string      | нет          | Наименование участка                                 |
| 33 | techPlaceId        | int         | нет          | Id тех. места                                        |
| 34 | techPlaceCode      | string      | нет          | Код тех. места                                       |
| 35 | techPlaceName      | string      | нет          | Наименование тех. места                               |
| 36 | defectId           | int         | нет          | Идентификатор дефекта                                  |
| 37 | defectNumber       | string      | нет          | Номер дефекта                                        |
| 38 | defects            | array       | нет          | Массив с данными о сообщении                           |
| 39 | requiresApproval   | bool        | нет          | Индикатор заказа, который требует утверждения        |
| 40 | priorityOrderId    | int         | нет          | Идентификатор приоритета заказа                        |
| 41 | priorityOrderName  | string      | нет          | Наименование приоритета заказа                         |
| 42 | total             | int          | нет          | Кол-во заказов переданных на выход                     |
| 43 | totalCurrent      | int          | нет          | Кол-во заказов текущих                                 |
| 44 | totalExpired      | int          | нет          | Кол-во просроченных заказов                             |
| 45 | totalDone         | int          | нет          | Кол-во выполненных заказов                               |
| 46 | page              | int          | нет          | Номер страницы                                          |
| 47 | size              | int          | нет          | Кол-во заказов на странице                               |
| 48 | totalAll          | int          | нет          | Кол-во всех заказов                                      |
| 49 | planDuration      | string       | нет          | Плановое время выполнения заказа                         |
| 50 | actualDuration    | string       | нет          | Фактическое время выполнения заказа 


### Алгоритм работы метода получения списка заказов

1.1. Проверить входные значения `page` и `size`.  
Допустимы только значения больше 0.

1.2. Если проверка не пройдена, вернуть ошибку:  
**«Некорректный запрос данных»**.

1.3. Рассчитать смещение (offset) по формуле:  
`offset = (page − 1) * size`.

1.4. Выбрать `size` записей, пропустив `offset` записей.

2.1. Проанализировать входные параметры `startDate` и `endDate`.

2.2. Если заполнены оба параметра — перейти к п.3.

2.3. Если параметры не заполнены — перейти к п.4.

3.1. Сформировать массив объектов `Orders` в соответствии с параметрами пагинации.

3.2. Собрать уникальные значения `ORDERS.id`, где `ORDERS.start_date` входит в период `<startDate, endDate>`.

3.3. Собрать уникальные значения `ORDERS.id`, где `ORDERS.create_date` входит в период `<createDateFrom, createDateTo>`.

3.4. Собрать уникальные значения `ORDERS.id`, где `ORDERS.start_date < текущая дата`.

4.1. `id` — `ORDERS.id`  
4.2. `orderNumber` — `ORDERS.order_number`  
4.3. `orderName` — `ORDERS.order_name`  
4.4. `orderTypeId` — `ORDERS.id_order_type`  
4.5. `orderTypeName` — `ORDER_TYPE.order_name`, где `ORDER_TYPE.object_type = ORDERS.id_order_type`  
4.6. `statusId` — `ORDERS.id_status`  
4.7. `statusName` — `STATUS.status_name`, где `STATUS.id = ORDERS.id_status`  
4.8. `equipmentId` — `ORDERS.id_equip`  
4.9. `equipmentName` — `EQUIPMENT.eq_name`  
4.10. `eqCode` — `EQUIPMENT.eq_code`  
4.11. `startDate` — `ORDERS.start_date`  
4.12. `endDate` — `ORDERS.end_date`  
4.13. `actualStart` — `ORDERS.actual_start`  
4.14. `actualEnd` — `ORDERS.actual_end`  
4.15. `priorityOrderId` — `ORDERS.priority_id`  
4.16. `priorityOrderName` — `PRIORITIES.priority_name`  
4.17. `planGroupId` — `ORDERS.plangroup_id`  
4.18. `planGroupName` — `PLANGROUPS_T.name`  
4.19. `actTypeId` — `ORDERS.id_job_code`  
4.20. `actTypeName` — `ORDER_TYPES.job_name`  
4.21. `plantId` — `ORDERS.plant_id`  
4.22. `plantName` — `PLANTS_T.name`  
4.23. `workCenterId` — `ORDERS.work_center_id`  
4.24. `workCenterName` — `WORK_CENTERS_T.name`  
4.25. `techPlaceId` — `ORDERS.tp_id`  
4.26. `techPlaceName` — `TECH_PLACES_T.name`  
4.27. `techPlaceCode` — `TECH_PLACE.tp_code`

5.1. Если заказ связан с дефектом (`ORDERS.defect_id`), вернуть:  
- `defectId`  
- `defectNumber`

5.2. Если заказ связан с несколькими дефектами через `ORDER_DEFECTS`, вернуть массив `defects` с полями:  
- `defectId`  
- `defectNumber`

6.1. `equipKindId` — `EQUIPMENT.kind_id`  
6.2. `equipKindName` — `EQUIPMENT_KINDS_T.name`  
6.3. `createDate` — `ORDERS.create_date`  
6.4. `planDuration` — разница между `ORDERS.end_date` и `ORDERS.start_date`  
6.5. `actualDuration` — `ORDERS.actual_duration`

7.1. `hasExecutor` — признак наличия исполнителей, если существует связка «Заказ → Операция → Исполнитель»  
7.2. `planMen` — сумма `OPERATION.executor_number` по операциям заказа  
7.3. `planTime` — сумма `OPERATION.work_duration` по операциям заказа  
7.4. `countMen` — количество исполнителей, назначенных на операции заказа

8.1. `planGroupId` — `ORDERS.plangroup_id`  
8.2. `plantId` — `ORDERS.plant_id`  
8.3. `statusId` — `ORDERS.status_id`  
8.4. `priorityId` — `ORDERS.priority_id`  
8.5. `equipmentId` — `ORDERS.id_equip`  
8.6. `techPlaceId` — `ORDERS.tp_id`  
8.7. `orderTypeId` — `ORDERS.order_type_id`  
8.8. `actTypeId` — `ORDERS.id_job_code`  
8.9. `workCenterId` — `ORDERS.work_center_id`  
8.10. `equipmentKind` — через `EQUIPMENT.kind_id`  
8.11. `orderNumber` — `ORDERS.id`  
8.12. `defectNumber` — `ORDERS.defect_id`  
8.13. `startDate` — `ORDERS.start_date >= startDate`  
8.14. `endDate` — `ORDERS.start_date <= endDate`  
8.15. `closeDate` — `ORDERS.end_date <= closeDate`  
8.16. `actualStart` — `ORDERS.actual_start`  
8.17. `actualEnd` — `ORDERS.actual_end`

9.1. `total` — количество записей в ответе  
9.2. `totalAll` — количество записей с учётом фильтров (без `statusId`)  
9.3. `totalCurrent` — заказы не в статусах «Выполнено», «Отклонен»  
9.4. `totalExpired` — просроченные заказы  
9.5. `totalDone` — заказы со статусом «Выполнено»  
9.6. `page` — номер страницы  
9.7. `size` — количество записей

10.1. Если переданы `sort` и `sortType`, выполнить сортировку по полям:  
`orderNumber`, `statusId`, `priorityOrderId`, `startDate`  

10.2. `asc` — по возрастанию  
10.3. `desc` — по убыванию



11.1. Некорректные параметры — ошибка **«Не удалось получить данные»**  
11.2. Некорректные необязательные параметры — **«Переданы некорректные параметры. Необходимо передать userId.»**

12.1. `requiresApproval` — значение `ORDERS.requires_approval`


