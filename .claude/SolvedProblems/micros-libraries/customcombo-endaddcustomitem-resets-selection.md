# `Refers.CustomCombo.EndAddCustomItem` сбрасывает выбор и заменяет весь список

*Найдено: 2026-07-30, проект CWATIS (`ContractsForm`, комбобокс шаблонов документов `Templ_DocCB`).*

## Symptom

Runtime. При сохранении контракта в режиме редактирования MySQL валит запрос по внешнему ключу
(`Cannot add or update a child row: a foreign key constraint fails`) на `cwatis.contracts.TemplDoc_Id` —
но **только** если у контракта шаблон уже был выбран. Ошибка появляется даже когда пользователь
шаблон вообще не трогал: правил номер, комментарий, что угодно ещё.

Внешне выглядит как ошибка процедуры сохранения. На самом деле в `UniSaver` уезжает
`"TemplDoc_Id": 0` — строки с id 0 в `templdocs` нет.

## Root cause

Декомпиляция `Refers.CustomCombo` (наследник `C1.Win.C1List.C1Combo`):

```csharp
public void EndAddCustomItem(List<SourseList> l = null, string displMember = "ID", bool IsSort = true)
{
    if (l != null) { base.DataMode = DataModeEnum.Normal; sArr = l; }   // ← список ПОЛНОСТЬЮ заменяется
    if (IsSort) sArr = sArr.OrderBy(s => s.SortCol).ToList();
    SetDataSource(sArr, displMember, "Name");
}

public void SetDataSource(object source, string vMember, string dMember, int linesCount = 0)
{
    base.DataSource = null;      // ← пересвязывание: SelectedValue/SelectedIndex обнуляются
    base.DataSource = source;
    EndFillCombo(vMember, dMember);
}
```

Из этого следуют два неочевидных факта:

1. **Каждый вызов `EndAddCustomItem` теряет текущий выбор** (`DataSource = null` перед присваиванием).
   Значение, выставленное ранее через `SelectedValue`, после перезаполнения не восстанавливается само.
2. **`l` заменяет список, а не дополняет его.** Популярный паттерн «сначала привязать пустой элемент,
   потом дозаполнить» ломается — второй вызов выкидывает пустой элемент:

```csharp
// НЕПРАВИЛЬНО: после второго EndAddCustomItem пустого элемента в списке нет,
// а выбор сброшен в null
ComboBoxExtentions.sArr = null;
cb.AddCustomItemS(0, "", true);
cb.EndAddCustomItem(ComboBoxExtentions.sArr);   // список = [пустой]
ComboBoxExtentions.sArr = null;
var types = await LoadAsync(...);               // ← ещё и await посреди работы со static sArr
foreach (var t in types) cb.AddCustomItemS(t.Id, t.Name, true, $"0{i}");
cb.EndAddCustomItem(ComboBoxExtentions.sArr);   // список = [типы], пустой элемент потерян
```

Дальше `Numeric.Int(cb.SelectedValue)` даёт `0`, дифф «старое 5 → новое 0» уезжает в БД, FK падает.

Третья мина в том же паттерне: `ComboBoxExtentions.sArr` — **статическое** поле, общее для всех
комбобоксов. Если метод заполнения объявлен `async void` и делает `await` между `AddCustomItemS`
и `EndAddCustomItem`, параллельное заполнение другого комбобокса затрёт накопленный список.

## Fix

Одна привязка на одно заполнение + явное восстановление выбора; метод — `async Task`, а не `async void`:

