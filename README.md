# Задание №1

## Задача 1.1

Разработать ETL-процесс для загрузки «банковских» данных из csv-файлов в соответствующие таблицы СУБД PostgreSQL. Покрыть данный процесс логированием этапов работы и всевозможной дополнительной статистикой (на ваше усмотрение). Обратите внимание, что в разных файлах может быть разный формат даты, это необходимо учитывать при загрузке.

### Требования к реализации задачи:

- В своей БД создать пользователя / схему «DS».
- Создать в DS-схеме таблицы под загрузку данных из csv-файлов.
- Начало и окончание работы процесса загрузки данных должно логироваться в специальную логовую таблицу. Эту таблицу нужно придумать самостоятельно. По логам должно быть видно дату и время старта и окончания загрузки, так же можете туда добавить любую дополнительную информацию, которую посчитаете нужным.
- После логирования о начале загрузки добавить таймер (паузу) на 5 секунд, чтобы чётко видеть разницу во времени между началом и окончанием загрузки. Из-за небольшого учебного объёма данных – процесс загрузки быстрый.
- Для хранения логов нужно в БД создать отдельного пользователя / схему «LOGS» и создать в этой схеме таблицу для логов.
- Для корректного обновления данных в таблицах детального слоя DS нужно выбрать правильную Update strategy и использовать следующие первичные ключи для таблиц фактов, измерений и справочников (должно быть однозначное уникальное значение, идентифицирующее каждую запись таблицы).

### Решение:

1. Подготовить SQL-скрипты для создания схем и таблиц.
2. Реализовать логику по загрузке данных и логированию в таблицу с логами.
3. Реализовать методы для корректного добавления данных в БД и их обновления.

### Запуск проекта

Из корневой папки выполняем команду:

```bash
docker-compose up
```

Ссылка на видо с демонстрацией <a href="https://disk.yandex.ru/i/CGkeQp4l8Gql2A">тут</a>


## Задача 1.2


После успешного наполнения детального слоя «DS» исходными данными из файлов, необходимо рассчитать витрины данных в слое «DM»:
- Витрина оборотов (`DM.DM_ACCOUNT_TURNOVER_F`)
- Витрина остатков (`DM.DM_ACCOUNT_BALANCE_F`)

## Витрина оборотов по лицевым счетам

Витрина оборотов содержит информацию по оборотам по лицевым счетам в рамках дня, когда были обороты. Необходимо создать процедуру расчета `ds.fill_account_turnover_f`, которая будет иметь один входной параметр – дату расчета (`i_OnDate`).

### Поля витрины оборотов:
- **on_date**: Дата, за которую производим расчет (`i_OnDate`).
- **account_rk**: Идентификатор счета, по которому были проводки в дату расчета.
- **credit_amount**: Сумма проводок (поле `DS.FT_POSTING_F.credit_amount`) за дату расчета (`DS.FT_POSTING_F.oper_date = i_OnDate`), где счет участвовал как счет по кредиту (`DS.FT_POSTING_F.credit_account_rk`).
- **credit_amount_rub**: `credit_amount`, умноженный на курс, действующий за эту дату (курс хранится в поле `ds.md_exchange_rate_d.reduced_cource`). Если информация о курсе отсутствует, умножаем на единицу.
- **debet_amount**: Аналогично `credit_amount`, но для проводок, где счет участвовал как счет по дебету (`DS.FT_POSTING_F.debet_account_rk`), и берем сумму из поля `DS.FT_POSTING_F.debet_amount`.
- **debet_amount_rub**: `debet_amount`, умноженный на курс, действующий за эту дату. Если информация о курсе отсутствует, умножаем на единицу.

Если по счету не было проводок в дату расчета, он не должен попадать в витрину в эту дату.

После создания процедуры необходимо рассчитать витрину за каждый день января 2018 года.

---

## Витрина остатков по лицевым счетам

### Инициализация остатков на 31.12.2017
Остатки за день считаются на основе остатков за предыдущий день. Необходимо заполнить витрину `DM.DM_ACCOUNT_BALANCE_F` за 31.12.2017 данными из `DS.FT_BALANCE_F`:
- **on_date**, **account_rk**, **balance_out**: Заполняются один в один.
- **balance_out_rub**: `balance_out`, умноженный на курс, действующий за 31.12.2017. Если информация о курсе отсутствует, умножаем на единицу.

### Процедура расчета витрины остатков
Создайте процедуру `ds.fill_account_balance_f` с одним входным параметром – датой расчета (`i_OnDate`).

#### Алгоритм заполнения:
1. Возьмите все счета, действующие за дату расчета (дата расчета лежит между датами актуальности записей в таблице `DS.MD_ACCOUNT_D`).
2. Для каждого счета рассчитайте `balance_out`:
   - **Для активных счетов** (`DS.MD_ACCOUNT_D.char_type = 'А'`):
     - Берем остаток в валюте счета за предыдущий день (если его нет, считаем его равным 0).
     - Прибавляем обороты по дебету в валюте счета (`DM.DM_ACCOUNT_TURNOVER_F.debet_amount`).
     - Вычитаем обороты по кредиту в валюте счета (`DM.DM_ACCOUNT_TURNOVER_F.credit_amount`) за этот день.
   - **Для пассивных счетов** (`DS.MD_ACCOUNT_D.char_type = 'П'`):
     - Берем остаток в валюте счета за предыдущий день (если его нет, считаем его равным 0).
     - Вычитаем обороты по дебету в валюте счета.
     - Прибавляем обороты по кредиту в валюте счета за этот день.
