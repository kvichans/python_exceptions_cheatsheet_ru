#Шпаргалка про исключения в Питоне**
**План**

1. Метафора "аварийный выход" (как в кинотеатре).
2. Термины и схема применения
3. Допустимые форматы для бросков
   - Запрет на броски базовых исключений
   - Запрет на броски без описания проблемы
4. Жонглирование исключениями
   - *Последний рубеж* (где перехватывать всех)
   - *Игнорирование* (не знаешь чем дополнить, не перехватывай)
   - *Уточнения* (если можешь дополнить, создай новое)
   - *Исключение не про ошибку*
5. Когда/как создавать новые классы исключений?
6. Логгирование исключений
7. *Гард-блоки* и броски
8. Передача управления в пределах функции

##Метафора "аварийный выход" (как в кинотеатре).**
У зрителей в кинотеатре есть 
    штатный выход (там где входили) 
и еще 
    аварийный (часто на другой этаж или даже на улицу).
Так и у функций есть 
    штатный возврат через `return` 
и еще 
    аварийный через `raise`.

Удобно представлять себе, что `raise` это специализированный аналог `return`. 
Внутри функции у них одинаковые смыслы - закончить выполнение и вернуть какие-то данные. 
А вот для окружающих они различаются: 

 - От `return` данные приходят в "присвоение" `переменная = функция(параметры)`.
 - А от `raise` данные хранит Питон.
Можно их либо игнорировать, либо анализировать после "перехвата" дополнительным кодом.

##Термины

