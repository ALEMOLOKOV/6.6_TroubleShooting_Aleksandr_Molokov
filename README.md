# 6.6_TroubleShooting_Aleksandr_Molokov

## Задача 1

Перед выполнением задания ознакомьтесь с документацией по [администрированию MongoDB](https://docs.mongodb.com/manual/administration/).

Пользователь (разработчик) написал в канал поддержки, что у него уже 3 минуты происходит CRUD операция в MongoDB и её 
нужно прервать. 

Вы как инженер поддержки решили произвести данную операцию:
- напишите список операций, которые вы будете производить для остановки запроса пользователя
- предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

## ОТВЕТ

Сначала необходимо определить проблемные запросы - db.currentOP()

Например для поиска запросов которые длятся больше 3 минут

db.currentOp(
   {
     "active" : true,
     "$ownOps": true,
     "secs_running" : { "$gt" : 180 }
   }
)

Для остановки запроса можно использовать db.killOP(), при использовании этой команды необходимо задуматься о последствиях, при неправильном применении можно нарушить консистентность данных если остановить update или incert данных.
Использование систем мониторинга может помочь определить большинство проблем.

Чтобы не было долгих запросов можно постриить / перестроить индексы

Можно еще использовать метод maxtimeMS - метод устанавливает ограничение по времени выполнения команды

Пример:

db.runCommand( { distinct: "collection",
                 key: "city",
                 maxTimeMS: 45 } )
                 


## Задача 2

Перед выполнением задания познакомьтесь с документацией по [Redis latency troobleshooting](https://redis.io/topics/latency).

Вы запустили инстанс Redis для использования совместно с сервисом, который использует механизм TTL. 
Причем отношение количества записанных key-value значений к количеству истёкших значений есть величина постоянная и
увеличивается пропорционально количеству реплик сервиса. 

При масштабировании сервиса до N реплик вы увидели, что:
- сначала рост отношения записанных значений к истекшим
- Redis блокирует операции записи

Как вы думаете, в чем может быть проблема?

## ОТВЕТ

Проблема может быть в нехватке памяти из-за истекших ключей, Redis блокирует операции записи в момент очистки истекших ключей.
Из манула получается, что при активном методе удаления Redis запускает очистку истекших ключей 10 раз в секунду, но если если в базе данных много-много ключей, срок действия которых истекает в одну и ту же секунду, и они составляют не менее 25% от текущего набора ключей с истекшим сроком действия, Redis может заблокировать, чтобы процент ключей, срок действия которых уже истек, был ниже 25%, и в этот момент может происходить блокировка операций. Задержка происходить только когда очень много клчей истекают в одно и тое время.


 
## Задача 3

Вы подняли базу данных MySQL для использования в гис-системе. При росте количества записей, в таблицах базы,
пользователи начали жаловаться на ошибки вида:
```python
InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
```

Как вы думаете, почему это начало происходить и как локализовать проблему?

Какие пути решения данной проблемы вы можете предложить?

## ОТВЕТ

Для определения проблемы необходимо проанализировать логи, чтобы понять какие запросы приводят к ошибке и понять объем и характер этих запросов. Возможно запросы объемные и данные не успевают передаться за отведённое время или клиент не успевает установить соединение.

Пути решения

Увеличить значение параметров : net_read_timeout, max_allowed_packet, interactive_timeout, wait_timeout, max_connections

net_read_timeout — время в секундах, в течение которого сервер будет ожидать получения данных, прежде чем соединение будет прервано. Если сервер не обслуживает клиентов с очень медленными или нестабильными каналами, то 15 секунд здесь будет достаточно.

interactive_timeout — время в секундах, в течение которого сервер ожидает активности со стороны интерактивного соединения (использующего флаг CLIENT_INTERACTIVE), прежде чем закрыть его.

wait_timeout — время в секундах, в течение которого сервер ожидает активности соединения, прежде чем прервет его. В общем случае 30 секунд будет достаточно.

max_allowed_packet — максимальный размер данных, которые могут быть переданы за один запрос. Следует увеличить, если столкнетесь с ошибкой «Packet too large».

max_connections — максимальное количество параллельных соединений к серверу. Увеличьте его, если сталкиваетесь с проблемой «Too many connections».


## Задача 4


Вы решили перевести гис-систему из задачи 3 на PostgreSQL, так как прочитали в документации, что эта СУБД работает с 
большим объемом данных лучше, чем MySQL.

После запуска пользователи начали жаловаться, что СУБД время от времени становится недоступной. В dmesg вы видите, что:

`postmaster invoked oom-killer`

Как вы думаете, что происходит?

Как бы вы решили данную проблему?

## ОТВЕТ

Самое лучшее решение - сначала проанализировать данные мониторинга, далее поработать с конфигурацией, 
а затем уже можно добавить ресурсов и детально разобраться с проблемой

Можно изменить  параметры, которые регулируют память в Postgres из основных это: max_connections, shared_buffer, work_mem, effective_cache_size, maintenance_work_mem.

shared_buffer - этот параметр устанавливает, сколько выделенной памяти будет использоваться PostgreSQL для кеширования.

wal_buffers - PostgreSQL сначала записывает записи в WAL (журнал пред записи) в буферы, а затем эти буферы сбрасываются на диск. Размер буфера по умолчанию, определенный wal_buffers, составляет 16 МБ. Но если у нас много одновременных подключений, то более высокое значение может повысить производительность.

effective_cache_size - предоставляет оценку памяти, доступной для кэширования диска. Это всего лишь ориентир, а не точный объем выделенной памяти или кеша. Он не выделяет фактическую память, но сообщает оптимизатору объем кеша, доступный в ядре. Если значение этого параметра установлено слишком низким, планировщик запросов может принять решение не использовать некоторые индексы, даже если они будут полезны. Поэтому установка большого значения всегда имеет смысл.

work_mem - если нам нужно выполнить сложную сортировку, увеличьте значение work_mem для получения хороших результатов. Сортировка в памяти происходит намного быстрее, чем сортировка данных на диске. Установка очень высокого значения может стать причиной узкого места в памяти для нашей среды, поскольку этот параметр относится к операции сортировки пользователя.

maintenance_work_mem - это параметр памяти, используемый для задач обслуживания.
