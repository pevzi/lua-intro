#### generic for

TODO

#### Метатаблицы

TODO

#### Coroutines

Сопрограммы (coroutines, _разг._ корутины) - отдельные потоки программы, выполнение которых можно приостановить, а затем продолжить с той же точки. И нет, это не совсем те потоки, о которых вы могли подумать. Корутины в Lua реализуют так называемую кооперативную многозадачность, при которой в конкретный момент выполняется только один поток, а переключение на другой поток происходит только тогда, когда выполняющийся поток это явно запрашивает. Этот вид многозадачности противопоставляется вытесняющей многозадачности, при которой переключение потоков производится планировщиком автоматически и может произойти в любой момент.

Управление корутинами производится с помощью функций из глобальной таблицы coroutine. coroutine.create создаёт новую сопрограмму, принимая в качестве аргумента функцию, с которой следует начать выполнение нового потока.

```lua
local co = coroutine.create(function ()
    print("hi")
    coroutine.yield()
    print("bye")
end)
```

Значение, которое возвращается из coroutine.create, имеет тип thread и используется для управления созданной сопрограммой. Сопрограмма не начинает своё выполнение сразу, а создаётся в приостановленном состоянии. В этом можно убедиться с помощью функции coroutine.status:

```lua
print(coroutine.status(co)) --> "suspended"
```

Чтобы начать или возобновить выполнение сопрограммы, необходимо использовать coroutine.resume. При этом текущий поток приостанавливает своё выполнение, и начинает выполняться указанная сопрограмма.

```lua
coroutine.resume(co) --> "hi"
```

Функция coroutine.yield приостанавливает выполнение текущей корутины и возвращает выполнение вызвавшему её потоку. Таким образом, рассмотренная в данном примере сопрограмма приостановит своё выполнение после вывода сообщения "hi", снова перейдя в состояние "suspended", а главный поток продолжит выполнение. Повторный вызов coroutine.resume позволит продолжить выполнение корутины с той точки, на которой оно остановилось в прошлый раз:

```lua
coroutine.resume(co) --> "bye"
```

Когда функция, переданная в coroutine.create, завершает своё выполнение, или если в корутине возникает ошибка, эта корутина умирает без возможности возобновить или перезапустить её.

```lua
print(coroutine.status(co)) --> "dead"
```

Если корутина была приостановлена с помощью yield или успешно завершила выполнение, то coroutine.resume возвращает true. Если при выполнении корутины произошла ошибка (в том числе, если была попытка возобновить умершую сопрограмму), то coroutine.resume возвращает false и сообщение об ошибке:

```lua
print(coroutine.resume(co)) --> false   "cannot resume dead coroutine"
```

При запуске и приостановке сопрограмм есть возможность передавать значения между сопрограммами. При первом запуске сопрограммы все дополнительные значения, переданные как аргументы в coroutine.resume, попадут в аргументы главной функции сопрограммы. Возвращаемые же функцией значения будут возвращены из coroutine.resume наряду с true, символизирующим успешное выполнение корутины. Не очень впечатляющий, но иллюстративный пример:

```lua
local add = coroutine.create(function (a, b)
    return a + b
end)

print(coroutine.resume(add, 1, 2)) --> true  3
```

Если coroutine.resume используется для возобновления выполнения корутины, приостановленной с помощью yield, то переданные в resume значения возвращаются из coroutine.yield, на котором остановилось выполнение возобновляемой сопрограммы. И наоборот, значения, переданные в качестве аргументов функции coroutine.yield, возвращаются из coroutine.resume.

TODO: более жизненный или более простой пример? этот можно реализовать и замыканиями

Следующая сопрограмма принимает числа по одному и yield'ит среднее среди всех принятых значений:

```lua
math.randomseed(os.time())

local average = coroutine.create(function (initial)
    local sum = initial
    local n = 1

    while true do
        sum = sum + coroutine.yield(sum / n, n)
        n = n + 1
    end
end)

for i = 1, 10 do
    print(coroutine.resume(average, math.random(0, 10)))
end
```

Первое число, переданное в resume, попадёт в переменную initial. Остальные будут возвращаться из функции coroutine.yield. В свою очередь, сопрограмма выдаёт промежуточные результаты, передавая их в качестве аргументов в coroutine.yield, и они возвращаются из функции coroutine.resume.

Результат выполнения кода выше выглядит примерно так:

```
true    9.0     1
true    7.0     2
true    8.0     3
true    6.25    4
true    6.4     5
true    7.0     6
true    6.1428571428571 7
true    5.625   8
true    5.3333333333333 9
true    4.9     10
```

TODO:
- у каждой корутины свой стек, т.е. йилднуться можно из любой функции
    - The coroutine cannot be running a C function, a metamethod, or an iterator. - 5.1
