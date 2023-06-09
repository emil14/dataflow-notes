# Что такое Dataflow

Это когда _данные управляют выполнением_. Такие программы реагируют, обновляясь, на данные, когда они приходят.

## Pipeline Dataflow

Простейшая реализация Dataflow.

## Узлы (nodes)

Вычислительный элемент со вводом и выводом, осуществляющий какую-то операцию. Единственный способ узлу получить и отправить данные это через "порты".

Pipeline dataflow позволяет один входной и один выходной порты. Когда у узлов несколько портов, они чаще всего имеют имена.

Узлы чаще всего _идемпотентны_ (чистые), но это не не обязательно. Исполнение кода узлом называется _активацией_ или "firing".

## Данные

Удивительно для методологии со словом "Data" в имени, dataflow не взаимодействует со значениями данных напрямую. Данные в dataflow представлены в виде "токенов" (представление данных независимо от их структуры).

Изменяемость (mutability) данных играет важную роль, т.к. аффектит параллелизм, однако, она напрямую не запрещена в dataflow. Pipeline dataflow обычно использует неизменяемость данных.

## Дуги (arcs)

Путь, который проделывает _токен_ от одного узла к другому называется _дугой_. Для упрощения будем думать, что данные движутся всегда от выходных портов к входным. Другие имена для дуги это _ребро, провод, соединение_ или _ссылка_.

О дуге можно думать как о трубе, которая один или более токенов. Обязательное условие для активации узла - как минимум один токен ждёт на входе и выходная дуга свободна. Вычитывание данных со входного порта освобождает дугу для ввода новых данных. _Вместимость дуги_ - кол-во токенов, которые она может вмещать в один момент времени.

Соединение и разделение дуг запрещено в pipeline dataflow, однако, разрешено в других ветвях dataflow и будет рассмотренно позже.

## Dataflow графы

Направленный граф содержащий узлы (вершины) и дуги (рёбра) между ними. Все соединения явно объявлены программистом. Граф также называется dataflow программой.

## Выполнение графа

В pipeline dataflow (и большинстве остальных dataflow направлений) узлы активированы только тогда, когда на входе есть данные. _Data-driven execution_ - появление данных вызывает активность узлов и их отсутствие, позволяет им уходить в _режим ожидания_ (idle). Узлы могут работать одновременно при условии постоянного потока данных.

Узел, получив данные на входе, не обязан отправлять данные на выход. Логика может быть какой угодно. От этого зависит активация _последующего_ (downstream) узла.

## Возможности dataflow систем

Pipeline dataflow очень проста в реализации, но, тем не менее, полезна. Однако, она накладывает ограничения на то, какие программы можно реализовать с её помощью.

Unix pipes, цепочка ответственности, stream processors
и фильтры - всё это pipeline dataflow. Это настолько очевидное решение, что оно переизобреталось множество раз. Посмотрим, как можно расширить pipeline dataflow, чтобы сделать его мощнее. Некоторые такие комбинации дают нам Synchronous Dataflow и Flow-Based Programming.

Некоторые возможности лучше не совмещать. Например, изменяемость данных и разделение дуг - придётся делать глупокое копирование для каждого получателя.

### Push vs Pull data

Определяет то, как токены движутся по системе.

Push - узлы шлют данные другим узлам всякий раз, когда можно. Поставщик данных управляет движением. Браузер, посылающий запросы серверу, это пример _push data_.

Pull - получатель управляет движением. Один узел запрашивает данные у второго, тот у третьего и так далее по цепочке до тех пор, пока не находится узел, который не запрашивает данные ни у кого.

Pull подход редко используется в dataflow системах, однако, полезен, когда вычисление, которое делает узел, дорого - pull позволяет его _отложить_ (lazy) до тех пор, пока оно кому-то не понадобится.

### Изменяемые и неизменяемые данные

Рассмотрим пример: один и тот же токен отправлен в 2 узла и был изменён в одном из них. В случае мутабельных данных, узел 2 получает изменённую версию и, если тоже изменяет данные, то данные из узла 1 потеряны. Изменяемость данных усложняет ключевую фишку dataflow - возможности параллельного исполнения множества узлов без необходимости думать о блокировках. Неизменяемость предпочтительна всегда, когда речь идет о параллелизме. Как уже было сказано, появляется необходимость (глубокого) копирования данных (которые отображает токен) при наличии split функции.

Копирование токена неизбежно при наличии мутаций. Мутации неизбежны при _интеропе_ с императивными языками.

### Статика VS Динамика

#### Динамика

Позволяет динамически изменять граф или переопределять узлы в _рантайме_. Похоже на механизм HOF в "обычных" языках.

