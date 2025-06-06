# Инициализация TVM

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

:::info
To maximize your comprehension of this page, familiarizing yourself with the [TL-B language](/v3/documentation/data-formats/tlb/cell-boc) is highly recommended.

- [TVM Retracer](https://retracer.ton.org/)
  :::

TVM вызывается во время фазы вычислений обычных и/или других транзакций.

## Начальное состояние

Новый экземпляр TVM инициализируется до выполнения смарт-контракта следующим образом:

- Оригинальный **cc** (текущая продолжительность) инициализируется с помощью ячейки среза, созданного из раздела `code` смарт-контракта. В случае замороженного или неинициализированного состояния аккаунта, код должен быть указан в поле `init` входящего сообщения.

- **cp** (текущая кодовая страница TVM) устанавливается на значение по умолчанию, которое равно 0. Если смарт-контракт хочет использовать другую кодовую страницу TVM *x*, то он должен переключиться на нее, используя `SETCODEPAGE` *x* в качестве первой инструкции своего кода.

- Значения **газа** (*лимиты газа*) инициализируются в соответствии с результатами фазы кредитования.

- Вычисление **библиотек** (*контекст библиотеки*) [описано ниже](#library-context).

- Процесс инициализации **стека** зависит от события, которое вызвало транзакцию, и его содержимое [описано ниже](#stack).

- Контрольный регистр **c0** (возвращаемая продолжительность) инициализируется экстраординарной продолжительностью `ec_quit` с параметром 0. При выполнении это продолжительность ведет к завершению TVM с кодом выхода 0.

- Контрольный регистр **c1** (альтернативная возвращаемая продолжительность) инициализируется экстраординарной продолжительностью `ec_quit` с параметром 1. При вызове она приводит к завершению TVM с кодом выхода 1. Обратите внимание, что оба кода выхода 0 и 1 считаются успешным завершением TVM.

- Контрольный регистр **c2** (обработчик исключений) инициализируется экстраординарной продолжительностью `ec_quit_exc`. При вызове она извлекает верхнее целое число из стека (равное номеру исключения) и завершает TVM с кодом выхода, равным этому целому числу. Таким образом, по умолчанию все исключения приводят к завершению выполнения смарт-контракта с кодом выхода, равным номеру исключения.

- Контрольный регистр **c3** (словарь кода) инициализируется ячейкой с кодом смарт-контракта, например **cc** (текущая продолжительность), описанным выше.

- Контрольный регистр **c4** (корень постоянных данных) инициализируется постоянными данными смарт-контракта, хранящимися в его разделе `data`. В случае замороженного или неинициализированного состояния аккаунта, данные должны быть предоставлены в поле `init` входящего сообщения. Обратите внимание, что постоянные данные смарт-контракта не обязательно должны быть загружены полностью, чтобы это произошло. Вместо этого загружается корень, и TVM может загружать другие ячейки по их ссылкам из корня только при доступе к ним, тем самым предоставляя форму виртуальной памяти.

- Контрольный регистр **c5** (корень действий) инициализируется пустой ячейкой. Примитивы "выходных действий" TVM, такие как `SENDMSG`, накапливают *выходные действия* (например, исходящие сообщения) в этом регистре, которые должны быть выполнены после успешного завершения смарт-контракта. Схема TL-B для ее сериализации [описана ниже](#control-register-c5)

- Контрольный регистр **c7** (корень временных данных) инициализируется как кортеж, а его структура [описана ниже](#control-register-c7)

## Контекст библиотеки

Контекст библиотеки_ (среда библиотеки) смарт-контракта — это хэш-карта, отображающая 256-битные хэши ячеек (представлений) в соответствующие ячейки. Когда во время выполнения смарт-контракта осуществляется доступ к ссылке на внешнюю ячейку, в библиотечной среде выполняется поиск ячейки, на которую ссылаются, и ссылка на внешнюю ячейку прозрачно заменяется найденной ячейкой.

Среда библиотеки для вызова смарт-контракта вычисляется следующим образом:

1. Глобальная среда библиотеки для текущего воркчейна берется из текущего состояния мастерчейна.
2. Затем она дополняется локальной средой библиотеки смарт-контракта, хранящейся в поле `library` состояния смарт-контракта. Учитываются только 256-битные ключи, равные хэшам соответствующих значений ячеек. Если ключ присутствует как в глобальной, так и в локальной среде библиотеки, локальная среда имеет приоритет при слиянии.
3. Наконец, она дополняется полем `library` поля `init` входящего сообщения (если таковое имеется). Обратите внимание, что если аккаунт заморожен или не инициализирован, поле `library` сообщения будет использоваться вместо локальной среды библиотеки из предыдущего шага. Библиотека сообщений имеет более низкий приоритет, чем локальная и глобальная библиотечные среды.

Наиболее распространенный способ создания общих библиотек для TVM — это публикация ссылки на корневую ячейку библиотеки в мастерчейне.

## Стек

Инициализация стека TVM происходит после формирования начального состояния TVM и зависит от события, вызвавшего транзакцию:

- внутреннее сообщение
- внешнее сообщение
- тик-так
- подготовка к разделению
- установка слияния

Последний элемент, помещенный в стек, всегда является *селектором функции*, который является *целым числом*, идентифицирующим событие, вызвавшее транзакцию.

### Внутреннее сообщение

В случае внутреннего сообщения стек инициализируется путем помещения аргументов в функцию `main()` смарт-контракта следующим образом:

- Баланс *b* смарт-контракта (после зачисления значения входящего сообщения) передается как *целое* количество nanoton.
- Баланс *b*<sub>m</sub> входящего сообщения *m* передается как *целое* количество nanoton.
- Входящее сообщение *m* передается как ячейка, которая содержит сериализованное значение типа *Message X*, где *X* - тип тела сообщения.
- Тело *m*<sub>b</sub> входящего сообщения, равное значению поля body *m* и переданное как срез ячейки.
- Селектор функции *s*, обычно равный 0.

После этого выполняется код смарт-контракта, равный его начальному значению **c3**. Он выбирает правильную функцию в соответствии с *s*, которая, как ожидается, обработает оставшиеся аргументы функции и затем завершается.

### Внешнее сообщение

Входящее внешнее сообщение обрабатывается аналогично [внутреннему сообщению, описанному выше](#internal-message), со следующими изменениями:

- Селектор функции *s* устанавливается в -1.
- Баланс *b*<sub>m</sub> входящего сообщения всегда равен 0.
- Начальный текущий лимит газа *g*<sub>l</sub> всегда равен 0. Однако начальный кредит газа *g*<sub>c</sub> > 0.

Смарт-контракт должен завершиться при *g*<sub>c</sub> = 0 или *g*<sub>r</sub> ≥ *g*<sub>c</sub>; в противном случае транзакция и содержащий ее блок недействительны. Валидаторы или коллаторы, предлагающие кандидата на блок, никогда не должны включать транзакции, обрабатывающие входящие внешние сообщения, которые являются недействительными.

### Тик-так

В случае транзакций тик-так стек инициализируется путем передачи аргументов в функцию `main()` смарт-контракта следующим образом:

- Баланс *b* текущего аккаунта передается как *целое* количество nanoton.
- 256-битный адрес текущего аккаунта внутри мастерчейна как беззнаковое *целое* число.
- Целое число, равное 0 для тик-транзакций и -1 для ток-транзакций.
- Селектор функции *s*, равный -2.

### Подготовка разделения

В случае транзакции подготовки разделения стек инициализируется путем передачи аргументов в функцию `main()` смарт-контракта следующим образом:

- Баланс *b* текущего аккаунта передается как *целое* количество nanoton.
- *Срез*, содержащий *SplitMergeInfo*.
- 256-битный адрес текущего аккаунта.
- 256-битный адрес дочернего аккаунта.
- Целое число 0 ≤ *d* ≤ 63, равное положению единственного бита, в котором адреса текущего и дочернего аккаунтов различаются.
- Селектор функции *s*, равный -3.

### Установка слияния

В случае транзакции установки слияния стек инициализируется путем передачи аргументов в функцию `main()` смарт-контракта следующим образом:

- Баланс *b* текущего аккаунта (уже объединенный с балансом nanoton дочернего аккаунта) передается как *целое* число nanoton.
- Баланс *b'* дочернего аккаунта, взятый из входящего сообщения *m*, передается как *целое* число nanoton.
- Сообщение *m* из дочернего аккаунта, автоматически сгенерированное транзакцией подготовки слияния. Его поле `init` содержит конечное состояние дочернего аккаунта. Сообщение передается как ячейка, которая содержит сериализованное значение типа *Message X*, где *X* — тип тела сообщения.
- Состояние дочернего аккаунта, представленное параметром *StateInit*.
- *Срез*, содержащий *SplitMergeInfo*.
- 256-битный адрес текущего аккаунта.
- 256-битный адрес дочернего аккаунта.
- Целое число 0 ≤ *d* ≤ 63, равное положению единственного бита, в котором адреса текущего и дочернего аккаунтов различаются.
- Селектор функций *s*, равный -4.

## Контрольный регистр c5

*Выходные действия* смарт-контракта накапливаются в ячейке, хранящейся в контрольном регистре **c5**: сама ячейка содержит последнее действие в списке и ссылку на предыдущее, таким образом образуя связанный список.

Список также может быть сериализован как значение типа *OutList n*, где *n* — длина списка:

```tlb
out_list_empty$_ = OutList 0;

out_list$_ {n:#}
  prev:^(OutList n)
  action:OutAction
  = OutList (n + 1);

out_list_node$_
  prev:^Cell
  action:OutAction = OutListNode;
```

Список возможных действий при этом состоит из:

- `action_send_msg` — для отправки исходящего сообщения
- `action_set_code` — для установки кода операции
- `action_reserve_currency` — для хранения коллекции валют
- `action_change_library` — для изменения библиотеки

Как описано в соответствующей схеме TL-B:

```tlb
action_send_msg#0ec3c86d
  mode:(## 8) 
  out_msg:^(MessageRelaxed Any) = OutAction;
  
action_set_code#ad4de08e
  new_code:^Cell = OutAction;
  
action_reserve_currency#36e6b809
  mode:(## 8)
  currency:CurrencyCollection = OutAction;

libref_hash$0
  lib_hash:bits256 = LibRef;
libref_ref$1
  library:^Cell = LibRef;
action_change_library#26fa1dd4
  mode:(## 7) { mode <= 2 }
  libref:LibRef = OutAction;
```

## Контрольный регистр c7

Контрольный регистр **c7** содержит корень временных данных в виде кортежа, сформированного типом *SmartContractInfo*, содержащим некоторые базовые данные контекста блокчейна, такие как время, глобальная конфигурация и т. д. Он описывается следующей схемой TL-B:

```tlb
smc_info#076ef1ea
  actions:uint16 msgs_sent:uint16
  unixtime:uint32 block_lt:uint64 trans_lt:uint64 
  rand_seed:bits256 balance_remaining:CurrencyCollection
  myself:MsgAddressInt global_config:(Maybe Cell) = SmartContractInfo;
```

Первым компонентом этого кортежа является значение *Integer*, которое всегда равно 0x076ef1ea, после чего следуют 9 именованных полей:

| Поле                | Тип                                                                                 | Описание                                                                                                                                                                                                                  |
| ------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `actions`           | uint16                                                                              | Изначально инициализируется 0, но увеличивается на единицу всякий раз, когда выходное действие устанавливается примитивом выходного действия не-RAW                                                                       |
| `msgs_sent`         | uint16                                                                              | Количество отправленных сообщений                                                                                                                                                                                         |
| `unixtime`          | uint32                                                                              | Временная метка Unix в секундах                                                                                                                                                                                           |
| `block_lt`          | uint64                                                                              | Представляет *логическое время* предыдущего блока этого аккаунта. [Подробнее о логическом времени](/v3/documentation/smart-contracts/message-management/messages-and-transactions#what-is-a-logical-time) |
| `trans_lt`          | uint64                                                                              | Представляет *логическое время* предыдущей транзакции этого аккаунта                                                                                                                                                      |
| `rand_seed`         | bits256                                                                             | Инициализируется детерминированно, начиная с `rand_seed` блока, адреса аккаунта, хеша входящего обрабатываемого сообщения (если есть) и логического времени транзакции `trans_lt`                      |
| `balance_remaining` | [CurrencyCollection](/v3/documentation/data-formats/tlb/msg-tlb#currencycollection) | Оставшийся баланс смарт-контракта                                                                                                                                                                                         |
| `myself`            | [MsgAddressInt](/v3/documentation/data-formats/tlb/msg-tlb#msgaddressint-tl-b)      | Адрес этого смарт-контракта                                                                                                                                                                                               |
| `global_config`     | (Возможно, ячейка)                                               | Содержит информацию о глобальной конфигурации                                                                                                                                                                             |

Обратите внимание, что в предстоящем обновлении TVM кортеж **c7** был расширен с 10 до 14 элементов. Подробнее об этом читайте [здесь](/v3/documentation/tvm/changelog/tvm-upgrade-2023-07).

## См. также

- Оригинальное описание инициализации TVM из технического документа
