# Избегаем состояния гонки в SharedArrayBuffers с помощью Atomics

*Перевод статьи [Lin Clark](http://code-cartoons.com/): [Avoiding race conditions in SharedArrayBuffers with Atomics](https://hacks.mozilla.org/2017/06/avoiding-race-conditions-in-sharedarraybuffers-with-atomics/). Распространяется по [лицензии CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/).*


Это третья статья в серии из трех частей:

1. [Быстрый курс по управлению памятью](https://medium.com/devschacht/a-crash-course-in-memory-management-b4863e000a5f)
2. [Иллюстрированное введение в ArrayBuffers и SharedArrayBuffers](https://medium.com/devschacht/a-cartoon-intro-to-arraybuffers-and-sharedarraybuffers-952198b0a1c9)
3. Избегаем состояния гонки в SharedArrayBuffers с помощью Atomics

---

В [предыдущей статье](https://medium.com/devschacht/a-cartoon-intro-to-arraybuffers-and-sharedarraybuffers-952198b0a1c9) я рассказала о том, как использование SharedArrayBuffers может привести к состоянию гонки. Это затрудняет работу с SharedArrayBuffers и мы не ожидаем, что разработчики приложений будут использовать SharedArrayBuffers напрямую.

Но разработчики библиотек, которые имеют опыт работы с многопоточным программированием на других языках, могут использовать эти новые низкоуровневые API для создания инструментов более высокого уровня, и разработчики приложений смогут использовать эти инструменты, чтобы не обращаться к SharedArrayBuffers или Atomics напрямую.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/02_15.png)

Несмотря на то, что вам, вероятно, не придётся напрямую работать с SharedArrayBuffers и Atomics, я думаю, что вам все еще интересно узнать, как они работают. Итак, в этой статье я расскажу, какие разновидности состояний гонки могут возникнуть, и как библиотеки Atomics помогают их избежать.

Но, во-первых, что такое состояние гонки?

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/02_13.png)

## Состояние гонки: пример, который вы, возможно, видели раньше

Довольно простой пример состояния гонки может случиться, если у вас есть переменная, доступ к которой есть у двух потоков. Скажем, один поток хочет загрузить файл, а другой поток проверяет, существует ли он. Для связи они используют общую переменную `fileExists`. 

Первоначально `fileExists` установлена в значение `false`.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_03.png)

Если код в потоке 2 исполняется первым, файл будет загружен.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_04.png)

Но если сначала выполнится код в потоке 1, то он выведет пользователю ошибку о том, что файл не существует.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_05.png)

Но это ложная ошибка. Дело не в том, что файл не существует. Реальная проблема — состояние гонки.

Даже в однопоточном коде многие JavaScript-разработчики сталкиваются с этим видом гонки. Вам не нужно ничего знать о многопоточности, чтобы понять, почему это гонка.

Тем не менее, есть некоторые виды состояний гонки, которые невозможны в однопоточном коде, но могут случится, когда вы работаете с несколькими потоками, и эти потоки обмениваются памятью.

## Различные виды состояний гонки и то, как Atomics помогает с ними справится
Давайте рассмотрим некоторые виды состояний гонки, которые вы можете получить в многопоточном коде и то, как Atomics помогает их предотвратить. Это не распространяется на все возможные состояния гонки, но должно дать вам представление о том, почему API предоставляет те методы, которые в нём заложены.

Прежде чем мы начнем, я хочу предупредить снова: вы не должны использовать Atomics напрямую. Написание многопоточного кода является известной сложной проблемой. Вместо этого вы должны использовать надежные библиотеки для работы с разделяемой памятью в многопоточном коде.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_06.png)

С этим нам не по пути...

## Состояние гонки в одной операции
Допустим, у вас было два потока, которые увеличивали одну и ту же переменную. Вы можете подумать, что конечный результат будет таким же, независимо от того, какой поток идет первым.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_07.png)

Но даже при том, что в исходном коде инкрементирование переменной выглядит как одна операция, если вы посмотрите на скомпилированный код, то увидите несколько операций.

На уровне CPU, приращение значения занимает три инструкции. Это связано с тем, что компьютер имеет как долговременную, так и кратковременную память. (Я более подробно рассмотрела то, как все это работает в [другой статье](https://hacks.mozilla.org/2017/02/a-crash-course-in-assembly/)).

![](https://hacks.mozilla.org/files/2017/06/03_08-768x521.png)

Все потоки делят долговременную память. Но кратковременная память(регистры) не разделяется между потоками.

Каждый поток должен получить значение из памяти в свою кратковременную память. После этого он может запустить вычисление этого значения в кратковременной памяти. Затем он записывает полученное значение обратно из своей кратковременной памяти в долговременную.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_09.png)

Если все операции в потоке 1 выполнятся первыми, а затем выполнятся все операции в потоке 2, мы получим результат, который ожидаем.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_10.png)

Но если они чередуются во времени, значение, которое поток 2 получил в свой регистр, не синхронизируется со значением в памяти. Это означает, что поток 2 не учитывает расчет потока 1. Вместо этого он просто затирает значение, которое поток 1 пишет в память.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_11.png)

