# eeprom_emul
EEPROM emulator for STM32F100 and 20's 32 bit variables in 3 page of N25Q032A SPI serial flash memory

## Задание

Нужно разработать модуль на языке C. 
Предложить решение и реализовать основную логику модуля
 
Задача: имеется 20 переменных размером 32 бита, которые должны быть доступны для быстрого доступа при работе устройства. Разные переменные изменяются с разной частотой, но среднее количество изменений составляет 3 раза в мин.
Данные должны восстанавливаться после перезагрузки и потеря данных является критической. При разработке необходимо исходить из того, что питание прибора может пропадать в любой момент.
В качестве энергонезависимой памяти используется флешка, например, N25Q032A с размером сектора 4кбайта и количеством стираний 10 тыс.
Под настройки отводится 3 сектора (размером 4кбайта)
Необходимо разработать концепцию модуля хранения и реализовать логику модуля. 
Для работы с флеш памятью предоставлен модуль, который обеспечивает запись на флеш. Интерфейс модуля представлен в файле во вложении.
 
Модуль будет выполняться на микроконтроллере, например, STM32, хотя желательно, чтобы он мог выполняться и на PIC, AVR произвольной разрядности.
Операционная система не используется.
Функции записи в буфер могут вызываться из различных прерываний и фоновых потоков программы. 
Запись должна выполняться максимально быстро.
Место в ОЗУ должно быть использовано максимально эффективно.

весь проект доступен тут
https://github.com/Mirn/eeprom_emul

## Лицензия
т.к. мы ещё не подписали трудовой договор, то код и реализация принадлежат мне Ситникову Евгению Николаевичу.
его можно редактировать, компилировать и запускать только для для ознакомления с моим решением задания.
использовать в коммерческих разработках разрешается только, если я буду принят на работу.

## Инструменты и платформа
* проект сделан для STM32F100C8
* на базе makefile и IDE Eclipse Kepler
* для gcc 4.5+
* диалект языка С99 
* без ос 

## Состав проекта:
* **flash_module.c** - реализация планировщика задач по работе с флеш памятью
* **flash_vars.c** - модуль работы с 20 переменными, которые сохраняются в флеш памяти (само задание)
* **mem.c** - низкоуровневый драйвер работы работы с флеш памятью для N25Q032A по SPI (пины можно настроить в hw.h)
* **логи и протоколирование работы** - в каждом из этих трёх модулей есть два вида логов: инициализации и работы, можно отключать или включать независимо
* **main.c** - тестовый пример как работать с библиотеками в целом
* **остальной проект** - общая сборка воедино, подготовка окружения, настройка периферии и тактирования от внутреннего генератора

## Анализ задания
"имеется 20 переменных размером 32 бита"
т.к. явно не указано, то предположил что, это эти переменные беззнаковые 32 битные целые
так же предположил, что каждая переменная не связана смыслом с остальными и независима, 
например накопленное потребление 3х фазного многоканального счётчика эл. энергии.

Для реализации "Функции записи вызываться из различных потоков программы. Запись должна выполняться максимально быстро." 
интерфейс доступа к этим переменным сделан в виде volatile массива, доступный из любого потока, и 
любой поток может сам изменять, как ему вздумается, их значения,
и разграничением многопоточного доступа моя библиотека не занимается.

Также имеется теневая копия всех переменных - она необходима для того, чтобы понять, какая переменная изменилась, и её необходимо сохранить немедленно.

"Место в ОЗУ должно быть использовано максимально эффективно.". 
Данный пункт конфликтует с отказоустойчивостью и скоростью доступа. 
Но постарался минимизировать потребление озу. 
Часть переменных ужал до 8 или 16 бит, например, номер текущей страницы.
Массив "pages_stat" можно разместить в каком-либо другом массиве, но это не стал делать, т.к. 
это костыль, который имеет смысл делать в последнюю очередь (я бы такой треш вообще ни за что бы не стал бы делать).

"Для работы с флеш памятью предоставлен модуль, который обеспечивает запись на флеш. Интерфейс модуля представлен"
насколько я понял, этот модуль решает две проблемы:
1. позволяет основной программе что-то делать пока флеш занята записью или стиранием страницы (а это несколько секунд)
2. единый интерфейс доступа к флеш памяти - удобно, если нескольким алгоритмам необходимо обращаться к одному ресурсу.

## Реализация
Основное требование - сохранение переменных при любых событиях.