3. Поле **balance_out_rub** заполняем аналогично `balance_out`, но для расчета берем поля в рублях.

Обратите внимание, что в какие-то дни по счету может не быть оборотов, но остаток по счету мы должны заполнить.

После создания процедуры рассчитайте витрину остатков за каждый день января 2018 года.


Ссылка на видо с демонстрацией <a href="https://disk.yandex.ru/i/mKhAs0bre0oVJw">тут</a>


## Задача 1.3

101 форма содержит информацию об остатках и оборотах за отчетный период, сгруппированных по балансовым счетам второго порядка. Вам необходимо создать процедуру расчета (назовите ее `dm.fill_f101_round_f`), которая должна иметь один входной параметр – отчетную дату (`i_OnDate`). Отчетная дата – это первый день месяца, следующего за отчетным. То есть, если мы хотим рассчитать отчет за январь 2018 года, то должны передать в процедуру 1 февраля 2018 года. В отчет должна попасть информация по всем счетам, действующим в отчетном периоде, группировка в отчете идет по балансовым счетам второго порядка (балансовый счет второго порядка – это первые 5 символов номера счета (`DS.MD_ACCOUNT_D.account_number`). 

### Поля витрины должны заполняться следующим образом:

- **FROM_DATE** – первый день отчетного периода.
- **TO_DATE** – последний день отчетного периода.
- **CHAPTER** – глава из справочника балансовых счетов (`DS.MD_LEDGER_ACCOUNT_S`).
- **LEDGER_ACCOUNT** – балансовый счет второго порядка.
- **CHARACTERISTIC** – характеристика счета (можно получить из поля `DS.MD_ACCOUNT_D.char_type`).
- **BALANCE_IN_RUB** – сумма остатков в рублях (`DM.DM_ACCOUNT_BALANCE_F.balance_out_rub`) за день, предшествующий первому дню отчетного периода (если отчет собирается за январь 2018 года, то это 31 декабря 2017 года), для рублевых счетов (рублевые счета, это те, у которых код валюты (поле `DS.MD_ACCOUNT_D.currency_code` равно 810 или 643)).
- **BALANCE_IN_VAL** – сумма остатков в рублях за день, предшествующий первому дню отчетного периода для всех счетов, кроме рублевых.
- **BALANCE_IN_TOTAL** – сумма остатков в рублях за день, предшествующий первому дню отчетного периода для всех счетов.
- **TURN_DEB_RUB** – сумма дебетовых оборотов в рублях (`DM.DM_ACCOUNT_TURNOVER_F.debet_amount_rub`) за все дни отчетного периода для рублевых счетов.
- **TURN_DEB_VAL** – сумма дебетовых оборотов в рублях за все дни отчетного периода для всех счетов, кроме рублевых.
- **TURN_DEB_TOTAL** – сумма дебетовых оборотов в рублях за все дни отчетного периода для всех счетов.
- **TURN_CRE_RUB** – сумма кредитовых оборотов в рублях (`DM.DM_ACCOUNT_TURNOVER_F.credit_amount_rub`) за все дни отчетного периода для рублевых счетов.
- **TURN_CRE_VAL** – сумма кредитовых оборотов в рублях за все дни отчетного периода для всех счетов, кроме рублевых.
- **TURN_CRE_TOTAL** – сумма кредитовых оборотов в рублях за все дни отчетного периода для всех счетов.
- **BALANCE_OUT_RUB** – сумма остатков в рублях (`DM.DM_ACCOUNT_BALANCE_F.balance_out_rub`) за последний день отчетного периода для рублевых счетов.
- **BALANCE_OUT_VAL** – сумма остатков в рублях за последний день отчетного периода для всех счетов, кроме рублевых.
- **BALANCE_OUT_TOTAL** – сумма остатков в рублях за последний день отчетного периода для всех счетов.

### Решение:

1. Создать процедуру `dm.fill_f101_round_f` с входным параметром `i_OnDate`.
2. Реализовать логику расчета всех полей витрины, описанных выше.
3. Убедиться, что данные корректно группируются по балансовым счетам второго порядка.
4. Протестировать процедуру на данных за январь 2018 года.

### Пример вызова процедуры:

```sql
CALL dm.fill_f101_round_f('2018-02-01');
```

Ссылка на видо с демонстрацией <a href="https://disk.yandex.ru/i/_7e5rOrFaiXeAw">тут</a>



## Задача 1.4

Теперь необходимо выгрузить эти данные в формате, который бы позволил легко обмениваться ими между отделами. Один из таких форматов – CSV.

Напишите скрипт, который позволит выгрузить данные из витрины `dm.dm_f101_round_f` в CSV-файл, первой строкой должны быть наименования колонок таблицы.

Убедитесь, что данные выгружаются в корректном формате и напишите скрипт, который позволит импортировать их обратно в БД. Поменяйте пару значений в выгруженном CSV-файле и загрузите его в копию таблицы 101-формы `dm.dm_f101_round_f_v2`.


Ссылка на видо с демонстрацией <a href="https://disk.yandex.ru/d/0i42EVLVfKRSxQ">тут</a>