```csharp
private async Task RefreshTemplDocCBAsync(int sdId, int selectedTemplDocId)
{
    var typesTask = ListTemplDocTypesAsync?.Invoke(sdId);
    var types = ((typesTask == null ? null : await typesTask) ?? Enumerable.Empty<CntrType>()).ToList();

    ComboBoxExtentions.sArr = null;
    Templ_DocCB.AddCustomItemS(NoTemplDocId, "", true);              // пустой элемент — в том же списке
    for (int i = 0; i < types.Count; i++)
        Templ_DocCB.AddCustomItemS(types[i].Id, types[i].Name, true, $"0{i + 1}");
    Templ_DocCB.EndAddCustomItem(ComboBoxExtentions.sArr);           // ровно одна привязка
    ComboBoxExtentions.sArr = null;

    Templ_DocCB.SelectedValue = types.Any(t => t.Id == selectedTemplDocId)
        ? selectedTemplDocId
        : NoTemplDocId;                                              // выбор восстанавливаем вручную
}
```

Плюс страховка на записи: `0` в nullable FK-колонке — это «не выбрано», в БД должен уйти `NULL`.
`MC_GlobalForms.HelperClasses.JsonHelper.BuildJson` сериализует и `DBNull.Value`, и `null`
в JSON `null` (проверено вызовом через рефлексию), так что `UniSaver` получит корректный `NULL`:

```csharp
string column = GetDbFieldName(diff.PropertyName);
fields[column] = column == "TemplDoc_Id" && Numeric.Int(diff.NewValue) == 0
    ? (object)DBNull.Value
    : diff.NewValue;
```

Прецедент: `CWATIS/Forms/Contracts/View/ContractsForm.cs` — `RefreshTemplDocCBAsync`, `BtnOK_Click`.

## Побочный симптом: после сохранения комбобокс пустой

Тот же корень даёт второе проявление. Если список комбобокса зависит от другого поля записи
(здесь — от подразделения), то «сбросить комбобокс после выхода из редактирования» —
неверное действие: в списке для «нулевого» фильтра нужного элемента нет, и отображать
сохранённое значение нечем.

```csharp
// НЕПРАВИЛЬНО: OnRefreshRightAsync выставил правильный шаблон,
// а следующая строка тут же перезатёрла его пустым списком
await OnRefreshRightAsync();
await RefreshTemplDocCBAsync(0, 0);
```

Правильно — заполнять такой комбобокс в том же методе, который синхронизирует остальные поля
с текущей записью (`ContractDataRefresh`), из полей самой записи, а «сбросы» после OK/Cancel убрать:

```csharp
await RefreshTemplDocCBAsync(contract.SdId ?? 0, contract.TemplDocId);
```

Вызов ставить **в конец** метода: `await` посреди присваивания контролов разрывает обновление
панели и пускает в неё сообщения UI-потока.

Если процедура списка тяжёлая (`cwatis.TemplDocsGet` возвращает ещё и содержимое .docx),
кэшируй список по значению фильтра — иначе один запрос на каждое перемещение по строкам мастера.
Перед входом в редактирование кэш обходи (`forceReload`), чтобы пользователь выбирал из актуального списка.

## Checklist

- Перезаполнил `CustomCombo` — **сразу же** верни `SelectedValue`. Само не восстановится.
- Всё содержимое комбобокса (включая пустой элемент «не выбрано») собирай в **один** список и
  привязывай **одним** `EndAddCustomItem`. Второй вызов = новый список, а не догрузка.
- Метод заполнения — `async Task` с `await` у вызывающего. `async void` + статический
  `ComboBoxExtentions.sArr` = гонка между комбобоксами.
- Если комбобокс перезаполняется динамически (в отличие от «залили один раз при инициализации»),
  считай потерю выбора нормой и проектируй восстановление сразу.
- Диффовое сохранение (`GetChangedProperties` → `UniSaver`) усиливает эту ошибку: потерянный выбор
  превращается в реальный `UPDATE` колонки. Для nullable FK маппь `0` → `DBNull.Value`.
- `cb.SelectedValue = x` бросает `ArgumentException: DataField or Caption not found`, если у контрола
  ещё нет колонок данных (не создан handle / нет привязки). Выставляй выбор только после
  `EndAddCustomItem` и только когда контрол уже на форме.
