# Form UI checklist (МВХ testing checklist)

The mandatory UI features every WinForms form is tested against before release. Source: the testing team's
checklist «Список тестирования — МВХ» («Обязательный интерфейсный функционал»), 2026-09-07. Apply it to every new or
changed form, and go through it before calling a form finished. The wording below follows the original document
(Russian); implementation notes are in *italics*.

How-to for the grid items: [SmartGridConventions.md](SmartGridConventions.md) (footers, «Информация о записи»,
`C1DataCollectionBindingList`), [RecordChangeLog.md](RecordChangeLog.md) (what the record log needs on the DB side).

## 1. Кнопки (стандартный функционал)

| # | Требование | Обязательно |
|---|---|---|
| 1.1 | «Новый» — добавление новой строки в таблице | да |
| 1.2 | «Дублировать» — дублирует активную строку в таблицу, с изменением наименования или других полей | при необходимости |
| 1.3 | «Редактировать» | да |
| 1.4 | «Удалить» — удаляет активную строку из таблицы | да |
| 1.5 | «Обновить» | да |
| 1.6 | У каждой кнопки при наведении курсора всплывает подсказка (tooltip) | да |

## 2. Окно формы

| # | Требование |
|---|---|
| 2.1 | Заголовок окна описывает объект, с которым выполняются работы в этом окне |
| 2.2 | Вверху окна (при необходимости) — «Зона фильтрации»: параметры для фильтрации информации по указанным значениям |
| 2.3 | Разумный размер формы в сжатом и развёрнутом состоянии |
| 2.4 | Высота формы не больше 600 px, чтобы форма целиком помещалась на мониторе 1366×768. *The source says «минимальная высота … 600 px», but the stated goal (fit on 1366×768) means the form must fit within 600 px.* |

## 3. Редактирование объектов

| # | Требование |
|---|---|
| 3.1 | Деньги и количества: формат с разделителем разрядов и 2 знаками после запятой, выравнивание вправо. Символы — влево |
| 3.2 | Оптимизировать навигацию по Tab (`TabIndex` в логичном порядке) |

## 4. Таблицы (грид)

| # | Требование | Примечание |
|---|---|---|
| 4.1 | По правой кнопке мыши — контекстное меню с работающим пунктом «Информация о записи» (лог-журнал активной строки) | *Needs the logging procedure on the DB side, see RecordChangeLog.md* |
| 4.2 | Итоговая строка с количеством строк («Всего: 234») | *The built-in `count` footer counts **non-empty cells**, not rows: if the column can be empty for a row, count rows yourself and draw the total in `OwnerDrawCell`, or anchor the aggregate on a never-empty column (№). SmartGridConventions: "a `Count` footer counts only non-empty cells"* |
| 4.3 | Денежные и количественные данные: разделитель разрядов, 2 знака после запятой, выравнивание вправо. Нулевое значение — пустая ячейка (не нули) | *Format `N2`; blank zeros in `OwnerDrawCell` — SmartGridConventions: "hiding zero values in a SmartGrid column"* |
| 4.4 | Текстовые значения — выравнивание влево | |
| 4.5 | Символы, аббревиатуры и заголовки колонок — по центру. Заголовки жирные | |
| 4.6 | Клик по заголовку колонки сортирует таблицу по ней | *Default behaviour with `C1DataCollectionBindingList`* |
| 4.7 | Все колонки, кроме колонок фиксированной ширины (даты, аббревиатуры, символы и т. п.), можно менять по ширине | |
| 4.8 | Колонку даты сужать ровно под формат `dd.mm.yyyy`, если заголовок это позволяет | |
| 4.9 | Ширина колонок — под ожидаемое содержимое; слишком длинный заголовок сократить в разумных пределах | |