- юзкейзы - сценарии как в beep, веб-краулер
    - http://keplerproject.github.io/copas/manual.html#introduction
    - http://www.lua.org/pil/9.4.html
    - подходы типа coil, convoke, hump.timer
    - http://lua-users.org/wiki/LuaCoroutinesVersusPythonGenerators
- чем круто (переключение дешёвое, без привлечения ОС, предсказуемое, не требуются механизмы синхронизации)
- coroutine.wrap
- "normal" if the coroutine is active but not running (that is, it has resumed another coroutine)

из вики:

Coroutines are useful to implement the following:

State machines within a single subroutine, where the state is determined by the current entry/exit point of the procedure; this can result in more readable code compared to use of goto, and may also be implemented via mutual recursion with tail calls.
Actor model of concurrency, for instance in video games. Each actor has its own procedures (this again logically separates the code), but they voluntarily give up control to central scheduler, which executes them sequentially (this is a form of cooperative multitasking).
Generators, and these are useful for streams – particularly input/output – and for generic traversal of data structures.
Communicating sequential processes where each sub-process is a coroutine. Channel inputs/outputs and blocking operations yield coroutines and a scheduler unblocks them on completion events.

из мануала:

function foo (a)
  print("foo", a)
  return coroutine.yield(2*a)
end

co = coroutine.create(function (a,b)
  print("co-body", a, b)
  local r = foo(a+1)
  print("co-body", r)
  local r, s = coroutine.yield(a+b, a-b)
  print("co-body", r, s)
  return b, "end"
end)

print("main", coroutine.resume(co, 1, 10))
print("main", coroutine.resume(co, "r"))
print("main", coroutine.resume(co, "x", "y"))
print("main", coroutine.resume(co, "x", "y"))

#### Окружения

Вы заметили, что неопределённые глобальные переменные ведут себя точно так же, как незаданные значения в таблицах? Они тоже равны nil, и установка значения nil эквивалентна удалению элемента. Это совпадение на самом деле не случайно: значения глобальных переменных действительно хранятся в специальной таблице, называемой окружением. По умолчанию для каждого чанка в качестве окружения устанавливается таблица, к которой можно обратиться через глобальную переменную \_G:

TODO: проверить действительно ли \_G устанавливается для модулей, включаемых через require
TODO: как-то убедиться, что чанки наследуют окружение от треда

TODO:

Threads are created sharing the environment of the creating thread. Userdata and C functions are created sharing the environment of the creating C function. Non-nested Lua functions (created by loadfile, loadstring or load) are created sharing the environment of the creating thread. Nested Lua functions are created sharing the environment of the creating Lua function.

Environments associated with userdata have no meaning for Lua. It is only a convenience feature for programmers to associate a table to a userdata.

Environments associated with threads are called global environments. They are used as the default environment for threads and non-nested Lua functions created by the thread and can be directly accessed by C code (see §3.3).

The environment associated with a C function can be directly accessed by C code (see §3.3). It is used as the default environment for other C functions and userdata created by the function.

```lua
foo = 5
print(_G["foo"]) --> 5
print(_G == _G._G) --> true

for k, v in pairs(_G) do
    print(k, v)
end
```

