% Небезопасный код

# Введение

Rust стремится обеспечить безопасные абстракции над низкоуровневыми деталями
процессора и операционной системы, но иногда бывает нужно спуститься и писать
код именно на низком уровне. Цель этого руководства - предоставить обзор
опасностей и мощи (эффективности), получаемых при использовании небезопасного
подмножества в Rust.

Rust обеспечивает аварийный люк в виде блока `unsafe { ... }`, который позволяет
программисту обойти некоторые из проверок компилятора и выполнить широкий спектр
операций, таких как:

- разыменование [raw pointers](#raw-pointers)
- вызов функций посредством FFI ([covered by the FFI guide](ffi.html))
- побитовое (поразрядное) преобразование типов (приведение типов, которое может
оказаться небезопасным и зависящим от реализации) (`transmute`, также известное
как "reinterpret cast")
- встраивание ассемблерного кода [inline assembly](#inline-assembly)

Обратите внимание, что блок `unsafe` не дает послабления для правил, относящихся
к: срокам жизни для `&` и замораживанию (фиксации) позаимствованных данных.

Любое использование `unsafe` равносильно заявлению программиста: "Я знаю больше,
чем ты", для компилятора. И таким образом, программист должен быть абсолютно
уверен в том, что он на самом деле знает что-то, почему этот кусок кода является
валидным. В целом, следует попытаться свести к минимуму количество опасного кода
в кодовой базе; предпочтительно использовать допустимо минимальное количество
`unsafe` блоков при создания безопасных интерфейсов.

> **Примечание**: низкоуровневые детали языка Rust еще могут меняться, поэтому
нет никакой гарантии стабильности или обратной совместимости. В частности, могут
быть изменения, которые не вызывают ошибок компиляции, но вызывают смысловые
изменения (например, приводящие к неопределенному (двусмысленному) поведению).
Таким образом, требуется крайняя осторожность.

# Указатели

## Ссылки

Одной из важнейших особенностей Rust является безопасность памяти. Она
достигается, в частности с помощью [системы владения](ownership.html), а именно:
благодяря ей компилятор может гарантировать, что каждая ссылка `&` всегда
валидна, и никогда не указывает на освобожденную память.

Эти ограничения для `&` имеют огромные преимущества. Тем не менее, они также
ограничивают и то, как мы можем их использовать. Например, поведение ссылок `&`
отличается от поведения указателей языка C, и поэтому ссылки не могут быть
использованы в качестве указателей в интерфейсах внеших функций (FFI). Кроме
того, и неизменяемые (`&`), и изменяемые (`&mut`) ссылки обладают некоторыми
гарантиями относительно псевдонимизации и замораживания (фиксации), необходимыми
для обеспечения безопасности памяти.

В частности, если у вас есть ссылка `&T`, то `T` не должен быть изменен с
помощью этой или любой другой ссылки. В стандартной библиотеке есть несколько
типов, например `Cell` и `RefCell`, которые обеспечивают внутреннюю
изменчивость, заменяя гарантии во время компиляции на динамические проверки во
время выполнения.

Ссылка `&mut` имеет различные ограничения: когда объект имеет ссылку `&mut T`,
указывающую на него, тогда эта `&mut` ссылка должна быть единственным возможным
способом обращения к этому объекту во всей программе. То есть, ссылки `&mut` не
могу быть псевдонимизированны (иметь псевдоним) любыми другими ссылками.

Использование `unsafe` кода, для того чтобы некорректно обойти и нарушить эти
ограничения, приведет к неопределеному (двусмысленному) поведению. Например,
следующий код создает два псевдонима для `&mut` указателя, и не является
валидным.

```
use std::mem;
let mut x: u8 = 1;

let ref_1: &mut u8 = &mut x;
let ref_2: &mut u8 = unsafe { mem::transmute(&mut *ref_1) };

// oops, ref_1 and ref_2 point to the same piece of data (x) and are
// both usable
*ref_1 = 10;
*ref_2 = 20;
```

## Сырье указатели (Raw pointers)

Rust предлагает два дополнительных вида указателя (*сырье указатели* (*raw
pointers*), написание которых `*const T` и `*mut T`. Они приблизительно
соответствуют `const T*` и `T*` языка C; действительно, одно из самых
распространенных их применений - для FFI, при взаимодействии с внешними
библиотеками C.

Сырье указатели дают гораздо меньше гарантий, чем другие типы указателей,
предоставляемые языком Rust и библиотеками. Например, они

- не гарантируют, что они указывают на действительную область памяти, и не
гарантируют, что они является ненулевыми указателями (в отличие от `Box` и `&`);
- не имеют никакой автоматической очистки, в отличие от `Box`, и поэтому требуют
ручного управления ресурсами;
- это простые структуры данных (plain-old-data), то есть они не перемещают право
собственности, опять же в отличие от `Box`, следовательно, компилятор Rust не
может защитить от ошибок, таких как использование освобождённой памяти (use-
after-free);
- лишены сроков жизни в какой-либо форме, в отличие от `&`, и поэтому компилятор
не может делать выводы о висячих указателях; и
- не имеют никаких гарантий относительно псевдонимизации или изменяемости, за
исключением изменений, недопустимых непосредственно для `*const T`.

К счастью, им присуща и положительная особенность: слабые гарантии означают и
слабые ограничения. Отсутствие ограничений делает сырые указатели подходящими в
качестве строительного блока для реализации внутри библиотек таких вещей как
смарт-указатели (умные указатели) и векторы. Например, для `*` указателей
разрешается задавать псевдонимы, что позволяет им быть использоваными при
написании типов для множественного права собственности, таких как указатели со
счетчиком ссылок и указатели со сборкой мусора и даже типов для потоко-
безопасного совместного использования памяти (типы `Rc` и `Arc` реализованы в
Rust в полной мере).

При работе с сырыми указателями есть две вещи, которые требуют большей
осторожности (т.е. требуют блок `unsafe { ... }`):

- pointer arithmetic via the `offset` [intrinsic](#intrinsics) (or
  `.offset` method): this intrinsic uses so-called "in-bounds"
  arithmetic, that is, it is only defined behaviour if the result is
  inside (or one-byte-past-the-end) of the object from which the
  original pointer came.
- разыменование: указатели могут содержать любое значение: поэтому возможные
результаты включают: аварии, чтение неинициализированной памяти, использование
памяти после освобождения или обычное чтение данных.
- арифметика указателей с помощью `offset` [intrinsic](#intrinsics)
(или `.offset` метода): внутреннее используется так называемая "in-bounds"
арифметика, то есть, он определен только поведение, если результат находится
внутри (или один байт после конца (one-byte-past-the-end)) объекта, из которого
первоначально указатель пришел.

Последнее предположение позволяет компилятору более эффективно оптимизировать
код. Как можно видеть, фактически *создание* сырого указателя не является
небезопасным, и не происходит преобразование указателя в целое число.

### Ссылки и сырые указатели

Во время выполнения и сырой указатель, `*`, и ссылка, указывающая на тот же
кусок данных, имеют одинаковое представление. По факту, ссылка `&T` будет неявно
приведена к сырому указателю `*const T` в безопасном коде, аналогично и для
вариантов `mut` (оба приведения могут быть выполнены явно, с помощью,
соответственно, `value as *const T` и `value as *mut T`).

Переход в обратном направлении, от `*const` к ссылке `&`, не являеся безопасным.
Ссылка `&T` всегда валидна, и поэтому, как минимум, сырой указатель `*const T`
должен указывать на правильный экземпляр типа `T`. Кроме того, в результате
указатель должен удовлетворять правилам псевдонимизации и изменяемости ссылок.
Компилятор предполагает, что эти свойства верны для любых ссылок, независимо от
того, как они были созданы, и поэтому любое преобразование из сырых указателей
равносильно утверждению, что они соответствуют этим правилам. Программист
*должен* гарантировать это.

Рекомендуемым методом преобразования является

```
let i: u32 = 1;
// explicit cast
let p_imm: *const u32 = &i as *const u32;
let mut m: u32 = 2;
// implicit coercion
let p_mut: *mut u32 = &mut m;

unsafe {
    let ref_imm: &u32 = &*p_imm;
    let ref_mut: &mut u32 = &mut *p_mut;
}
```

Разыменование с помощью конструкции `&*x` является более предпочтительным, чем с
использованием `transmute`. Последнее является гораздо более мощным
инструментом, чем необходимо, а более ограниченное поведение сложнее
использовать неправильно. Например, она требует, чтобы `x` представляет собой
указатель (в отличие от `transmute`).

## Делаем небезопасный код безопасным

Есть различные способы создать безопасный интерфейс вокруг некоторого
небезопасного кода:

- хранить указатели приватно (т.е. не в публичных полях публичных структур), так
чтобы вы могли видеть и контролировать всех, кто читает и пишет по указателю, в
одном месте.
- использовать макрос `assert!()` повсеместно: так как вы не можете полагаться
на защиту компилятора и систему типов, чтобы удостовериться в корректности
вашего `unsafe` кода во время компиляции, то используйте `assert!()`, чтобы
убедиться, что он работает правильно во время выполнения.
- реализовывать `Drop` для очистки ресурсов с помощью деструктора, и
использовать RAII (получение ресурса есть инициализация). Это уменьшает
потребность в каком-либо ручном управлении памятью пользователями, и
автоматически гарантирует, что очистка всегда выполняется, даже когда в потоке
произошла паника.
- гарантировать, что любые данные, хранящиеся по сырому указателю разрушаются в
соответствующее время.