Подмена узла (_Node replacement_) - замена узла другим во время выполнения. Подмена дуги (_Arcs modifying_) - другой подход, изменений структуры графа в рантайме. Звучит круто, но по факту ведёт усложнению дебага.

Замена узла безопаснее чем дуги, но также должна использоваться в случае реальной необходимости. Например, уменьшить кол-во шаблонного кода, когда его реально много. Оба - замена узла и замена дуги - ведут к усложнению дебага.

#### Статика

Никакие изменения не могут быть сделаны в рантайме. Граф (узлы и дуги) - зафиксированы. Это похоже на то, как нельзя изменить (статичную) структуру в языках вроде C.

Статичные dataflow программы могут, заметим, использовать сколько угодно копий одного и того же узла. Обратное ограничение относится к dataflow железу, но не софту.

Статичный dataflow похож на реальную электронную схему - нельзя просто так "из воздуха" создать новый узел. Структура схемы известна заранее и неизменяема.

### Функциональные (чистые) узлы или с состоянием

Первые не имеют стейта и их вывод зависит исключительно от ввода, вторые могут иметь внутреннее состояние.

Подобно тому, как рекурсивная функция может накапливать состояние, передавая новые аргументы в рекурсивные вызовы, любой dataflow узел с состоянием может избавиться от состояния, посредством добавления выходного порта, обращённого через дугу в свой же вход.

В зав-ти от реализации, петли могут, т.к. требуют доп. дуг, ухудшить производительность и, возможно, лучше добавить состояние. Узлы без состояния, однако, способствуют более простому дизайну.

Неизменяемость данных хорошо сочетается, хотя и не обязана быть использована в комбинации, со stateless узлами. Обратное справедливо и для изменяемости в соечетании со stateful узлами.

### Синхронная VS Асинхронная активация

#### Асинхронная

Узел активируется всякий раз, когда тому способствуют условия. Возможно, одновременно с другими узлами. Часто комбинируется с _вместимостью дуги_ более одного токена. У узлов могут быть _всплески активности_ и метрика создания токенов может скакать.

#### Синхронная

Это когда ноды активируются в заранее определённой манере, строго определяющей их порядок и то, какие из них могут быть активированы одновременно. В каких-то ситуациях, когда мы заранее чётко знаем, в каком порядке должна происходить активация, такой подход может быть более производительным. В других, однако, на этапе компиляции это невозможно, так как частота генерации токенов в разных местах неизвестна и может зависеть от нагрузки во время исполнения.

Синхронная активация не может комбинироваться с динамическим подходом т.к. порядок активаций должен быть определён на этапе компиляции.

#### Гибрид

Можно иметь оба движка и использовать синхронный для изолированной части приложения, где порядок строго известен, а второй для внешней координации.

Синхронный dataflow не может иметь _взаимной блокировки_ (deadlock), но не всякую программу можно выразить в терминах синхронного dataflow.

### Множественные входы и выходы

#### Множественные входы

Pipeline dataflow позволяет лишь один вход и выход для узла, что подобному языку, где функции могут иметь только один входной параметр.

Множественные порты нужны для выполнения самых простых задач таких как сложение, однако, их добавление усложняет логику активации узла - узел может быть активирован когда данные есь на всех портал или хоть на одном (ALL vs ANY). Некоторые операции, однако, независимо от принятого решения невозможны, в силу своей природы, пока данные не появятся на всех портах - например, сложение (или вычитание) чисел. Другие же узлы, напротив, могут работать с частно приходящими данными. Другой пример это merge узел, который сливает несколько входных потоков в один. Он, напротив, может (и, вероятно, должен, для избежания лишних блокировок) обрабатывать данные по мере их поступления. Добавление множественных портов выходит за рамки pipeline dataflow.

#### Множественные выходы

Множественные выходы не заставляют нас думать о _правилах активации_ (fireing rules) в отличии от входов. Как правило, узел имеет несколько выходов потому что есть несколько вариантов ответа.

### Паттерны активации

Условие, которое необходимо соблюсти, чтобы узел был активирован. Список кодов:

- 1 - токен ожидает входа
- 0 - порт не имеет ожидающих токенов
- X - не важно
- - - тоже не важно, всегда активировать

Примеры паттернов, описывающих активацию узла: `[1,1,X]`, `[1,1,*]` или так (с указанием конкретных портов) - `[1:input a, 1:input b, X:input c]`.

Узел может иметь несколько паттернов активации, они читаются слева направо и считаются сопоставленными, как только сопоставлен первый попавшийся. Любая реализация может позволить программисту самостоятельно указывать эти паттерны для каждого узла. Паттерн активации для узла `merge` выглядел бы так (пример того, что один узел имеет несколько паттернов):