(Возвращаясь к производительности глобальных переменных. Так как обращение к глобальной переменной - это ни что иное, как обращение к элементу хеш-таблицы, а локальные переменные по сути берутся из массива по индексу (они хранятся в регистрах виртуальной машины), неудивительно, что второе происходит быстрее. Кстати, подробности реализации виртуальной машины Lua 5.0 можно почитать [здесь](http://www.lua.org/doc/jucs05.pdf). Думаю, большая часть написанного до сих пор актуальна как для современных официальных реализаций, так и для LuaJIT.)

Окружение сохраняется для каждой функции при её создании. По умолчанию оно совпадает с окружением функции, в которой эта функция была создана. Например, если у чанка установлено окружение \_G, то все функции, созданные непосредственно внутри этого чанка, будут так же иметь окружение \_G. Но существует возможность задать в качестве окружения функции любую другую таблицу, и тогда при обращении внутри этой функции к глобальной переменной будет происходить обращение к заданной таблице. Для получения и изменения окружения используются стандартные функции, соответственно, getfenv и setfenv:

```lua
local env = {v = 5, print = print} -- предоставляем доступ только к глобальной функции print и переменной v

local function foo()
    print(v, pairs) -- переменная v есть в env, переменной pairs нет
end

print(getfenv(foo) == _G) --> true

setfenv(foo, env) -- устанавливаем таблицу env в качестве окружения функции foo

foo() --> 5 nil
```

TODO: установка нового окружения для функции не действует ретроактивно на уже созданные в ней функции

Также getfenv и setfenv могут вместо функции принимать число, обозначающее уровень вложенности функции в стеке. Так, на уровне 1 будет текущая функция (то есть, та, из которой был вызван setfenv), на уровне 2 - функция, вызвавшая текущую и так далее.

```lua
local function foo()
    local env = {}
    setfenv(1, env) -- устанавливаем таблицу env в качестве окружения функции foo

    a = 1
    b = 2

    return env
end

local t = foo()
print(t.a, t.b)
```

В таком виде использование окружений напоминает инструкцию with из Pascal или ActionScript. Однако, обратите внимание, что после установки пустого окружения вы не сможете обращаться к стандартным функциям Lua и вообще к любым глобальным переменным. Самый простой способ предоставить к ним доступ - унаследовать глобальное окружение с помощью метаметода \__index:

```lua
local env = setmetatable({}, {__index = _G})
```

Этот подход можно использовать, например, при создании модулей:

```lua
local P = setmetatable({}, {__index = _G})
setfenv(1, P)

dist = function (x1, y1, x2, y2) -- dist попадает в таблицу P
    return math.sqrt((x1 - x2) ^ 2 + (y1 - y2) ^ 2) -- библиотека math остаётся доступна
end

return P
```

Так можно не бояться, что какая-то из глобальных переменных "утечёт" в \_G, ибо новые ключи в любом случае сохраняются в P.

TODO: почему не рекомендуется так делать модули?

TODO: As a special case, when f is 0 setfenv changes the environment of the running thread. In this case, setfenv returns no values.

Также использование окружений позволяет реализовать песочницу (sandbox), то есть, урезанное в возможностях окружение для запуска потенциально вредоносного кода. Можно предоставить такому коду лишь набор безопасных функций для взаимодействия с внешним миром, при этом отобрав у него возможность сделать что-то нехорошее (например, получить доступ к файлам через библиотеку io).

TODO: метатаблица на \_G для ограничения использования глобальных переменных

Да, заметьте, что сама по себе переменная \_G языком не используется. То есть, если вы присвоите этой переменной другую таблицу, это никак не повлияет на окружения функций. Однако, модифицировать саму таблицу, на которую ссылается переменная, конечно, можно.

Теперь обратите внимание, что написанное выше актуально только для Lua <=5.1 (и, соответственно, LuaJIT). В Lua 5.2 полностью переделали механизм, отвечающий за окружение функций. Если вы не планируете использовать новые версии Lua, следующий раздел можете не читать.

##### Окружения в Lua >=5.2

По сути, текущая реализация сводится к двум пунктам:

1. Обращение к любой глобальной переменной _var_ превращается в \_ENV._var_ (т.е. это просто "синтаксический сахар").
2. По умолчанию каждый чанк имеет upvalue \_ENV, равный таблице \_G.

Таким образом, \_ENV - это просто локальная переменная, так что можно, например, объявить новую переменную \_ENV, чтобы перекрыть уже существующую, или же изменить существующую \_ENV (в том числе, относящуюся к чанку).

TODO: пример с изменением \_ENV

Если в функции происходит обращение к глобальной переменной, но в функции нет локальной переменной \_ENV, функция замыкается на ближайшей внешней \_ENV:

TODO: пример из статьи leafo

TODO: Пример выше с использованием нового механизма можно переписать, например, так:

```lua
local function foo()
    local _ENV = {a = 5, print = print} -- если не использовать local, мы изменим _ENV, который выше (т.е. для чанка)
    print(a)
end

foo() --> 5
```

Или даже как-то так:

```lua
local foo

do
    local _ENV = {a = 5, print = print}
    foo = function ()
        print(a)
    end
end

foo() --> 5
```

Заметьте главное отличие от getfenv/setfenv. Если раньше окружение функции могло задаваться динамически после создания функции, то теперь окружение функции определяется лексически по обычным правилам обращения к переменным (точно так же, как используемые ей внешние локальные переменные). Теперь окружение функции считается такой же "неприкосновенной" частью функции, как её код и upvalues. К тому же, новый подход ближе к "философии Lua", так как использует уже существующий, более общий механизм вместо того, чтобы сохранять окружение в отдельном магическом свойстве функции. Новая система проще в реализации, и её внедрение позволило внести в язык [некоторые другие полезные изменения](http://www.inf.puc-rio.br/~roberto/talks/novelties-5.2.pdf).

Неудивительно, что это решение вызвало много споров.

TODO: function encapsulation
Changing the environment of a function after it has been loaded is not "regular functionality"; it breaks the concept of function encapsulation. All the situations where setting \_ENV is OK do not require the debug library. – Deco Aug 20 '12 at 2:20

TODO: load может принимать значение для \_ENV в качестве аргумента

TODO:
- можно реализовать старое поведение через debug
    - http://stackoverflow.com/questions/14290527/recreating-setfenv-in-lua-5-2
    - http://leafo.net/guides/setfenv-in-lua52-and-above.html
- http://lua-users.org/wiki/EnvironmentsTutorial
- http://stackoverflow.com/questions/20735570/modify-lua-chunk-environment-lua-5-2
