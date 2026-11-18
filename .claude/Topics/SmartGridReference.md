# SmartGrid — расширенная сетка данных

SmartGrid — это расширенная версия компонента C1FlexGrid, которая включает в себя ряд полезных функций и улучшений. Компонент предназначен для более удобной работы с табличными данными и включает дополнительные возможности, такие как многоуровневые заголовки, выделение строк клавишами, расширенное контекстное меню, тултипы, изменение размера шрифта с помощью колеса мыши, управление сортировкой и построение иерархических структур.

## Основные функциональные возможности

- Многоуровневые заголовки с автоматическим объединением ячеек
- Выделение строк клавишами в стиле Bookkeep
- Расширенное контекстное меню с дополнительными функциями
- Тултипы для отображения содержимого ячеек
- Динамическое изменение размера шрифта с помощью колеса мыши
- Пользовательская сортировка с возможностью возврата к исходному порядку
- Форматирование футера согласно формату колонки
- Построение иерархических структур (деревьев)


## Многоуровневые заголовки

SmartGrid предоставляет удобный механизм для создания и редактирования многоуровневых заголовков таблицы с автоматическим объединением ячеек.

### Свойства

```csharp
[Category("Micros")]
[Description("Helps to add merged multirow headers")]
[Editor(typeof(HeadersUIEditor), typeof(UITypeEditor))]
public string[] Headers { get; set; }
```

### Использование

Свойство Headers решает проблему гибкого формирования многострочных заголовков таблицы, когда требуется объединять соседние ячейки по горизонтали и вертикали в разных вариациях.

- Перед редактированием Headers следует выделить необходимое количество фиксированных строк таблицы, отводимых под заголовки: категория Rows → Fixed.
- При нажатии на значке редактирования свойства Headers — […] — открывается окно редактора с таблицей с указанным количеством строк, предназначенных для формирования заголовков.
- Объединение по вертикали и горизонтали происходит автоматически при заполнении соседних ячеек одинаковыми значениями.
- Кнопка Merge preview/Unmerge позволяет включить/выключить предпросмотр результата слияния ячеек непосредственно в редакторе.
- Закончив заполнение, нужно подтвердить ввод нажатием кнопки OK.
- Кнопка Cancel отменяет операцию.
- Если ширина столбцов редактируемой таблицы после установки заголовков была изменена в штатном дизайнере FlexGrid (C1FlexGrid Column Editor), эти изменения сохранятся и в редакторе заголовков.
- Также в штатном дизайнере можно свободно добавлять, удалять и перемещать столбцы — эти изменения также будут отражены в редакторе заголовков.
### Пример использования

```csharp
// Получение списка выделенных строк
List<Row> selectedRows = smartGrid.GetSelectedRows();

// Снятие выделения со всех строк
smartGrid.UnselectAll();
```

## Контекстное меню

SmartGrid предоставляет расширенное контекстное меню с дополнительными функциями для работы с таблицами.

### Стандартные пункты меню

- "Копировать ячейку" — копирует содержимое выделенной ячейки в буфер обмена
- "Суммировать" — суммирует значения в выбранной колонке
- "Экспорт в Excel" — экспортирует всю таблицу в Excel-файл
- "Экспорт выделенного в Excel" — экспортирует только выделенные строки в Excel-файл
- "Скрыть выделенную колонку" — скрывает выбранную колонку
- "Показать скрытые колонки" — отображает ранее скрытые колонки

### События и делегаты

```csharp
public Predicate<SmartGrid> XLExportPredicate;
```

### Методы для работы с контекстным меню

```csharp
public void CreateContextMenu()
public void AddItemToContextMenu(string name, Bitmap image, ClickEventHandler handler)
public void AddSeparator()
public void RemoveDefaultContextMenuItems(bool removeCopyCell = false, bool removeSumCol = false, 
                                         bool removeExportToExcel = false, bool removeHideColumn = false)
```

### Пример использования

```csharp
// Добавление своего пункта меню
smartGrid.AddItemToContextMenu("Мой пункт меню", MyBitmap, MyMenuItemHandler);

// Добавление разделителя
smartGrid.AddSeparator();

// Удаление стандартных пунктов меню
smartGrid.RemoveDefaultContextMenuItems(removeCopyCell: true, removeSumCol: true);

// Установка предиката для валидации перед экспортом в Excel
smartGrid.XLExportPredicate = (grid) => {
    // Проверка перед экспортом
    bool canExport = /* ваша логика проверки */;
    return canExport;
};
```

## Тултипы для ячеек

SmartGrid предоставляет возможность отображения тултипов для ячеек таблицы.

### Свойства

```csharp
public C1SuperTooltip ToolTip { get; set; }
```

### Настройка тултипов

Для корректной работы тултипов необходимо в конструкторе формы привязать свойство ToolTip грида к компоненту C1SuperTooltip, размещенному на форме.

```csharp
// В конструкторе формы
public MyForm()
{
    InitializeComponent();
    
    // Создаем компонент C1SuperTooltip
    C1SuperTooltip tooltip = new C1SuperTooltip();
    
    // Привязываем его к SmartGrid
    smartGrid.ToolTip = tooltip;
}
```

## Ресайз шрифта колесом мыши

SmartGrid позволяет изменять размер шрифта в ячейках с помощью колеса мыши при зажатой клавише Ctrl.

### Параметры

- **Начальный размер шрифта:** 10 пунктов
- **Минимальный размер шрифта:** 8 пунктов
- **Максимальный размер шрифта:** 20 пунктов