Одной из задач атомарных операций является исполнение таких операций, которые люди считают одиночными операциями, но которые не являются таковыми для компьютера, как единого целого.

Вот почему они называются атомарными операциями. Это потому, что они выполняют операции, которые обычно имеют несколько инструкций (где инструкции могут быть приостановлены и возобновлены), и делают так, что все они кажутся выполненными одномоментно, как если бы это была одна инструкция. Это как неделимый атом.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_12.png)

При использовании атомарных операций, код для инкремента будет выглядеть несколько иначе.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_13.png)

Теперь, когда мы используем `Atomics.add`, различные этапы, связанные с инкрементированием переменной, не будут смешиваться между потоками. Вместо этого один поток завершит свою атомарную операцию и предотвратит запуск другого. После этого другой поток начнет свою собственную атомарную операцию.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_14.png)

Методы Atomics, которые помогают избежать гонки такого рода:

* [Atomics.add](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/add)
* [Atomics.sub](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/sub)
* [Atomics.and](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/and)
* [Atomics.or](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/or)
* [Atomics.xor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/xor)
* [Atomics.exchange](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/exchange)

Вы заметите, что этот список довольно ограничен. Он даже не включает такие вещи, как деление и умножение. Однако разработчик библиотеки может создавать атомарные операции для других вещей.

Для этого разработчик будет использовать [Atomics.compareExchange](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange). При этом вы получаете значение из SharedArrayBuffer, выполняете операцию над ним и записываете его обратно в SharedArrayBuffer только в том случае, если ни один другой поток не обновил его с момента первой проверки. Если другой поток обновил значение, вы можете получить это новое значение и повторить попытку.


## Состояния гонки в нескольких операциях
Таким образом, эти методы Atomics помогают избежать состояний гонки во время «одиночных операций». Но иногда вы хотите изменить несколько значений на объекте (используя несколько операций) и убедиться, что никто другой не вносит изменения в этот объект в тот же самый момент. В принципе, это означает, что в течение каждого применения набора изменений к объекту этот объект блокируется и недоступен для других потоков.

Объект Atomics не предоставляет каких-либо инструментов для обработки этой ситуации напрямую. Но он предоставляет инструменты, которые авторы библиотеки могут использовать для этого. Авторы библиотек могут создавать блокировки.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_15.png)

Если код хочет использовать данные с блокировкой от изменений, он должен получить доступ к этой блокировке. Затем он может использовать блокировку для ограничения доступа других потоков. Только он сможет получить доступ или обновить данные, пока блокировка активна.

Чтобы создать блокировку, авторы библиотеки будут использовать [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait) и [`Atomics.wake`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wake), а также [`Atomics.compareExchange`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/compareExchange) и [`Atomics.store`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/store). Если вы хотите понять, как это будет работать, взгляните на эту базовую реализацию блокировки.

В этом случае поток 2 получит управление блокировкой данных и установит значение `locked` в `true`. Это означает, что поток 1 не может получить доступ к данным до тех пор, пока поток 2 не разблокирует их.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_16.png)

Если поток 1 хочет получить доступ к данным, он попытается получить управление блокировкой. Но поскольку блокировка уже используется, он не сможет сделать это. Затем поток будет ждать (так что он будет остановлен) до тех пор, пока управление блокировкой не станет доступно.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_17.png)

Как только поток 2 будет завершен, он снимет блокировку. Механизм управления блокировками уведомит один или несколько ожидающих потоков о том, что можно перехватить управление.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_18.png)

Затем другой поток может перехватить управление блокировкой и заблокировать данные для собственного использования.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_19.png)

Библиотека управления блокировками может использовать много разных методов в объекте Atomics, но наиболее важными для этого варианта использования являются:

* [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)
* [`Atomics.wake`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wake)

## Состояния гонки, вызванные переупорядочением команд

Существует еще одна проблема синхронизации, о которой Atomics так же позаботился. Это может быть удивительно.

Вероятно, вы этого не осознаете, но есть очень хороший шанс, что код, который вы пишете, не работает в том порядке, в котором вы ожидаете. И компилятор и CPU переупорядочивают код, чтобы заставить его работать быстрее.

Например, допустим, вы написали код для вычисления общей суммы. Вы хотите установить флаг, когда расчет будет завершен.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_20.png)

Чтобы скомпилировать это, нам нужно решить, какой регистр использовать для каждой переменной. Затем мы можем перевести исходный код в машинные инструкции.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_21.png)

Пока все так, как и ожидалось.

Что не очевидно, если вы не понимаете, как компьютеры работают на уровне процессора (и как работают конвейеры, которые процессоры используют для выполнения кода), так это то, что строка 2 в нашем коде должна немного подождать, прежде чем она сможет быть исполнена.

Большинство компьютеров разбивают процесс выполнения инструкции на несколько этапов. Это гарантирует, что все части CPU будут заняты все время, чтобы наилучшим образом использовать все ресурсы.

