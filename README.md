
Хто я? Миронюк Павло Ярославович. Software Developer. Люблю Rust, знаю JS, можу писати на Java. Використовую Arch та Vim. Сьогодні розкажу вам чому я люблю Rust і вважаю його найкращою мовою для backend розробки і не тільки.

Я не буду розказуваити детально де використовується Rust. Якщо це вас дуже сильно цікавить, то подивіться на цю цікаву статистику: [Rust Programming - The State of Developer Ecosystem in 2021 Infographic](https://www.jetbrains.com/lp/devecosystem-2021/rust/).

Ця стаття предназначена для людей, які взагалі не знають що таке Rust та як він працює, але знайомі хоча б із однією іншою мовою програмування.

## Зміст:

* Що тае Rust?
* Чому він класний?
* Мінуси Rust
* Як вчити Rust? Із чого почати?
* Післямова


## Що таке Rust?

[Вікіпедія](https://en.wikipedia.org/wiki/Rust_(programming_language)) нам каже, що:

`
Rust is a multi-paradigm, high-level, general-purpose programming language designed for performance and safety, especially safe concurrency. Rust is syntactically similar to C++, but can guarantee memory safety by using a borrow checker to validate references. Rust achieves memory safety without garbage collection, and reference counting is optional.
`

В загальному це хороше визначення, хоча, звичайно, повністю не розкариває суті Rust-а. Єдине із чим я не згодний, то це слова про схожість із синтаксисом С++. На мою думку це не так. Я залишу тут статтю на Habr ([Так ли токсичен синтаксис Rust?](https://habr.com/ru/post/532660/)) із аналізом синтаксису Rust. Далі я його обговорювати не буду, оскільки це справа смаку. Особисто мені він дуже подобається.

Перше, із чого я хочу почати, то це memoty model. І так, Rust в нас повністю compile мова. Тобто ваший вихідний код компілюється у виконуваний файл (.exe on Windows and binary file on Linux. Like in C/C++). Тут не має ніякого інтерпретатора, віртуальної машини чи garbage collector-а (як в Java, JS, python). Не має ніякого фонового процесу, який шукав би та видаляв би куски непотрібної пам'яті. Це означає, що інструкції для видалення пам'яті вже закладені всередині отриманого виконуваного файлу після компіляції. Як ці інструкції там з'являються? Якщо ми візьмемо для прикладу С++, то там ми руками вказуємо коли і який кусок пам'яті видалити `delete ptr;`. Така система має багато незручностей: ми можемо випадково не видалити якусь пам'ять (тоді буде memory leak), ми можемо спробувати видалити пам'ять яка вже видалена (тоді програма просто ~~крашнеться~~). В Rust-і ми явно не вказуємо який кусок пам'яті коли видаляти. Справа в тому, що на етапі компіляції компілятор сам ставить в потрібних місцях інструкції для видалення конкретних областей пам'яті (умовно він сам розставляє `delete ptr;`). Це є найкласніша штука в Rust: компілятор нам завжди гарантує, що в нас не буде меморі ліка (це не на 100% правда, потім роскажу чому). Саме тому всі кажуть, що він `designed for performance and safety, especially safe concurrency and can guarantee memory safety`. 

Тут виникає логічне питання: як компілятор розуміє, що певна область пам'яті нам не потрібна? Відповідь в наступному розділі.

## Чому він класний?

### Borrow checker

Я не буду сильно вдаватися в деталі як працює Rust. Просто пару речень для загального розуміння.

Давайте введемо три правила:

1. Кожне значення має свого власника (owner).
2. В кожного значення тільки один власник. 
3. Коли власник виходить за межі свого контексту (scope), то це значення видаляється (пам'ять, виділена під це значення, очищається).

Це є три фунтаментальні правила, за якими працює Rust. Що означає контекст (scope) я тут пояснювати не буду, оскільки припускаю, що ви знаєте що це таке. Тільки зауважу, що в Rust можна створювати власні контексти, як в JS:

```Rust
fn main() {
    {
        // new scope
        println!("Hello, world!");
    }
}
```

Приклад видалення пам'яті, коли власник виходить із свого контексту (також цей приклад показує що таке take ownership):

```Rust
fn debug(msg: String) {
    println!("{}", msg);
}

fn main() {
    let name = "Pavlo".to_owned(); // don't mind what is "to_owned()"
    println!("Author: {}", name); // ok
    debug(name);
    println!("Author: {}", name); // error. Memory of name already deleted
}
```

Коротко про те, що сталося. Спочатку `name` є власником пам'яті, виділеною під ім'я, та тримає це значення. Коли ми викликали функцію `debug` із параметром `name`, то власником цього значення стала змінна `msg`. Тут відбулася передача "прав" на значення, змінився власник (take ownership). Це означає, що в кінці функції `debug` змінна `msg` вийде за межі свого контексту і її значення буде видалено. Тобто після виклику функції `debug`, значення `"Pavlo"` буде видалене.

Насправді в кінці функції `debug` компілятор поставить спеціальну інструкцію, яка видалить значення `msg`. Саме через це Rust не гарантує ніде хвостової рекурсії. Навіть якщо ми напишемо функцію, на перший погляд, із хвостовою рекурсією, то компілятор може дописати інструкції видалення пам'яті (і наша хвостова рекурсія вже не буде хвостовою :smile: ).

Що робити, якщо нам потрібно передати значення у функцію, але потім його знову використовувати? Рішення: використовувати посилання (references). Ось перероблений код:

```Rust
fn debug(msg: &String) {
    println!("{}", msg);
}

fn main() {
    let name = "Pavlo".to_owned();
    println!("Author: {}", name); // ok
    debug(&name);
    println!("Author: {}", name); // ok
}
```

Примітка. Якщо ви працюєте із рядками, то краще використовувати `&str`.

Звичайне посилання (`&`) є імутабельним (immutable). Тобто на прикладі вище ми не можемо змінити значення `msg`. Щоб ми могли його змінити, треба використовувати мутабельні силки (mutable references `&mut`). Приклад:

```Rust
fn debug(msg: &mut String) {
    println!("msg: {}", msg); // msg: Pavlo
    msg.push_str(" Myroniuk");
}

fn main() {
    let mut name = "Pavlo".to_owned();
    println!("Author: {}", name); // Author: Pavlo
    debug(&mut name);
    println!("Author: {}", name); // Author: Pavlo Myroniuk
}
```

На кожне значення може бути тільки одна мутабельна посилання або багато імутабельних. Та штука, яка перевіряє передачу прав на значення, правильність використання силок і так далі, це `borrow checker`. Саме він буде казати, що ви дибіл, коли  перший раз компілюватимете свою програму на Rust :smile: (і скаже це не раз).

### No `null`

Да, в Rust не має `null`. Ви ніяк не зробите "заглушку". Якщо вам не потрібно відразу ініціалізовувати значення, наприклад, рядка, то у вас два варіанти:

1. Просто написати `let mut msg = "";` (тобто присвоїти якесь початкове, "дефолтне" значення)
2. Використовувати [`Option`](https://doc.rust-lang.org/std/option/). Приклад:
```Rust
let mut msg = Option::None;
// do some work and get the result
msg = Option::Some("some text");
```

Якщо у вас із функції не завжди вертається об'єкт, то замість того, щоб вертати `null`, найкращим способом буде використвоувати `Option`.

Якщо ви не впевнені, що відсутність `null`, це дуже і дуже класно, то почитайте [The billion dollar mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/). Також раджу прочитати статтю із блогу одного чувака: [Why null sucks, even if it's checked](https://ihatereality.space/01-why-null-sucks/).

Із свого досвіду можну сказати, що відсутність `null` дуже сильно допомагає.

### No `exceptions`

Да, тут Rust теж обділили :) В нього не має ексепшинів, як всіх привичних нам мовах (like Java, JS, etc). Якщо у нас виникла десь помилка, то є два варіанти:

1. Якщо помилка не страшна і не критична для програми, то функція нехай вертає [Result](https://doc.rust-lang.org/std/result/).
2. Якщо це критична помилка і відновлення неможливе, то ми панікуємо `panic!("error msg")`. В такому випадку робота програми закінчиться та повернеться повідомлення, яке ми передали в макрос `panic!`.

На перший погляд може здатися, що кожен раз перевіряти, що вернула фкункція `Result::Ok` чи `Result::Err`, є не сильно зручно і заморочно, але це направді не так. У Rust є багато способів обробки помилок. Детальніше почитайте в цій [статті]().

На практиці такий спосіб роботи із помилками дуже зручний та допомагає уникнути багатьох помилок. Для прикладу я взяв випадок, коли усі помилки зазвичай не враховуються, а якщо враховуються, то обробляються дуже погано - парсинг. Припустимо нам потрібно написати рапсинг .obj файлу і ми пишемо функцію, яка парсить рядок, що відповідає за вершину в просторі. На Rust я це колись написав ось так:
```Rust
fn parse_vertex(parts: &[&str]) -> Result<Vertex, SceneIOError> {
    let x = parts[0].parse().map_err(|err| SceneIOError::FailedToReadObj {
        description: format!("Failed to parse vertex x: {}", err),
    })?;
    let y = parts[1].parse().map_err(|err| SceneIOError::FailedToReadObj {
        description: format!("Failed to parse vertex y: {}", err),
    })?;
    let z = parts[2].parse().map_err(|err| SceneIOError::FailedToReadObj {
        description: format!("Failed to parse vertex z: {}", err),
    })?;
    let w = if parts.len() >= 4 {
        parts[3].parse().map_err(|err| SceneIOError::FailedToReadObj {
            description: format!("Failed to parse vertex w: {}", err),
        })?
    } else {
        1.0
    };
    Ok(Vertex { x, y, z, w, })
}
```

Тепер проаналізуємо цей код.

* Для кожної координати (x, y, z, w) ми робимо парсинг та відразу описуємо, яка помилка повернеться. І якщо там дійсно помилка, то вона повернеться із функції. Як би це була інша мова, то цей код просто помістили б в один `try {} catch {}` блок і користувачеві просто повернулася б інформація про факт невдалого парсингу. У нашому випадку також буде повідомлено що саме невдалося розпарсити.
* `if {} else {}` блок повертає значення. Тобто ми завжди можемо написати `let a = if parts.len() >= 4 { parse() } else { 1.0 };`
* результатом вважається значення останнього виразу. ` else { 1.0 }`. Аналогічно ми можемо написати із функціями (сорі за складний дженерік):
```Rust
fn sum<T: Add + Add<Output = T>>(a: T, b: T) -> T {
    a + b
}
```
* Ми можемо конструювати об'єкт певного типу "на ходу". Мені це трохи нагадує JS: `let v = Vertex { x, y, z, w, };`

### Compiler

Вище я вже розповів, який компілятор Rust-а класний, як багато він первіряє і так далі. Тут я хочу показати приклад помилоки компіляції. Візьмемо функцію `sum` із прикладу вище та опишемо дженерік неправильно:
```Rust
fn sum<T: Add>(a: T, b: T) -> T {
    a + b
}
```

При спробі компіляції нам повернеться наступна помилка:

```
error[E0308]: mismatched types
 --> src/main.rs:4:5
  |
3 | fn sum<T: Add>(a: T, b: T) -> T {
  |        - this type parameter  - expected `T` because of return type
4 |     a + b
  |     ^^^^^ expected type parameter `T`, found associated type
  |
  = note: expected type parameter `T`
            found associated type `<T as Add>::Output`
help: consider further restricting this bound
  |
3 | fn sum<T: Add + Add<Output = T>>(a: T, b: T) -> T {
  |               ^^^^^^^^^^^^^^^^^

error: aborting due to previous error
```

Нам тут сказано:
1. Причина та місце помилки: `mismatched types`, `src/main.rs:4:5`.
2. Пояснення чому саме ця помилка виникла і чому компілятор очікує саме такі типи: `this type parameter  - expected `T` because of return type`, `expected type parameter `T`, found associated type`.
3. Нам показаний код помилки: `error[E0308]`. Це означає, що ми можемо перейти на сторінку [https://doc.rust-lang.org/stable/error-index.html#E0308](https://doc.rust-lang.org/stable/error-index.html#E0308), де буде описано детально ця помилка, її додатковий приклад та можливі шляхи вирішення.
4. Нам запропонований варіант її вирішення (який є правильним): `<T: Add + Add<Output = T>>`.

Я ще не бачив ніякого компілятора, який так детально описував би помилку та давав так багато інформації про неї.

### Pattern matching

Зараз буде трохи складно та прикольно. Ні в одній із популярних мов програмування не має такої штуки як pattern matching. У останніх версіях Java їх починають додавати потроху, але це є просто продвинутий `instanse of`. Єдина мова, в якій я бачив подібний механізм, то це [SML](https://en.wikipedia.org/wiki/Standard_ML). Я не буду вдаватися в теорію що це таке. Покажу усе на прикладах.

В нас є enum `Result`, який я згадував вище:
```Rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```
Та функція, яка завантажує конфіг із файлу та повертає `Result`:
```Rust
// Config and ConfigError are custom types
fn load_config(filepath: &Path) -> Result<Config, ConfigError>;
```
Тепер її використання:
```Rust
fn init_app() {
    let config = load_config(Path::new("config.json"));
}
```

### Деякі інші плюшки

Як то кажуть: дрібниці, але приємно.

* **Shadowing**. Це коли ми об'являємо нову змінну із назвою, яка вже в нас є. При цьому відбувається не помилка, а просто "затінення" попередньої змінної. Приведу простий приклад, коли це може бути корисно. Припустимо ми робимо логіку для аутентифікації за паролем:
```Rust
let user = user_repository.find_by_usename(creds.get_username()); // user has type Option<User>
if (user.is_some()) {
    let user = user.unwrap(); // ok. now user has type User
    // check password
} else {
    // report an error
}
```
Хоча в ідеалі цей приклад можна переписати використовуючи pattern matching (що буде, на мою думку, краще):
```Rust
if let Some(user) = user_repository.find_by_usename(creds.get_username()) {
    // user has a type User. Now we can check password
} else {
    // report an error
}
```
* **Scalar types**. Мені дуже подобається, як описані скалярні типи в Rust: `u8, u16, u32, u64, u128` (думаю не треба пояснювати, що цифра біля букви - це кількість бітів) - це все беззнакові цілі числа (unsigned). `i8, i16, ...` - знаковмі цілі. `f8, f16, ...` - дробові числа (із плавачою точкою). Мені це здаєтья шикарним, простим та лаконічним.
* **Макроси**. Ну цей прикол зрозуміють ті, хто хоч трохи шарить за метапрограмування. Тут я це пояснювати не буду, просто знайте, що в Rust цей інструмент присутній та дуже потужний. `println!`, `vec!`, `dbg!` - одні із найпопулярніших стандартних макросів у Rust.

## Мінуси Rust

None

## Як вчити Rust? Із чого почати?

Я раджу починати із офіційної книги [The Rust Programming Language](https://doc.rust-lang.org/book/). У ній детально розписано як все працює в Rust.

Далі, думаю, ви вже самі знатимете, що хочете вивчати далі, але дуже раджу звернути увагу на наступні матеріали:

* [Ergonomic error handling with Rust](https://dev.to/senyeezus/ergonomic-error-handling-with-rust-13bj)
* [rust-cookbook](https://rust-lang-nursery.github.io/rust-cookbook/about.html). Якщо коротко, то в цій книжці знаходяться best practice по частими задачам із якими ми стикаємося у повсякденному житті: Time & Date, Databases, Command Line, Network, etc.
* [idiomatic-rust](https://github.com/mre/idiomatic-rust). В цьому репозиторієві зібрані патерни, характерні для раста, api guidelines, та інші посилання на класні навчальні матеріали по Rust. Тут ви точно знайдете щось, що ви не знаєте про Rust.
* [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - is a collection of runnable examples that illustrate various Rust concepts and standard libraries.
* [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) - детально розбирається будова структур даних, циклічних структур та інші теми, які ставлять новачків у ступор.
* Якщо вам ще мало, то ось тут точно для вас вистачить матеріалу із головою: [rust-learning](https://github.com/ctjhoa/rust-learning).

## Післямова

У цій статті я розказав, чому вважаю Rust дуже класною мп. Якщо вам сподобалася ця стаття, то ви можете:

* Follow me on Github: [@TheBestTvarynka](https://github.com/TheBestTvarynka). Поки що там не сильно великий актив, але коли я вирішу деякі питання, то продовжу контрібютити в open source
* Endorse me Rust skill on LinkedIn: [thebesttvarynka](https://www.linkedin.com/in/thebesttvarynka/) (I'll accept every connection request)

Чи будуть ще статті? Скоріш так, ніж ні, але не найближчим часом. Можлива наступна стаття буде про потоки, мультизадачність та асинхронність в Rust. Можливо про web development в Rust. Мижливо на обидві теми відразу. Залежить від того, як складеться моє майбутнє.