### Использование

Для изменения размера шрифта:

1. Наведите курсор на таблицу
2. Зажмите клавишу Ctrl
3. Прокручивайте колесо мыши вверх для увеличения размера шрифта или вниз для уменьшения

## Установка сортировки по умолчанию

SmartGrid добавляет возможность восстановления исходной сортировки данных после сортировки по возрастанию и убыванию.

### Делегаты

```csharp
public event Action SetDefaultSorting;
```

### Настройка

```csharp
// В конструкторе формы
public MyForm()
{
    InitializeComponent();
    
    // Добавляем обработчик для восстановления исходной сортировки
    smartGrid.SetDefaultSorting += RefreshData;
}

// Метод для загрузки данных в исходном порядке
private void RefreshData()
{
    // Код для загрузки данных в исходном порядке
}
```

После этой настройки порядок сортировки при кликах по заголовкам столбцов будет следующим:

1. По возрастанию
2. По убыванию
3. По умолчанию (исходный порядок)

## Форматирование футера

SmartGrid автоматически форматирует значения в футере в соответствии с форматом колонки.
Без необходимости дополнительной настройки, значения агрегации в футере (суммы, средние значения и т.д.) будут отображаться с тем же форматированием, что и данные в соответствующих колонках.

### Поддерживаемые форматы и типы данных

- Числовые значения (double, decimal, int)
- Различные агрегаты (Sum, Average, Percent, Min, Max)


## Построение иерархии (дерева)

SmartGrid поддерживает построение иерархических структур (деревьев) с возможностью перетаскивания узлов.

### Основные свойства и делегаты

```csharp
public string IdName { get; set; }
public SortingType SortingType { get; set; }
public IComparer Comparer { get; set; }
public bool AllowNodeMove { get; set; }
public Row CurrRow => Rows[Row];
public List<int> SelectedIds => SetSelectedIds(SelectedRows);
public List<int> SelectedNodeIds => SetSelectedNodeIds();

// События
public event SetCurrentValuesDelegate SetCurrentValues;
public event NodeMoveDelegate BeforeNodeMove;
public event NodeMoveDelegate AfterNodeMove;
public event ValidateDragTargetDelegate ValidateDragTarget;
```

### Методы для работы с деревом

```csharp
// Построение дерева
public void BuildTree<T>(IEnumerable<T> source) where T : ITreeData, new()

// Перемещение узлов
public bool MoveNode(Node currentNode, Node parentNode)
public bool IsSubroot(Node rootNode, Node Node)

// Управление узлами
public void ExpandBranch(Node node)
public void CollapseByLevel(int startLevel = 0)
public void ExpandByLevel(int startLevel = 0)
public void SetSelectedRow(int id)

// Добавление узлов
public void AddNodeToCurrentNode(DataRow row)
public void AddNode(Node ParentNode, DataRow row)
public void UpdateCurrentNode(DataRow row)
public void UpdateNode(Node node, DataRow row)

// Стилизация
public void SetImages(string typeField, Dictionary<string, Image> dict)
public void SetImagesToConcreteColumn(string typeField, Dictionary<string, Image> dict)
public void SetImages(Image img)

// Сортировка
public void SortNodes()
public void SortNodes(RowsComparer rowComparer)
public void SortNodes(string field, SortingType sortingType)

// Работа с данными
public T GetCurrentObj<T>() where T : class, ITreeData
public T GetObjOfParentCurrentNode<T>() where T : class, ITreeData
public List<T> GetObjFromCurrentNode<T>() where T : class, ITreeData
public void AddNodeToCurrentNode<T>(T value) where T : class, ITreeData
public void SetValueToCurrentNode<T>(T value) where T : class, ITreeData
public List<int> GetSelectedNodeIds<T>() where T : class, ITreeData
```

### Пример использования

```csharp
// Создание модели для дерева
public class TreeItem : ITreeData
{
    public int Id { get; set; }
    public int ParentId { get; set; }
    public string Name { get; set; }
    // Другие свойства...
}

// Получение данных
List<TreeItem> treeData = GetTreeData();

// Настройка дерева
smartGrid.IdName = "Id";
smartGrid.AllowNodeMove = true;

// Добавление обработчиков событий
smartGrid.BeforeNodeMove += (Node currentNode, Node parentNode, ref bool allowMove) => {
    // Проверка возможности перемещения
    allowMove = CheckIfMoveAllowed(currentNode, parentNode);
};

smartGrid.AfterNodeMove += (Node currentNode, Node parentNode, ref bool allowMove) => {
    // Обновление базы данных после перемещения
    UpdateDatabase(currentNode, parentNode);
};

// Построение дерева
smartGrid.BuildTree(treeData);

// Сортировка узлов
smartGrid.SortNodes("Name", SortingType.Ascending);

// Установка изображений для узлов
Dictionary<string, Image> images = new Dictionary<string, Image>();
images.Add("Folder", Properties.Resources.Folder);
images.Add("Document", Properties.Resources.Document);
smartGrid.SetImages("Type", images);

// Получение выбранного объекта
TreeItem selectedItem = smartGrid.GetCurrentObj<TreeItem>();
```

## Вспомогательные свойства и методы

```csharp
public bool IsEditing { get; set; }
```

Свойство IsEditing предназначено для коррекции поведения клавиши Escape для SmartGrid на форме, унаследованной от TemplateFormMDI.