Вот один из примеров шагов, через которые проходит инструкция:

1. Получить следующую команду из памяти
2. Выяснить, что говорит нам инструкция (иначе говоря, декодировать инструкцию) и получить значения из регистров
3. Выполнить инструкцию
4. Записать результат обратно в регистр


![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_22.png)

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_23.png)

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_24.png)

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_25.png)

Вот как одна инструкция проходит через конвейер. В идеале, мы хотим, чтобы вторая инструкция последовала сразу после первой. Как только она переместится на этап 2, мы хотим получить следующую инструкцию.

Проблема в том, что существует зависимость между инструкцией №1 и инструкцией №2.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_26.png)

Мы могли бы просто приостановить CPU до тех пор, пока команда №1 не обновит `subTotal` в регистре. Но это замедлит работу.

Чтобы сделать исполнение кода более эффективным, многие компиляторы и процессоры производят изменения порядка исполнения кода. Они ищут другие инструкции, которые не используют `subTotal` или `total` и помещают их между двумя этими строками.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_27.png)

Это обеспечивает постоянный поток инструкций, перемещающихся по конвейеру.

Поскольку строка 3 не зависит от каких-либо значений в строке 1 или 2, компилятор или процессор решают, что изменение порядка исполнения является безопасной операцией. Когда вы работаете в одном потоке, никакой другой код даже не увидит эти значения до тех пор, пока вся функция не будет выполнена.

Но когда у вас есть другой поток, работающий одновременно на другом процессоре, это не так. Другой поток не должен ждать, пока функция не будет выполнена, чтобы увидеть эти изменения. Он может видеть их почти сразу после их записи в память. Поэтому он может сказать, что `isDone` был установлен до вычисления `total`.

Если вы использовали `isDone` как флаг, что `total` подсчитан и может использоваться в другом потоке, то такое переупорядочение создало бы состояние гонки.

Atomics пытается решить некоторые из этих проблем. Когда вы используете атомарную запись, это похоже на создание забора между двумя частями вашего кода.

Атомарные операции не переупорядочиваются относительно друг друга, и другие операции не перемещаются вокруг них. В частности, две операции, которые часто используются для обеспечения порядка:

* [`Atomics.load`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/load)
* [`Atomics.store`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/store)

Все обновления переменных выше вызова `Atomics.store` в коде функции гарантированно завершатся, прежде чем `Atomic.store` завершит запись своего значения обратно в память. Даже если инструкции, отличные от атомарных, переупорядочиваются относительно друг друга, ни одна из них не будет перемещена ниже вызова `Atomic.store`, который был указан ниже в исходном коде.

Аналогично, все загрузки переменных после функции `Atomics.load` в функции гарантированно завершаться после того, как Atomics.load получит своё значение. Опять же, даже если неатомарные инструкции будут переупорядочены, ни одна из них не будет перемещена над `Atomics.load`, которая была указана выше их в исходном коде.

![](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2017/06/03_28.png)

Примечание. Цикл `while`, который я показываю здесь, называется циклической блокировкой, и он очень неэффективен. Если он находится в основном потоке, он может остановить ваше приложение. Вы почти наверняка не захотите столкнуться с этим в реальном коде.

Еще раз, эти методы на самом деле не предназначены для непосредственного использования в коде приложения. Вместо этого библиотеки будут использовать их для создания управляемых блокировок.

## Вывод
Программирование нескольких потоков, разделяющих память, является сложной задачей. Есть много разных состояний гонки, которые поджидают вас.

![Здесь могут водиться драконы](https://hacks.mozilla.org/files/2017/06/03_29-768x275.png)

Вот почему вам не нужно напрямую использовать `SharedArrayBuffers` и `Atomics` в коде приложения. Вместо этого вы должны опираться на проверенные библиотеки от разработчиков, которые имеют опыт работы с многопоточными процессома и которые потратили время на изучение того, как работает память.

`SharedArrayBuffer` и `Atomics` ещё молоды и такие библиотеки пока не созданы. Но эти новые API обеспечивают базовую основу для создания.


## О [Лин Кларк](http://code-cartoons.com/)
Лин работает инженером в команде Mozilla Developer Relations. Она занимается JavaScript, WebAssembly, Rust и Servo, а также рисует комиксы про то, как работает наш код.

---

*Слушайте наш подкаст в [iTunes](https://itunes.apple.com/ru/podcast/девшахта/id1226773343) и [SoundCloud](https://soundcloud.com/devschacht), читайте нас на [Medium](https://medium.com/devschacht), контрибьютьте на [GitHub](https://github.com/devSchacht), общайтесь в [группе Telegram](https://t.me/devSchacht), следите в [Twitter](https://twitter.com/DevSchacht) и [канале Telegram](https://t.me/devSchachtChannel), рекомендуйте в [VK](https://vk.com/devschacht) и [Facebook](https://www.facebook.com/devSchacht).*

[Статья на Medium](https://medium.com/devschacht/avoiding-race-conditions-in-sharedarraybuffers-with-atomics-8de5323aad63)