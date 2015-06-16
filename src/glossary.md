% Глоссарий

Не каждый пользователь Rust имеет опыт работы с системами программирования, или
необходимые знания в области компьютерной науки, поэтому мы добавили разъяснения
терминов, которые могут быть незнакомы.

<a name="arity"></a>
### Арность

Арность относится к числу аргументов функции или операции, котое они принимают.

```rust
let x = (2, 3);
let y = (4, 6);
let z = (8, 2, 6);
```

В приведенном выше примере `x` и `y` имеют арность 2. `z` имеет арность 3.

<a name="abstract-syntax-tree"></a>
### Абстрактное синтаксическое дерево (Дерево абстрактного синтаксического анализа)

Когда компилятор компилирует программу, он делает целый ряд различных вещей.
Одна из вещей, которые он делает, это преобразует текст вашей программы в
'Абстрактное синтаксическое дерево,' или 'AST.' Это дерево является
представлением структуры вашей программы. Например, `2 + 3` может быть
преобразовано в дерево:

```text
  +
 / \
2   3
```

А `2 + (3 * 4)` будет выглядеть следующим образом:

```text
  +
 / \
2   *
   / \
  3   4
```