- `[1, X, X]`
- `[X, 1, X]`
- `[X, X, 1]`

Узел без входных портов, также известный как _узел источник_ (source node), имеет паттерн `[*]` т.к. его активация происходит всегда.

Правило "ALL" из пред. секции про множественные порты имеет вид `[1, 1, 1, ...]` и "ANY" - `[1, X, X], [X, 1, X], [X, X, 1, ...]`.

### Циклы и обратная связь

Термины _петли_ (loops), _циклы_ (cycles) и _обратная связь_ (feedback) все имеют один и тот же смысл. Когда данные из одной части графа посылаются в обратном направлении. В теории графов это называется _цикличный граф_ (pipeline dataflow, однако, ацикличен). В обычном программировании мы часто называем использование цикла _итерированием_.

Примером простейшей программы может служить возведение в степень, при условии, что у нас есть начальное значение для множителя (1), иначе (из-за паттерна ALL `[1,1]`) нас ждёт взаимная блокировка. Мы шлём в `x` узел (умножение) некоторый `n` + токен, отображающий постоянно нарастающий множитель.

Начальное значение это токен, который привязан к дуге ещё до того, как программа стартует. Это распространённая практика в dataflow системах, однако, ей есть альтернатива - _одноразовые_ (one-shot) узлы. У разработчиков часто есть страх зацикленных dataflow программ, дескать, легко написать их с ошибкой так, что они никогда не прекратят выполняться. Это необоснованный страх, основанный на привычке работать с однопоточным кодом:

- Dataflow программы _предназначены_ работать постоянно/продолжительное время
- Циклы можно не использовать (но есть подмножество программ, невозможных без них)
- Проектировать схему так, чтобы циклы прекращались

### Рекурсия

Узел содержит сам себя. Это предполагает, что разрешены _составные узлы_. Рекурсия похожа на цикл с тем различием, что кол-во "итераций" (рекурсивных вызовов) неизвестно до времени выполнения, а условие для остановки рекурсии является внутренним свойством узла.

Цикл:

```
x = 0
for (count = 0 to 100) {
    x = node(x)
}
```

Переводя это в dataflow: узел `node` с 1 входом и 1 выходом имеет дугу от своего выхода к своему входу с доп.узлом по середине, который прерывает цикл, если `count>100`. Обратим внимание, что цикл (петля дуги) и его условие являются внешними по отношению к самому узлу.

```
function int node(int x) {
    if (x <= 0) {
        return 0
    } else {
        return x + (node(x-1))
    }
}
```

В терминах dataflow `node` это узел с также 1 вводом и 1 выводом, определённый в терминах самого себя. Условие прекратить рекурсию заключено у него внутри.

#### Реализация рекурсивных узлов

Трудна в реализации т.к. в отличии от традиционных ЯП в dataflow нет стека вызовов, а как-то различать, какие токены относятся к какому уровню рекурсии, на котором есть инстанс одного и того же узла, всё же надо.

В dataflow для этого применяется техника "цветных токенов" - каждому уровню рекурсии присваивается "цвет" (по факту просто id). Токены, созданные на определённом уровне имеют соотв. цвет. Узел одновременно принимает токены только своего цвета. 

Вместимость дуги должна быть больше 1, поскольку отныне дуга содержит токены для всех последующих уровней рекурсии.

### Составные узлы

Возможность составлять узлы, используя композицию других, фундаментальна в отношении уменьшения сложности и переиспользования кода. Её отсутствие - всё равно что программирование без функций. _Примитивные_ (атомарные, элементарные) узлы - те, что не состоят из других. Подобны _ключевым словам_ (или операторам) в обычных ЯП.

####  Исполнение составных узлов

В статичном dataflow наличие составных узлов вопрос просто удобства - на этапе компиляции сложная структура программы "разворачивается" в плоскую, где есть только примитивные узлы. Это то, что dataflow движок по факту исполняет.

В динамичном всё сложнее. Есть 2 стратегии: 1) работать с составными узлами как с просто обёртками, которые распределяют траффик между своими (возможно, также составными) узлами. Этот подход замедляет поток токенов. 2) более эффективный, но и более сложный подход заключается в применинии (по своей сути) JIT-компиляции - модификация графа в рантайме так, чтобы он включал только примитивные узлы. Любые изменения в рантайме влекут за собой такую "рекомпиляцию". В динамичных системах, где структура графа в основном статична, подход с JIT уместен, а там, где изменения регулярны, оптимальней выбрать решение с обёртками.

### Вместимость дуги > 1