Переменные сохраняются независимо в отдельных записях.

Структура записи одной переменной:

1. Сама переменная 32 бита = 4 байта.
2. С каждой переменной сохраняется номер переменной. 8 бит = 1 байт.
3. Дополнительно сохраняется номер переменной проинвертированный.  8 бит = 1 байт. Это нужно, чтобы можно было различить не записанную запись от записанной, даже если переменная, её адрес и crc все биты равны единицам. Это гарантирует, что в записи будут нулевые биты.
4. КРК16 код 16 бит = 2 байта - нужен, чтобы отличить мусор и повторную перезапись в случае сбоев. А так-же не до конца стёртых страниц - в них бывает мусор.

Итого: 8 байт на запись. 

**состояние "state_write_check"**
Флеш память можно обнулять без стирания, но выставить единицы можно только стиранием - я дописываю записи последовательно в одну единственную страницу.
каждая запись во всех состояниях завершается чтением записанного и верификацией

**состояние "state_newpage"**
Как только текущая страница заполнится, то библиотека делает следующее:

1. стирает следующую по циклу страницу
2. в следующую страницу записывает записи со всем массивом переменных
3. стирает текущую страницу
4. текущую страницу делает следующей.

### Первичная инициализация:
**Состояние "state_erase_all"** при первом включении все страницы стираются.

### Инициализация с восстановлением состояния:
1. **_Состояние "state_read_stat"**
Для последующей инициализации библиотека считывает записи во всех страницах, отведённых под переменные, и собирает статистику, количество целых, битых и пустых записей.

2. **Анализ в функции "stat_process"**
Страницы со сбойными записями игнорируются - они либо мусор, либо не до конца стёрты
Страницы без записей тоже.
Восстановление состояния производится из одной страницы, либо из частично заполненной, если там кол-во записей больше или равно кол-ву переменных, иначе из заполненой.
Либо из страницы полностью заполненной последней по порядку следования, включая ситуацию перехода из конца очереди в начало. 

3. **Состояние "state_read_restore"**
восстановление содержимого из одной выбранной страницы на шаге 2 

4. в случае ошибок происходит переход в state_erase_all и обнуление всех переменных.

## Порядок инициализации
Для инициализации нужно время, и нужно знать результат (успешен или нет)
для этого, соответственно, добавлена пара функций 
`bool flash_vars_read_init_done(void); //запрос готовности`
`bool flash_vars_read_init_error(void); //запрос флага ошибки`

также во время инициализации необходимо вызывать цикле `flash_module_proc` и `flash_vars_proc`
до тех пор, пока `flash_vars_read_init_done` не вернёт истину, до этого момента все 20 переменных недоступны и не должны изменяться извне

сам модуль `flash_vars.с` функцию `flash_module_proc`  не вызывает, дабы не плодить неявные вызовы, странные зависимости и не раскидывать точки вызывов, где только можно.
т.к. с модулем флеш памяти наверняка будут работать другие модули.

## Настройки
- Количество переменных можно задавать от 1 до 255
- Количество страниц от 3 до 255
- Можно задать размер страницы, но не менее, чем кол-во переменных, умноженное 8
- Адреса и номера секций флеш памяти можно задать любые, но не меньше, чем размер страницы, порядок не важен.

## Как проверял
Нашёл у себя изделие с данной флеш памятью; более того, там были битые байты и слова.
Проверял на этом работающем изделии, десятки тысяч циклов (написал скрипт, включающий и выключающий рандомно питание).
При этом питание флеш памяти и мк реально коммутировалось ключом на полевом транзисторе. 
И я видел, что иногда страницы не до конца стирались.
Также проверил, что в случае деградировавших слов мой алгоритм верификации их детектирует, обходит и сохраняет далее. 
Проверил, что работает с полноценным восстановлением после сброса питания даже в случае такой частично неработающей памяти.
Проверил, что восстанавливается, если мусор в флеше или битые данные имеются, и их игнорирует. 

В основной функции main два потока:
один в основном цикле и меняет две переменных: одну очень быстро, другую очень медленно.
другой поток - на прерывании системного таймера, изменяются остальные переменные.
Все переменные увеличиваются на +1, это удобно, чтобы отследить изменения и их количество между сбросами питания и тд.


## Потребление ресурсов:
при -O2, по секциям:
* .text + .rodata:  ~2100 байт
* .data: 0 байт
* .bss: ~250 байт
* .stack: ~100 байт
