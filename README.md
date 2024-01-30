# Тестовое задание на позицию junior backend разработчика

Вам необходимо разработать систему **сменных заданий**.
Её функционал заключается в том, чтобы получать сменные задания (партии) и уникальные идентификаторы продукции в рамках этой партии, а так же проверять (по запросу из внешней системы), принадлежит ли данный идентификатор продукции данной партии.

Язык программирования -- Python, фреймворк -- FastAPI, БД -- PostgreSQL.

Для выполнения этого задания используйте этот репозиторий как template (кнопка ``Use this template``), перед отправкой заполните следующие пункты:

**Основные задачи:**

- [ ] Эндпойнт добавления сменных заданний
- [ ] Эндпойнт получения сменного задания по ID
- [ ] Эндпойнт изменения сменного задания по ID
- [ ] Эндпойнт получения списка сменных заданий по фильтрам
- [ ] Эндпойнт "аггрегации" продукции

**Задачи "со звездочкой":**

- [ ] Тесты
- [ ] docker
- [ ] Базовый CI/CD

**Запуск проекта:**

Опишите, как запустить ваш проект и как запустить тесты (если они есть).

Решение нужно отправить в виде ссылки на ваш репозиторий с проектом (не забудьте сделать его публичным).

Пожалуйста, в ходе решения этой задачи постарайтесь комитить ваши действия (а не просто отправить готовое решение одним коммитом), нам важно оценить не только ваше решение, но и как вы к нему пришли.

