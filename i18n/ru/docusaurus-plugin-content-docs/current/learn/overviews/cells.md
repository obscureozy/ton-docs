# Клетки как хранилище данных

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

Все данные в TON хранятся в ячейках. Ячейка - это структура данных, содержащая:

- до **1023 бит** данных (не байтов!)
- до **4 ссылок** на другие ячейки

Биты и ссылки не смешиваются (они хранятся отдельно). Циклические ссылки запрещены: для любой ячейки ни одна из ее потомков не может иметь эту исходную ячейку в качестве ссылки.

Таким образом, все клетки представляют собой направленный ациклический граф (DAG). Вот хороший рисунок для иллюстрации:

![Directed Acylic Graph](/img/docs/dag.png)

## Типы клеток

В настоящее время существует 5 типов клеток: *обычные* и 4 *экзотических*.
К экзотическим типам относятся следующие:

- Обрезанная клетка ветви
- Справочная ячейка библиотеки
- Ячейка с доказательством Меркла
- Ячейка обновления Меркле

:::tip
Подробнее об экзотических клетках см: [**TVM Whitepaper, раздел 3**] (https://ton.org/tvm.pdf).
:::

## Вкусовые качества клеток

Ячейка - это непрозрачный объект, оптимизированный для компактного хранения.

В частности, она дедуплицирует данные: если в разных ветвях есть несколько эквивалентных подячеек, на которые ссылаются разные ветви, их содержимое хранится только один раз. Однако непрозрачность означает, что ячейка не может быть изменена или прочитана напрямую. Таким образом, существует 2 дополнительных вида ячеек:

- *Строитель* для частично построенных ячеек, для которых могут быть определены быстрые операции по добавлению битовых строк, целых чисел, других ячеек и ссылок на другие ячейки.
- *Slice* для "разрезанных" ячеек, представляющих собой либо остаток частично разобранной ячейки, либо значение (подячейку), находящееся внутри такой ячейки и извлеченное из нее с помощью инструкции синтаксического анализа.

В TVM используется еще один особый клеточный вкус:

- *Продолжение* для ячеек, содержащих оп-коды (инструкции) для виртуальной машины TON, смотрите [обзор TVM с высоты птичьего полета](/learn/tvm-instructions/tvm-overview).

## Сериализация данных в ячейки

Любой объект в TON (сообщение, очередь сообщений, блок, состояние всего блокчейна, код и данные контракта) сериализуется в ячейку.

Процесс сериализации описывается схемой TL-B: формальное описание того, как данный объект может быть сериализован в *Builder* или как разобрать объект заданного типа из *Slice*.
TL-B для ячеек - это то же самое, что TL или ProtoBuf для байтовых потоков.

Если Вы хотите узнать больше подробностей о (де)сериализации ячеек, Вы можете прочитать статью [Cell & Bag of Cells](/develop/data-formats/cell-boc).

## См. также

- [Язык TL-B](/develop/data-formats/tl-b-language)