Есть множество причин, почему дуге стоит иметь вместимость больше одного токена. В асинхронных системах один узел может порождать токены быстрее, чем другой будет успевать их поглощать. _Высокочастотные_ (multi-rate) dataflow системы спроектированы так, чтобы порождать несколько токенов за активацию и требуют вместимости дуги более одного токена. 

Некоторые dataflow программы могут ловить _взаимную блокировку_ при вместимости 1 и не ловить её иначе. Системы могут быть вместимость как фиксированную (в т.ч. больше 1), так и изменяемую. Наиболее гибкие dataflow системы дают программисту устанавливать вместимость каждой дуги.

Бесконечная вместимость может эмулироваться с помощью заранее выделенного участка памяти который, будучи заполнен, должен выбрасывать исключение.

### Соединение и разделение дуги

Разделение - 1 вход и 2 или более выхода у дуги. Токены копируются для каждого направления. Изменяемость данных заставляет делать глубокое копирование. 

Соединение производит эффект, который зависит от того, как вы его определите. Наиболее распространённый способ заключается в том, чтобы просто пропускать токены в том порядке, в котором они пришли. Также есть подход, при котором токены могут сливаться. Например, 2 числовых токена могут быть сложены друг с другом, породив новый, третий токен вместо себя.

Некоторые реализации dataflow не разрешают _разделения_ (splits) или _соединения_ (joints). Так, например, FBP не позволяет разделений, а большинство синхронных dataflow систем - соединений.

И соединения и разделения оба могут быть эмулированы узлами. 

### Высокочастотная отправка и получение токенов

Высокочастотные системы позволяют узлам отправлять и получать 1 или более токенов за активацию. Кол-во токенов может быть фиксированным или изменяемым. Чем больше токенов может отправляться и приниматься за активацию, тем выше, в общем случае, производительность dataflow системы. Отсюда также следует, что вместимость дуги должна быть более одного.

## Распространённые dataflow узлы

### Свич / выбор 

2 и более входа, 1 специальный "контрольный" вход и 1 выход. Направляет один из входов на выход в зав-ти от значения на контрольном входе.

### Узел слияния / соединения

То же самое, что и дуга соединения

### Распространитель / разделитель

То же самое, что дуга разделения

### Gate узел

1 вход, 1 выход и 1 контрольный вход. Пропускает или блокирует входной сигнал. Чаще всего сами данные не важны.

### Terminal узел

1 вход и 1 выход. Данные протекают без изменений. Допустим вариант со множеством портов.

### Source узел

Не имеет входов и имеет 1 или больше выходов. Генератор source токенов. Имеет особые правила активации за неимением входов.

### Sink узел

1 или более входов и 0 выходов. Токены входят туда и никогда не возвращаются, как из чёрной дыры. Часто используется для интеропа с другими языками.

## Дополнительно

### Гранулярность

Параллелизм в dataflow измеряется степенью _гранулярности_ системы. Маленькая гранулярность (мелкозернистость) это когда узлы малы, просты, выполняют примитивные операции вроде сложения. Большая гранулярность это когда узлы скорее как сопрограммы целого приложения. Нет конкретного правила, что есть какая гранулярность.

Системы с большой гранулярностью обычно имеют асинхронные правила активации т.к. их сопрограммы крупны и они используют dataflow скорее как message-passing. Мелкозернистые, как электросхемы, напротив, применяют синхронную активацию из-за своей более высокой пропускной способности.

## А когда закончить?

Основной источник недопонимания dataflow это убеждение, что программа должна прекратить своё выполнение. Dataflow программа должна работать до тех пор, пока мы явно её не выключим. Dataflow оперирует потоками, тогда как обычные программы - одиночными значениями. Возьмём для примера мат.функцию `sin` - она даст ответ для одного конкретного значения, но для потока входных значений ответом будет поток выходных. 

Кажется очевидным, но стоит нам добавить циклы, как даже одиночное значение способно порождать потоки токенов на выходе. Если dataflow программа должна просто вычислить значение и прекратить своё выполнение, скорее всего, циклов лучше избежать и придержаться обыкновенного pipeline подхода. Чуть менее надёжный подход, для ситуаций когда циклов не избежать, заключается в том, чтобы добавить дополнительный "терминирующий" узел, проверяющий условие и, когда надо, прерывающий цикл.

В большинстве инженерных профессий устройства, ожидается, должны работать продолжительное время. Лучшие устройства, в конце концов, это те, что не перестают работать. Это всё влияние идей Фон Нейманна, из-за которого программисты до сих пор уделяют столько внимания тому, как и когда программа должна прекратить выполняться.