Нам будет особенно приятно, если вы будете использовать [conventional commits](https://www.conventionalcommits.org/en/v1.0.0/#summary).

Если в ходе реализации возникнут какие-то вопросы, смело задавайте их в [issues](https://github.com/axon-expert/backend-test-task/issues) данного репозитория, мы на них обязательно ответим.

## Основные задачи

### Эндпойнт добавления сменных заданий

Принимает список сменных заданий в виде json.

Сменное задание состоит из следующих полей с следующими типами:

**СтатусЗакрытия**: bool

**ПредставлениеЗаданияНаСмену**: str

**Рабочий центр**: str

**Смена**: str

**Бригада**: str

**НомерПартии**: int

**ДатаПартии**: date

**Номенклатура**: str

**КодЕКН**: str

**ИдентификаторРЦ**: str

**ДатаВремяНачалаСмены**: datetime

**ДатаВремяОкончанияСмены**: datetime

Пример:
```json
[
	{
		"СтатусЗакрытия": false,
		"ПредставлениеЗаданияНаСмену": "Задание на смену 2345",
		"РабочийЦентр": "Т2",
		"Смена": "1",
		"Бригада": "Бригада №4 ХПС",
		"НомерПартии": 22222,
		"ДатаПартии": "2024-01-30",
		"Номенклатура": "Какая то номенклатура",
		"КодЕКН": "456678",
		"ИдентификаторРЦ": "A",
		"ДатаВремяНачалаСмены": "2024-01-30T20:00:00+05:00",
		"ДатаВремяОкончанияСмены": "2024-01-31T08:00:00+05:00"
	}
]
```

Да, поля на русском, потому что эти данные приходят из 1С заказчика) Чтобы корректно с этим работать, советуем использовать [validation_alias](https://docs.pydantic.dev/latest/concepts/fields/#field-aliases) в ваших pydantic моделях. Как эти поля назвать в ваших pydantic моделях, остается на ваше усмотрение.

Каким образом хранить это в БД остается так же на ваше усмотрение, кроме одного момента: у сменного задания по-мимо поля ``СтатусЗакрытия`` должно быть поле ``closed_at`` (время закрытия), которое выставляется при закрытии партии (если она еще открыта, то это поле имеет значение ``null``). Так же у сменного задания должен быть какой-то внутренний id (в качестве ``primary key``), по которому можно было бы получить информацию о конкретном сменном задании, либо же изменить ее (например, закрыть сменное задание).

Важно: пара НомерПартии и ДатаПартии всегда уникальна! Если уже существует какая-то партия с аналогичным номером партии и датой партии, мы должны ее перезаписать.

### Эндпойнт получения сменного задания (партии) по ID (``primary key``).
Данный эндпойнт должен вернуть json с сменным заданием по его внутреннему ID вместе с списком продукции, привязанной к этой партии (о привязке продукции к партии далее).

Если сменного задания с данным ID нет, то вернуть 404 ошибку.

В респонсе нет нужды использовать поля на русском языке, этот эндпойнт используем мы, а не заказчик. (то есть в респонсе будут поля с такими же названиями, какие вы им дали)

### Эндпойнт изменения сменного задания (партии) по ID (``primary key``).
Данный эндпойнт должен позволять изменить одно или несколько полей. Если сменного задания с данным ID нет, то вернуть 404 ошибку.

На вход должен быть json с полями и их новыми значениями (без ``closed_at`` и ``id``). Если какого-то поля в json'е нет, то нужно оставить то значение, которое было изначально.
Если статус закрытия партии меняется на ``True``, то в ``closed_at`` необходимо выставить текущий ``datetime``, а если наоборот -- то ``null``.

Советуем для этого использовать специальную pydantic модель с ``Optional`` полями (у которых дефолтное значение ``None``). В данной модели нет нужды использовать ``validation_alias``, наименования полей будет такое, какое вы сами написали.

В качестве респонса необходимо вернуть json обновленной партии.

### Эндпойнт получения сменных заданий по различным фильтрам 

Данный эндпойнт должен возвращать json со списком сменных заданий по различным фильтрам (СтатусЗакрытия, НомерПартии, ДатаПартии и так далее, рассчитываем на ваш полет фантазии).

Тут уже нет нужды использовать поля на русском языке, с этими данными работаем мы.

Желательно реализовать механизм пагинации (с помощью ``offset`` и ``limit``).

### Эндпойнт добавления продукции для сменного задания (партии).

Данный эндпойнт получает из 1С заказчика список продукции (УникальныйКод и ЕКН) и сменных заданий (НомерПартии и ДатаПартии), к которому данная продукция **привязана**.

Пример:
```json
[
	{
		"УникальныйКод": "12gRV60MMsn1",
		"ЕКН": "456678",
		"НомерПартии": 22222,
		"ДатаПартии": "2024-01-30"
	},
	{
		"УникальныйКод": "12gRV60MMsn2",
		"ЕКН": "456678",
		"НомерПартии": 33333,
		"ДатаПартии": "2024-01-31"
	}
]
```

Необходимо сохранить продукцию с привязкой к партии. Дополнительно к продукции добавить поля ``is_aggregated`` (bool поле, была ли данная продукция уже использована) и ``aggregated_at`` (datetime, когда была использована данная продукция). Если продукция передана с несуществующей партией (нет сменного задания с указаным номером партии и датой партии), то данную продукцию можно игнорировать.

Уникальный код должен быть уникальным в принципе (а не только в рамках какой-то партии). Если переданная продукция с данным уникальным кодом уже существует, то ее можно игнорировать.

### Эндпойнт "аггрегации" продукции.
Данный эндпойнт принимает ID партии (``primary key``) и уникальный код продукции. Если продукция с данным уникальным кодом существует и привязана к партии с данным ID, и при этом данная продукция не использована, то необходимо изменить ``is_aggregated`` на true и ``aggregated_at`` на текущий ``datetime`` и вернуть данную продукцию в виде json'a.

Если данная продукция уже использована, то вернуть 400 ошибку с описанием: "Данная продукция уже была использована в {aggregated_at}".

Если продукция существует, но привязана к другой партии, необходиму вернуть 400 ошибку с описанием: "Данная продукция принадлежит другой партии".

Если продукции с данным уникальным кодом не существует, то необходимо вернуть 404 ошибку.

Формат входных данных выбираете вы сами.

## Задания "со звездочкой"

### Добавить тесты (unit, functional, integration)

### Использовать контейнеризацию (docker)

### Базовый CI/CD
Используя ``Github actions``:
1. Добавить проверку кода линтерами (mypy, flake8) и форматтерами (black, isort). Вместо flake8, isort и black можно использовать ruff.
2. Добавить запуск тестов (если есть).
3. Добавить сборку проекта в docker образ и отправку этого образа в docker hub или github container registry.