- **Бросок**/throw/подъем/`raise`.
Общепринятые и всем понятные термины "бросок", "бросать исключение".
В Питоне у команды `raise` смысл подъем/поднимать. Можно говорить и так. 
Только при переходе в другой язык придется переучиваться.
- **Контролируемый участок** кода.
В Питоне такой участок выделяется блоком `try:`.
- **Перехват**/catch/перехватывать/**ловить**.
В Питоне это `except [Тип [as объект]]:`
- **Реакция на перехват**.
Это содержимое блока `except`.
Изредка, реакция даже бывает пустой (`pass` или `...`), если это нужно алгоритму.

##Схема применения

```python
if проверка:
    raise ТипБроска(параметры) # (1) (2)
try:
    #<контролируемый код>      # (3)
except:                        # (4)
except ТипПерехвата:           # (5)
except Тип as имя:             # (6)
    #<код с реакцией на перехват>
    raise ТипБроска(параметры) # (7)
except (Тип1, Тип2):           # (8)
```

*Замечания:*

- (1) Запрещены броски типов **не наследников** от `Exception`.
- (2) Параметры для исключения не обязательны. 
Но, почти всегда, это один строковый параметр с **описанием проблемы**.
- (3) Какие исключения прилетают от "контроллируемого кода"?
Это нужно выяснять по докам к каждой применяемой детали.
- В Питоне допускается вольности:
   - (4) Можно "**ловить все**", если писать `except:код`.
   - (5) Можно "**ловить тип**, но не объект", если писать `except Тип:код`.
   - (6) Обычно ловят тип с объектом `except Тип as объект:код`.
- (5)(6)(8) Перехват "*с типом*" ловит только исключения, 
которые являются **наследниками** от указанного(ых).
Например, `except Exception` ловит все исключения. 
- (4)(5)(6)(8) Блоков `except` может быть несколько - **они конкурируют**!
Исключение достанется первому, у которого подходящие ожидания.
Поэтому нужно располагать самые *узкоожидающие* сверху, 
а самые *охватывающие* (например, `except:`) снизу.
- (7) Броски в реакциях уместны. 
Тип для броска не обязан совпадать с типом для перехвата.
- (8) Если реакции совпадают, то можно (*нужно!*) объединить перехваты.

##Допустимые форматы для бросков

1. **Запрет** на броски строковых значений (они же не наследники `Exception`).
2. **Запрет** на броски исключений базового типа `Exception`.
Это лишает вызывающую сторону гибкого поведения. 
Она будет не в состоянии отличить такой бросок от других сбоев.
Нужно 
   - либо подобрать штатный тип (это лучше), 
   - либо создать своего наследника (это допустимо).
3. **Запрет** на броски без описания проблемы.
Один только тип не дает конкретики.

##Жонглирование исключениями

1. **Последний рубеж** – место для перехвата всех исключений.
Если ваш код содержит головную функцию с бесконечным циклом, 
то в ней обязательно нужен перехват всех долетевших исключений. 
Поймать все, залоггировать каждое – дать циклу работать дальше.

2. **Игнорирование**.
Это самое контр-интуитивное правило.
Если к пролетающему исключению добавить нечего, то его обязательно **пропустить мимо**.
Если не знаете, как правильно реагировать, обязаны **пропустить**.
Частое нарушение этого правила - избыточный перехват (например, по `Exception`).
Следствие из этого правила: *контролируемый код лучше делать минимальным*.

3. **Уточнения**.

- Если есть что добавить к описанию проблемы, 
то после перехвата бросается **новое исключение** с дополненным описанием.
- Если тип пролетающего исключение из либы, известной только внутри функции/модуля, 
а бросок будет перехватываться там, где про эту либу не знают, **замена типа** обязательна!
*Пример*. Извлечение страницы по урлу выполняется спец-либой (например, `requests`), 
импортированной в текущий модуль. Тогда пропускать мимо сбой нельзя. 
Нужно поймать и поменять тип на независимый от этой либы.

4. **Исключение не про ошибку**.
Во многих случаях "перехват" нужен для обнаружения ошибки, но при этом исключение ее не описывает.
*Пример*. Параметр функции используется как ключ к внутреннему словарю - может возникнуть `KeyError`. 
Но проблема не в промахе ключа, а в значении параметра.
Поэтому нужно не только сформировать новое описание для исключения, но и задать ему новый тип.

##Когда/как создавать новые классы исключений?
Если среди штатных типов исключений есть *идеально подходящий к ситуации*, лучше применять его.
Учтите, что штатные типы вызывающей стороне более удобны - их не нужно дополнительно импортировать.
Если штатного нет, создать свой, наследуя его от *одного из штатных*.

Тут есть "*подводные камни*".

   - *Что такое "идеально подходящий к ситуации"?*
Нужно рассмотреть проблему с максимального удаления, 
то есть выделить в ней самое абстрактное ядро.
Если для такого "ядра" есть штатный тип, то его и нужно применять. 
А все детали ситуации, которые вне абстрактного ядра, 
описать текстом при создании исключения.
   - *От какого типа наследовать?*
Опять нужно выделить в ситуации "абстрактное ядро".
И опять наследовать от типа, подходящего к такому ядру.
Обычно применяется наследование от `Exception` (подходит для всего).

При создании своего класса-исключения важно знать про "ошибку новичка": 
    *Добавлять конструктор вредно!*
Должен работать конструктор от базового класса - он гибкий, 
так как готов принимать разнообразные параметры.
Если свой конструктор действительно нужен, 
то в нем обязательно должен быть вызов "от базового" через 
    `super().__init__(параметры)`.

##Логгирование исключений
Недопустимо логгировать "голое исключение".
Всегда нужно вставлять его текст в описание ситуации.

##Гард-блоки и броски
Так как бросок для функции это полный аналог возврата, 
броски прекрасно сочетаются с гард-блоками, 
то есть с организацией кода, при которой делаются проверки для раннего возврата.

```python
if есть-недостаток:
    raise ТипИсключения('<описание проблемы>')
if есть-недостаток:
    return что-то
if есть-недостаток:
    raise ТипИсключения('<описание проблемы>')
```

##Передача управления в пределах функции
Кроме работы аналогом `return` броски могут передавать управление внутри функции, 
то есть работать аналогами `break` с дополнительными данными.