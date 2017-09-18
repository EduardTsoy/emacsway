
Реализация паттерна Repository в браузерном JavaScript
======================================================

.. post::
   :language: en
   :tags: Repository, ORM, JavaScript
   :category:
   :author: Ivan Zakrevsky

.. 06 Aug, 2017

Хорошая архитектура освобождает Вас от привязки к конкретной реализации.
Она позволяет Вам отложить момент принятия решения о реализации, и `начать конструирование кода еще не имея этого решения <Service Stub_>`__.
Принципиально важным моментом является то, что Вы обретаете возможность принять решение в момент наибольшей информированности, а также всегда можете легко подменить конкретную реализацию на любую другую.
Вот эта обязанность возложена на паттерн `Repository`_.


.. contents:: Содержание


Таким образом, у Вас появляется полная абстракция от источника данных, будь то REST-API, MBaaS, SaaS, IndexedDB, HTML, сторонний сервис по протоколу JSON-RPC или `Service Stub`_.

    "We often forget that it is also best to postpone decisions until the last possible moment.
    This isn’t lazy or irresponsible; it lets us make informed choices with the best possible information.
    A premature decision is a decision made with suboptimal knowledge. We will have that
    much less customer feedback, mental reflection on the project, and experience with our
    implementation choices if we decide too soon."
    («Clean Code: A Handbook of Agile Software Craftsmanship» [#fnccode]_)

..

    A good architecture allows major decision to be deferred! (Robert Martin)

.. A good architecture allows you to defer critical decisions, it doesn’t force you to defer them. However, if you can defer them, it means you have lots of flexibility.
   («Clean Architecture» [#fnca]_)

..

    You would make big decisions as
    late in the process as possible, to defer the cost of making the decisions and to have
    the greatest possible chance that they would be right. You would only implement
    what you had to, in hopes that the needs you anticipate for tomorrow wouldn't come
    true.
    (Kent Beck [#fnxp]_)

Кроме того, у Вас появляется возможность реализовать паттерны `Identity Map`_ и `Unit of Work`_.
Последний очень часто востребован, так как позволяет сохранять на сервере только измененные объекты окончательно сформированного агрегата вложенных объектов, либо выполнить откат состояния локальных объектов в случае, если сохранить данные невозможно (пользователь передумал или ввел невалидные данные).


Модель предметной области (Domain Model)
========================================

Наибольшим преимуществом полноценных `Моделей предметной области <Domain Model_>`__ в программе является возможность использования принципов Domain-Driven Design (DDD) [#fnddd]_.
Если Модели содержат исключительно бизнес-логику, и освобождены от служебной логики, то они могут легко читаться специалистами предметной области (т.е. представителем заказчика).
Это освобождает Вас от необходимости создания UML-диаграмм для обсуждений и позволяет добиться максимально выского уровня взаимопонимания, продуктивности, и качества реализации моделей.

В одном из проектов я пытался реализовать достаточно сложную доменную логику (которая содержала около 20 доменных Моделей) в парадигме реактивного программирования, когда атрибуты экземпляра Модели, содержащие аннотации агрегации или зависимые от них, изменяли свое значение путем реакции на изменения других моделей и хранилищ.
Суть в том, что вся эта реактивная логика уже не принадлежала самой доменной модели, и располагалась в разного рода `слушателях <Observer_>`_ и обработчиках.

    "The whole point of objects is that they are a technique to package data with the processes used
    on that data. A classic smell is a method that seems more interested in a class other than the one
    it actually is in. The most common focus of the envy is the data."
    («Refactoring: Improving the Design of Existing Code» [#fnrefactoring]_)   

..

    "If the framework's partitioning conventions pull apart the elements implementing the
    conceptual objects, the code no longer reveals the model.

    There is only so much partitioning a mind can stitch back together, and if the framework uses 
    it all up, the domain developers lose their ability to chunk the model into meaningful pieces."
    («Domain-Driven Design: Tackling Complexity in the Heart of Software» [#fnddd]_)

Это привело к такому огромному количеству хитросплетений слушателей, что превосходство в performance было утрачено, но еще раньше была утрачена читаемость кода.
Даже я не мог на следующий день сказать что делает тот или иной фрагмент кода, не говоря уже о специалисте предметной области.
Мало того, что это в корне разрушало принципы Domain-Driven Design, так это еще и в значительной мере :doc:`снижало скорость разработки новых функций проекта <../en/how-to-quickly-develop-high-quality-code>`.

Надежды на такой подход окончательно рухнули когда выяснилось, что каждый экземпляр модели должен изменять значения своих атрибутов, содержащих аннотации агрегации или зависимых от них, в зависимости от контекста использования (выбранной группировки отображения или критериев фильтрации).

Впоследствии модели вернули себе свои концептуальные контуры и читаемость кода, и вместе с тем сохранили механизм реакций при добавлении, изменении или удалении объектов.
Для достижения этого результата пришлось своими силами создать библиотеку реализующую паттерн Repository, так как существующих решений для реляционных данных с качественной кодовой базой я не смог найти.


Парадигма реактивного программирования
======================================

Сегодня модно увлекаться реактивным программированием.
Знаете ли Вы, что разработчики dojo впервые `применили реактивное программирование <https://github.com/dojo/dojo/commit/4bd91a5939d4dbc8a43d673cc279bb3d39ed0895#diff-48ec1f2998cbe6d644df0c9abd32d9d0R35>`__ в своей реализации паттерна Repository еще 13 сентября 2010?

Реактивное программирование дополняет (а не противопоставляет) паттерн `Repository`_, о чем красноречиво свидетельствует опыт `dojo.store`_, `Dstore`_ и нового `Dojo 2 - data stores <https://github.com/dojo/stores>`_.

Разработчики dojo - команда высококвалифицированных специалистов, чьи библиотеки используют такие серьезные компании как IBM.
Примером того, насколько серьезно и комплексно они подходят к решению проблем, может служить `история библиотеки RequireJS <http://requirejs.org/docs/history.html>`_.


Примеры реализаций паттерна Repository в JavaScript
===================================================

Примеры простейших реализаций паттерна Repository на JavaScript в проекте `todomvc.com <http://todomvc.com/>`_:

- Angular2: https://github.com/tastejs/todomvc/blob/gh-pages/examples/angular2/app/services/store.ts
- Angular1: https://github.com/tastejs/todomvc/blob/gh-pages/examples/angularjs/js/services/todoStorage.js
- React: https://github.com/tastejs/todomvc/blob/gh-pages/examples/react-alt/js/stores/todoStore.js

Другие реализации:

- `Dojo2 Stores <https://github.com/dojo/stores>`_ - \
  Excellent implementation of `Repository`_ pattern in paradigm of `Reactive Programming`_ for non-relational data.
- `Dstore <http://dstorejs.io/>`_ - \
  yet another excellent implementation of `Repository`_ pattern.
- `Dojo1 Store <https://dojotoolkit.org/reference-guide/1.10/dojo/store.html>`_ - \
  Dojo1 implementation of `Repository`_ pattern.
- `JS-Data <http://www.js-data.io/>`_ - \
  ORM written by JavaScript for relational data. Does not support composite relations.
- `9 JavaScript Libraries for Working with Local Storage <https://www.sitepoint.com/9-javascript-libraries-working-with-local-storage/>`_ - \
  article with interesting comments.
- `Kinvey Data Store <http://devcenter.kinvey.com/angular/guides/datastore>`_ - \
  implementation of `Repository`_ pattern by MBaaS Kinvey, `source code <https://github.com/Kinvey/js-sdk/tree/master/src/datastore/src>`__
- `Pocket.js <https://github.com/vincentracine/pocketjs>`_ - \
  a wrapper for the window.localStorage. It provides helpful methods which utilise MongoDB's proven syntax and provides a powerful lightweight abstraction from the complexity of managing and querying local storage.

Я хотел бы добавить сюда и `Ember.js <https://emberjs.com/>`_, но он реализует паттерн `ActiveRecord`_.


Реализация реляционных связей
=============================


Синхронное программирование
---------------------------

На заре появления ORM, мапперы делали таким образом, чтобы они извлекали из базы данных все связанные объекты одним запросом (см. `пример реализации <https://bitbucket.org/emacsway/openorm/src/default/python/>`_).

Domain-Driven Design подходит к связям более строго, и рассматривает связи с позиции концептуальных контуров агрегата вложенных объектов [#fnddd]_.
Доступ к объекту осуществлялся либо по ссылке (от родительского объекта к вложеному), либо через Repository.
Здесь также особую роль играет направление связей, и соблюдение принципа минимальной достаточности ("дистиляция моделей" [#fnddd]_).

    In real life, there are lots of many-to-many associations, and a great number are naturally
    bidirectional. The same tends to be true of early forms of a model as we brainstorm and explore
    the domain. But these general associations complicate implementation and maintenance.
    Furthermore, they communicate very little about the nature of the relationship.

    There are at least three ways of making associations more tractable.

    1. Imposing a traversal direction
    2. Adding a qualifier, effectively reducing multiplicity
    3. Eliminating nonessential associations

    It is important to constrain relationships as much as possible. A bidirectional association means
    that both objects can be understood only together. When application requirements do not call for
    traversal in both directions, adding a traversal direction reduces interdependence and simplifies
    the design. Understanding the domain may reveal a natural directional bias.
    («Domain-Driven Design: Tackling Complexity in the Heart of Software» [#fnddd]_)

..

    Minimalist design of associations helps simplify traversal and limit the explosion of relationships
    somewhat, but most business domains are so interconnected that we still end up tracing long,
    deep paths through object references. In a way, this tangle reflects the realities of the world,
    which seldom obliges us with sharp boundaries. It is a problem in a software design.
    («Domain-Driven Design: Tackling Complexity in the Heart of Software» [#fnddd]_)

С появлением ORM, в синхронном программировании активно начали применяться ленивые вычисления для разрешения связей.
В Python для этого активно используются `Descriptors <https://docs.python.org/3/howto/descriptor.html>`__, а в Java - AOP и Cross-Cutting Concerns [#fnccode]_.

Ключевым моментом является освобождение Domain Model от логики доступа к источнику данных.
Это необходимо как из принципа чистоты архитектуры и проектных решений, чтобы снизить сопряжение (`Coupling`_), так и из принципа простоты тестирования.
Наибольших успехов позволяет достигнуть принцип Cross-Cutting Concerns, который полностью освобождает модель от служебной логики.

С появлением ОРМ, организация связей стала настолько легкой, что о ней перестали задумываться.
Там где требуются однонаправленные связи, разработчики с легкостью применяют двунаправленные связи.
Появились механизмы оптимизации выборки связанных объектов, которые неявно предзагружают все связанные объекты, что значительно сокращает количество обращений в базу данных.


Избегание связей
----------------

Стоит упомянуть и другую распространенную точку зрения, которая гласит, что объект не должен отвечать за свои связи, а исключительное право на доступ к объекту должно принадлежать только Repository.
Такой точки зрения придерживаются некоторые уважаемые мною друзья.


Асинхронное программирование
----------------------------

Рост популярности асинхронных приложений заставил пересмотреть устоявшиеся представления о ленивой реализации связей.
Асинхронное обращение к каждой ленивой связи каждого объекта значительно усложняет ясность программного кода, и препятствует оптимизации.

Это привело к росту популярности объекто-ориентированных баз данных в асинхронном программировании, которые позволяют сохранять агрегаты целиком.
Все чаще REST-frameworks стали использоваться для передачи клиенту `агрегатов вложенных объектов <http://www.django-rest-framework.org/api-guide/serializers/#dealing-with-nested-objects>`_.

    To do anything with an object, you have to hold a reference to it. How do you get that reference?
    One way is to create the object, as the creation operation will return a reference to the new
    object. A second way is to traverse an association. You start with an object you already know and
    ask it for an associated object. Any object-oriented program is going to do a lot of this, and these
    links give object models much of their expressive power. But you have to get that first object.

    I actually encountered a project once in which the team was attempting, in an enthusiastic
    embrace of MODEL-DRIVEN DESIGN , to do all object access by creation or traversal! Their objects
    resided in an object database, and they reasoned that existing conceptual relationships would
    provide all necessary associations. They needed only to analyze them enough, making their entire
    domain model cohesive. This self-imposed limitation forced them to create just the kind of endless
    tangle that we have been trying to avert over the last few chapters, with careful implementation of
    ENTITIES and application of AGGREGATES . The team members didn't stick with this strategy long, but
    they never replaced it with another coherent approach. They cobbled together ad hoc solutions
    and became less ambitious.

    Few would even think of this approach, much less be tempted by it, because they store most oftheir objects in relational databases.
    This storage technology makes it natural to use the third way
    of getting a reference: Execute a query to find the object in a database based on its attributes, or
    find the constituents of an object and then reconstitute it.
    («Domain-Driven Design: Tackling Complexity in the Heart of Software» [#fnddd]_)

Необходимость обхода агрегатов активизировала интерес к функциональному программированию, особенно в сочетании с парадигмой реактивного программирования.

Однако, решение одной проблемы порождало другую проблему.


Функциональное программирование
-------------------------------

Функциональное программирование сложнее использовать для объектов предметной области, так как его сложнее структурировать логически (особенно при отсутствии поддержки `множественной диспетчеризации <https://ru.wikipedia.org/wiki/%D0%9C%D1%83%D0%BB%D1%8C%D1%82%D0%B8%D0%BC%D0%B5%D1%82%D0%BE%D0%B4>`__), что зачастую приводит к появлению плохо читаемого кода, который выражает не то, "что" он делает, а то, "как" он делает непонятно что.

    If you wanted polymophism in C, you’d have to manage those pointers yourself;
    and that’s hard. If you wanted polymorphism in Lisp you’d have to manage those pointers yourself (pass them in as arguments to some higher level algorithm (which, by the way IS the Strategy pattern.))
    But in an OO language, those pointers are managed for you.
    The language takes care to initialize them, and marshal them, and call all the functions through them.

    ... There really is only one benefit to Polymorphism; but it’s a big one. It is the inversion of source code and run time dependencies.
    («OO vs FP» [#fnoovsop]_)

..

    However, my experience is that the cost of change rises
    more steeply without objects than with objects.
    (Kent Beck [#fnxp]_)

А между тем, неясность намерений и целей автора - это ключевая проблема при чтении чужого кода.

    A six-month study
    conducted by IBM found that maintenance programmers "most
    often said that understanding the original programmer's intent was
    the most difficult problem" (Fjelstad and Hamlen 1979).
    («Code Complete» [#fncodec]_)

Как упоминалось в статье ":doc:`../en/how-to-quickly-develop-high-quality-code`", в процессе конструирования кода разработчик 91% времени читает код, и только 9% времени он вводит символы с клавиатуры.
А это значит, что плохо читаемый код на 91% влияет на темпы разработки.

Также такой подход разрушает все выгоды использования Domain-Driven Design, и разделяет элементы, реализующие концептуальные объекты, которые оказываются физически разделенными, что приводит к появлению кода, который больше не выражает модель.

Все `это способствовало появлению <https://groups.google.com/d/msg/reactjs/jbh50-GJxpg/82CHQKeaG54J>`__ в сообществе ReactJS таких библиотек как:

- `Normalizr <https://github.com/paularmstrong/normalizr>`_ - \
  Normalizes (decomposes) nested JSON according to a schema.
- `Denormalizr <https://github.com/gpbl/denormalizr>`_ - \
  Denormalize data normalized with normalizr.


Лирическое отступление
----------------------

Несмотря на то, что приемы функционального программирования часто используются совместно с парадигмой реактивного программирования, в своей сути эти парадигмы не всегда сочетаемы в каноническом виде в веб-разработке.

Это потому, что реактивное программирование основано на распространении изменений, т.е. подразумевает наличие переменных и присваивания.

    This means that it becomes possible to express static (e.g. arrays) or dynamic (e.g. event emitters) data streams with ease via the employed programming language(s), and that an inferred dependency within the associated execution model exists, which facilitates the automatic propagation of the change involved with data flow.

    For example, in an imperative programming setting, ``a := b + c`` would mean that ``a`` is being assigned the result of ``b + c`` in the instant the expression is evaluated, and later, the values of ``b`` and/or ``c`` can be changed with no effect on the value of ``a``.
    However, in reactive programming, the value of ``a`` is automatically updated whenever the values of ``b`` and/or ``c`` change;
    without the program having to re-execute the sentence ``a := b + c`` to determine the presently assigned value of ``a``.

    ... For example, in an model–view–controller (MVC) architecture, reactive programming can facilitate changes in an underlying model that automatically are reflected in an associated view, and contrarily.
    ("`Reactive programming <https://en.wikipedia.org/wiki/Reactive_programming>`__", wikipedia)

Именно поэтому парадигма реактивного программирования `может сочетаться с различными парадигмами <https://en.wikipedia.org/wiki/Reactive_programming#Approaches>`__, императивной, объектно-ориентированной и функциональной.

Однако, вся суть вопроса заключается в том, что в каноническом виде функциональное программирование не имеет переменных (от слова "переменчивость", изменяемость). т.е. изменяемого состояния:

    A true functional programming language has no assignment operator.
    You cannot change the state of a variable.
    Indeed, the word “variable” is a misnomer in a functional language because you cannot vary them.

    ...The overriding difference between a functional language and a non-functional language is that functional languages don’t have assignment statements.

    ... The point is that a functional language imposes some kind of ceremony or discipline on changes of state. You have to jump through the right hoops in order to do it.

    And so, for the most part, you don’t.
    («OO vs FP» [#fnoovsop]_)

Поэтому, использование подходов функционального программирования не делает программу функциональной до тех пор, пока программа имеет изменяемое состояние, - это просто процедурное программирование.
А если это так, то отказ от Domain-Driven Design просто отнимает превосходства обоих подходов (ни полиморфизма объектно-ориентированного программирования, ни неизменяемости функционального программирования), объединяя все худшее, подобно объектам-гибридам [#fnccode]_, так и не делая программу по настоящему функциональной.

    Hybrids

    This confusion sometimes leads to unfortunate hybrid structures that are half object and
    half data structure. They have functions that do significant things, and they also have either
    public variables or public accessors and mutators that, for all intents and purposes, make
    the private variables public, tempting other external functions to use those variables the
    way a procedural program would use a data structure (this is sometimes called Feature Envy from "Refactoring" [#fnrefactoring]_).
    Such hybrids make it hard to add new functions but also make it hard to add new data
    structures. They are the worst of both worlds. Avoid creating them. They are indicative of a
    muddled design whose authors are unsure of—or worse, ignorant of—whether they need
    protection from functions or types.
    («Clean Code: A Handbook of Agile Software Craftsmanship» [#fnccode]_)

Мне, как емаксоиду, импонирует парадигма функционального программирования, но я должен быть честным - она более требовательна к уровню квалификации разработчика, требует соответствующих навыков, и имеет свою нишу, в которой ее преимущества очевидны.
Я не отношу к этой нише проектирование объектов реального мира.
В своей практике я встречал такие безобразия в парадигме функционального программирования, которые было бы весьма затруднительно воспроизвести используя принципы Domain-Driven Design.
Универсальных инструментов не существует, и при всей симпатии к микроскопу его нельзя использовать в качестве молотка, хотя бы из уважения к нему.

Каноническое функциональное программирование не имеет состояния, и поэтому идеально подходит для распределенных вычислений и обработки потоков данных.

    The benefit of not using assignment statements should be obvious.
    You can’t have concurrent update problems if you never update anything.

    Since functional programming languages do not have assignment statements, programs written in those languages don’t change the state of very many variables.
    Mutation is reserved for very specific sections of the system that can tolerate the high ceremony required.
    Those sections are inherently safe from multiple threads and multiple cores.

    The bottom line is that functional programs are much safer in multiprocessing and multiprocessor environments.
    («OO vs FP» [#fnoovsop]_)

Но значит ли это то, что парадигма объектно-ориентированного программирования противостоит парадигме функционального программирования?

Несмотря на то, что парадигма ООП традиционно считается разновидностью императивной парадигмы, т.е. основанной на состоянии программы, Robert C. Martin делает поразительный вывод - так как объекты предоставляют свой интерфейс, т.е. поведение, и скрывают свое состояние, то они не противоречат парадигме функционального программирования.

    "Objects are not data structures.
    Objects may use data structures; but the manner in which those data structures are used or contained is hidden.
    This is why data fields are private.
    From the outside looking in you cannot see any state.
    All you can see are functions.
    Therefore Objects are about functions not about state."
    («OO vs FP» [#fnoovsop]_)

Поэтому некоторые классические функциональные языки программирвания имеют поддержку ООП:

- `Enhanced Implementation of Emacs Interpreted Objects <https://www.gnu.org/software/emacs/manual/html_mono/eieio.html>`_
- `Common Lisp Object System <https://en.wikipedia.org/wiki/Common_Lisp_Object_System>`_

    Are these two disciplines mutually exclusive?
    Can you have a language that imposes discipline on both assignment and pointers to functions?
    Of course you can.
    These two things don’t have anything to do with each other.
    And that means that OO and FP are not mutually exclusive at all.
    It means that you can write OO-Functional programs.

    It also means that all the design principles, and design patterns, used by OO programmers can be used by functional programmers if they care to accept the discipline that OO imposes on their pointers to functions.
    («OO vs FP» [#fnoovsop]_)

Эмулировать объекты можно даже в функциональных языках программирования с помощью замыканий, см. статью "`Function As Object <https://martinfowler.com/bliki/FunctionAsObject.html>`_" by Martin Fowler.
Тут нельзя обойти вниманием замечательную книгу "`Functional Programming for the Object-Oriented Programmer <https://leanpub.com/fp-oo>`_" by Brian Marick.

Давайте вспомним главу "Chapter 6. Working Classes: 6.1. Class Foundations: Abstract Data Types (ADTs): Handling Multiple Instances of Data with ADTs in Non-Object-Oriented Environments" книги «Code Complete» [#fncodec]_.

    An abstract data type is a collection of data and operations that work on that data.
    («Code Complete» [#fncodec]_)

..

    Abstract data types form the foundation for the concept of classes.
    («Code Complete» [#fncodec]_)

..

    Thinking about ADTs first and classes second is an example of programming into a language vs. programming in one.
    («Code Complete» [#fncodec]_)

Я не буду переписывать сюда достоинства АТД, их можно прочитать в указанной главе этой книги.

Но ведь изначально вопрос состоял в том, стоит ли отказываться от АТД в объектно-ориентированном языке при проектировании объектов предметной области в пользу "`Anemic Domain Model`_", и стоит ли приносить в жертву все выгоды Domain-Driven Design в угоду удобства конкретной реализации обработки связей?

    The bottom, bottom line here is simply this.
    OO programming is good, when you know what it is.
    Functional programming is good when you know what it is.
    And functional OO programming is also good once you know what it is.
    («OO vs FP» [#fnoovsop]_)

Также стоит отметить, что далеко не все виды связей вписываются в концепцию агрегата.
Если объект логически не принадлежит агрегату, то мы не можем вкладывать его в агрегат ради удобства разрешения связей, ибо в таком случае у нас интерфейс будет следовать за реализацией что в корне разрушает фундаментальный принцип абстракции.
Также агрегат не совместим со связями типа Many-To-Many и перекрестными иерархиями связей.


Реализация связей путем присваивания
------------------------------------

Принцип физического присваивания связанных объектов `реализован так же и в библиотеке js-data <http://www.js-data.io/v3.0/docs/relations#section-eagerly-loading-relations>`__.

В нашей библиотеке мы предусмотрели как возможность декомпозиции агрегатов вложенных объектов, так и возможность их композиции из плоских данных в Repositories.
Причем, агрегат всегда сохраняет актуальное состояние, и при добавлении, изменении, удалении объекта в Repository, изменения автоматически отображаются в структурах соответствующих агрегатов.
Библиотека реализует это поведение как в парадигме Реактивного программирования, так и в парадигме Событийно-ориентированного программирования (на выбор).

Существует также возможность формировать двусторонние связи.
Но, несмотря на то, что современные интерпретаторы легко чистят мусор с кольцевыми ссылками, с концептуальной точки зрения лучше когда вложенные объекты не осведомлены о своем родителе, если на то нет веских оснований.

Таким образом, для реализации связей объекту совершенно не требуется никакая служебная логика доступа к данным, что поддерживает нулевое сопряжение (`Coupling`_) и образует кристально чистые доменные модели.
Это значит, что доменные модели могут быть инстанцией "класса" Object.

Я также уважительно отношусь к той точке зрения, что доменная модель не должна отвечать за связи.
Поэтому предусмотрена возможность легкого доступа к любому объекту через его Repository.


Исходный код
============

На данный момент исходный код библиотеки пока еще не раскрыт.
Но такая вероятность существует в обозримом будущем.


.. rubric:: Footnotes

.. [#fnccode] «`Clean Code: A Handbook of Agile Software Craftsmanship`_» by `Robert C. Martin`_
.. [#fncodec] «`Code Complete`_» Steve McConnell
.. [#fnpoeaa] «`Patterns of Enterprise Application Architecture`_» by `Martin Fowler`_, David Rice, Matthew Foemmel, Edward Hieatt, Robert Mee, Randy Stafford
.. [#fnddd] «Domain-Driven Design: Tackling Complexity in the Heart of Software» by Eric Evans
.. [#fngof] «Design Patterns Elements of Reusable Object-Oriented Software» by Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides, 1994
.. [#fnrefactoring] «`Refactoring: Improving the Design of Existing Code`_» by `Martin Fowler`_, Kent Beck, John Brant, William Opdyke, Don Roberts
.. [#fnoovsop] «`OO vs FP`_» by Robert C. Martin
.. [#fnca] «`Clean Architecture`_» by Robert C. Martin
.. [#fntca] «`The Clean Architecture`_» by Robert C. Martin
.. [#fnxp] «`Extreme Programming Explained`_» by Kent Beck


.. update:: 04 Sep, 2017


.. _Clean Code\: A Handbook of Agile Software Craftsmanship: http://www.informit.com/store/clean-code-a-handbook-of-agile-software-craftsmanship-9780132350884
.. _Code Complete: http://www.informit.com/store/code-complete-9780735619678
.. _Robert C. Martin: http://informit.com/martinseries
.. _Patterns of Enterprise Application Architecture: https://www.martinfowler.com/books/eaa.html
.. _Refactoring\: Improving the Design of Existing Code: https://martinfowler.com/books/refactoring.html
.. _Martin Fowler: https://martinfowler.com/aboutMe.html
.. _Extreme Programming Explained: http://www.informit.com/store/extreme-programming-explained-embrace-change-9780321278654
.. _OO vs FP: http://blog.cleancoder.com/uncle-bob/2014/11/24/FPvsOO.html
.. _Clean Architecture: https://8thlight.com/blog/uncle-bob/2011/11/22/Clean-Architecture.html
.. _The Clean Architecture: https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html

.. _ActiveRecord: http://www.martinfowler.com/eaaCatalog/activeRecord.html
.. _Domain Model: http://martinfowler.com/eaaCatalog/domainModel.html
.. _Identity Map: http://martinfowler.com/eaaCatalog/identityMap.html
.. _Query Object: http://martinfowler.com/eaaCatalog/queryObject.html
.. _Repository: http://martinfowler.com/eaaCatalog/repository.html
.. _Service Stub: http://martinfowler.com/eaaCatalog/serviceStub.html
.. _Unit of Work: http://martinfowler.com/eaaCatalog/unitOfWork.html
.. _Anemic Domain Model: http://www.martinfowler.com/bliki/AnemicDomainModel.html

.. _Coupling: http://wiki.c2.com/?CouplingAndCohesion
.. _Cohesion: http://wiki.c2.com/?CouplingAndCohesion
.. _Observer: https://en.wikipedia.org/wiki/Observer_pattern
.. _Reactive Programming: https://en.wikipedia.org/wiki/Reactive_programming
.. _dojo.store: https://dojotoolkit.org/reference-guide/1.10/dojo/store.html
.. _Dstore: http://dstorejs.io/