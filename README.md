<h1>Русская дока про Floxim</h1>

<h2>1. Произношение навания по-русски</h2>

Флóксим - ударение на первый слог. Слово мужского рода. Правильно "я возился с Флóксимом". Неправильно: "возилися с Флóксимой", "возился с Флокси́мом" и т.д.

<h2>2. Устройство репозиториев</h2>

https://github.com/Floxim/site-phototeam - демо-сайт. Создать проект в папке "phototest.loc" можно вот так:
<pre>
composer create-project floxim/site-phototeam phototest.loc -s dev
</pre>
Далее открываем в браузере и указываем параметры БД и проч. в инсталлере.
<br />
<br />
https://github.com/Floxim/floxim - ядро системы<br/>
https://github.com/Floxim/floxim-cache - кэшер
<br />
<br />

Эти два репозитория попадают в /vendor/floxim/
<br />
<br />
https://github.com/Floxim/module-smth - модули, попадают в /module/Floxim/Smth<br /> 
https://github.com/Floxim/module-main - "главный модуль", объявляющий базовые контроллеры, логику работы контентных типов данных (т.е. объектов, составляющий контент сайта) и т.д.

<br />
<br />
Модули и темы можно цеплять Композером, указав тип и подцепив плагин-установщик: https://github.com/Floxim/composer
<br />
<br />
Еще есть приватные репозитории под моим аккаунтом, там лежит первый набросок сааса.


<h2>3. Процесс обработки запроса страницы</h2>

<ul>
 <li> все запросы приходят на /index.php
 <li> он подключает /boot.php, где происходит подключение автолодера и загрузка конфига из /config.php
 <li> для всех модулей пытаемся дернуть метод init() - там можно навешать обработчики событий, зарегистрировать кастомные роутеры и т.д.
 <li> далее в index.php запрос пропускается через цепочку роутеров, пока какой-нибудь не вернет результат
 <li> результат выводится на экран.
</ul>

<h2> 4. Роутеры</h2>
 
Из коробки сейчас есть такие роутеры:

<ul>
 <li> Admin - обрабатывает запросы на /floxim/, возвращает JSON для отрисовки админский форм и контролов
 <li> Infoblock - обрабатывает запросы на /~ib/, используется для перерисовки блоков при редактировании / настройке
 <li> Ajax - обрабатывает запросы на /~ajax/, позволяет получать результат работы отдельного контроллера / инфоблока, отличается от Infoblock-роутера тем, что не умеет перерисовывать блок с переопределенными параметрами, соответственно может использоваться для обычных пользователей
 <li> Front - основной роутер, показывающий страницы сайта
 <li> Error - последний в цепочке, показывает страницу /404/, если ничего не нашлось.
</ul>

<h2>5. Роутер Front</h2>
 
Front-роутер работает следующим образом:

<ul>
 <li> по URL получает путь к текущей странице
 <li> для этого у всех зарегистрированных роутеров опрашивается метод getPath($url, $site_id), по умолчанию срабатывает метод самого Front-роутера, который находит страницу по URL и возвращает массив с ней и всеми ее предками
 <li> на основе пути находится инфоблок-лейаут для текущей страницы и выполняется его рендеринг - запускается контроллер \Floxim\Floxim\Controller\Layout, он получает инфоблоки, которые надо показать на странице, далее результат пропускается через шаблон-тему
 <li> тема содержит области (area, например "шапка", "сайдбар", "контент"), которым назначены инфоблоки, в процессе отрисовки лейаута области заполняются инфоблоками, каждый из которых тоже рендерится
</ul>

<h2>6. Устройство инфоблока</h2>

<p>
Инфоблок хранит информацию о том, что на таких-то страницах сайта в таком-то месте нужно показать результат работы такого-то контроллера, отрисованный таким-то шаблоном. То есть это запись в БД, с такими полями (таблица fx_infoblock):

<ul>
 <li> название блока (например "Главное меню")
 <li> сайт (Floxim - мультисайтовая система)
 <li> контроллер-экшн, который нужно дернуть для получения данных (например, контроллер floxim.nav.section, экшн list_infoblock - соответствует методу \Floxim\Nav\Section\Controller::doListInfoblock )
 <li> параметры экшна (задаются в настройках блока через UI, например сколько объектов показывать, как сортировать и т.д.)
 <li> scope - "область видимости" - правило, описывающее набор страниц, на которых отображается блок (например: Все страницы типа "Новость", вложенные в раздел "О компании")
 
 </ul>
 
 
У инфоблока есть "визуальная часть", вынесенная в таблицу fx_infoblock_visual - для того, чтобы один блок можно было показывать разными шаблонами в зависимости от текущей выбранной шкуры сайта. Там есть вот что:


<ul>
 <li> "шаблон-обертка" (wrapper) - результат отрисовки блока можно "обернуть" в другой шаблон, для единообразного оформления блоков
 <li> параметры обертки (wrapper_visual, json-данные)
 <li> шаблон для рендеринга данных, пришедших из контроллера (template)
 <li> параметры шаблона (template_visual, json-данные, которые добавляются к результату работы контроллера перед рендерингом шаблона)
 <li> код области, где должен располагаться блок (area)
 <li> позиция блока (priority)
 
 </ul>
 
Таким образом, каждый блок рендерится в рамках своего микро-MVC, и этот процесс конфигурируется при помощи механизма инфоблоков.

<h2>7. Модель данных</h2>

В отличие от многих фреймворков, в модели Floxim явно выделяется два класса:

<ul>
 <li> Finder - класс, отвечающий за выборки и непосредственно работу с БД, используется для построения запросов (queryBuilder)
 <li> Entity - класс, представляющий экземпляр модели
</ul>


Все "системные" модели живут в папке /vendor/Floxim/Floxim/Component/Smth 
<br /><br /> 
Например, для модели infoblock (т.е. табличка с инфоблоками) Finder - класс \Floxim\Floxim\Component\Infoblock\Finder, а Entity - \Floxim\Floxim\Component\Infoblock\Entity.
<br /><br />
Получить все инфоблоки для сайта #18 можно вот так:

<pre>
$finder = new \Floxim\Floxim\Component\Infoblock\Finder();
$blocks = $finder->where('site_id', 18)->all();
</pre>

Чтобы не писать это все, есть метод-хелпер fx::data(). Можно написать и так:

<pre>
$blocks = fx::data('infoblock')->where('site_id', 18)->all();
</pre>

Каждый полученный блок - экземпляр \Floxim\Floxim\Component\Infoblock\Entity, который предоставляет доступ к свойствам через ArrayAccess и методы для сохранения/удаления. Можно сделать, например, вот это:

<pre>
foreach ($blocks as $block) {
    $block['site_id'] = 19;
    $block->save();
}
</pre>

<h2>8. Коллекции</h2>

Почти везде вместо массивов используются "коллекции" (экземпляры \Floxim\Floxim\System\Collection). Это обертка над массивом, нужная для двух вещей.

<br /><br />

Добавить вкусностей, типа такого:

<pre>
$blocks = fx::data('infoblock')->all();

$blocks
	// найти все блоки, у которых action = 'list_infoblock'
	->find('action', 'list_infoblock')
    // сгруппировать по свойству controller (вернет коллекцию коллекций)
    ->group('controller')
    // для каждой группы применить коллбэк
    ->apply(
    	function($group, $key) {
    		echo 'Com '.$key.' has '.count($group).' ibs: '.join(", ", $group->getValues('id'))."<br />";
		}
	);
</pre>

  Коллекции являются объектами, соответственно никогда не копируются при передаче.

  
<h2>9. Компоненты</h2>

Компонент - это тип данных (таблица в БД), соответствующие классы моделей, класс-контроллер и набор шаблонов. На данный момент следуюет отличать "пользовательские" компоненты (например "Блог") от системных моделей (например, тот же Infoblock).
<br /><br />
Компоненты можно создавать через UI (вкдалка "Разработка - Компоненты"), через админку можно добавлять/изменять/удалять поля. При создании компонента генерируются нужные классы модели и контроллера, которые далее можно заполнить своими методами.
<br /><br />
Компонент должен быть привязан к модулю, а модуль - к вендору. Ключевое слово компонента имеет вид "вендор.модуль.код", например "floxim.nav.section" (компонент "Раздел" модуля "Nav" вендора "Floxim"). Такая запись используется для получения данных, именования шаблонов и т.д. По умолчанию используется вендор "My", его можно переопределить, добавив в /config.php параметр 'dev.vendor' => 'yourname' (можно несколько через запятую).
<br /><br />
Компоненты во Floxim наследуются, то есть дочерний компонент имеет все поля, которые есть у родителя. Реализовано это через общие записи ID в нескольких таблицах (например, есть "Новость" наследуется от "Страницы", каждой записи из таблицы "Новости" соответствует запись из таблицы "Страницы" с тем же идентификатором).  При этом классы моделей/контроллеров также повторяют цепочку наследования компонента.
<br /><br />
Сейчас все компоненты должны наследоваться от компонента floxim.main.content - в нем определены системные поля: тип записи, кто и когда создал запись, к какому инфоблоку она относится, состояние публикации, поля для хранения иерархии (parent_id, level, materialized_path). 
<br /><br />
Поля компонентов описываются в табличке fx_field. Данные, характерные для конкретного типа поля, заказываются в json в поле format. 
<br /><br />
Есть планы по объединению "системных" и "пользовательских" компонентов в один механизм, примерно так:

<ul>
 <li> сделать возможность заводить компоненты, не являющиеся наследниками floxim.main.content (то есть образующие собственное пространство ID) - для всяких служебно-словарных данных, которым не нужно положение в иерархии сайта
 <li> сделать, чтобы информация о полях "системных" компонентов также хранилась в fx_fields и была возможность добавлять кастомные поля к системным объектам.
 </ul>
 
 
<h2>10. Контроллеры и экшны</h2>


Экшн - метод контроллера с префиксом "do". Имя метода (как любых прочих методов) пишется в camelCase, но при нотации в шаблонах / кратких вызовах и т.д. может использоваться и under_score. Инициализировать и выполнить контроллер можно вот так (но не нужно, т.к. обычно такой вызов происходит глубоко внутри инфоблока):

<pre>
$res = fx::controller(
            // контроллер компонента News из модуля Blog от вендора Floxim
            // метод doListFiltered
            'floxim.blog.news:list_filtered',
            // массив параметров, например "ограничиться 5-ю результатами"
            array('limit' => 5)
       )
       // выполнить и вернуть результат
       ->process();
</pre>


В классе-контроллере компонента floxim.main.content объявляются методы-экшны, реализующие ряд стандартных задач. Это три "списочных" экшна и экшн для отрисовки отдельной страницы.

<ul>
 <li> list_infoblock - выводит записи, привязанные к текущему инфоблоку. В инфоблок с таким контроллер-экшном можно добавлять данные.
 <li> list_filtered - выводит записи, отфильтрованные по набору условий (может быть не задан) - годится для всяких "Последние новости", "Свежие фото", а тажке для дублирования навигации
 <li> list_selected - выводит записи, выбранные вручную.
 <li> record - выводит запись, соответствующую текущей странице.
 </ul>
 
 
Влиять на поведение метода можно через систему событий (ну, кроме полного переопределения). В процессе выполнения базовый контроллер генерирует события:

<ul>
 <li> запрос готов (можно пойамать и добавить к запросу дополнительные фильтры)
 <li> объекты готовые (можно обойти объекты и как-то их модифицировать, отфильтровать или дополнить)
 <li> результат готов (общее событие для всех экшнов).
 </ul>
 
Из компонента-наследника эти события можно обрабатывать так:


<pre>
public function doListInfoblock() {
    $this->onQueryReady(function($e) {
        // получаем finder-запрос
        $e['query']
            // очищаем все сортировки
            ->order(null)
            // добавляем правильную сортировку
            ->order('cool_field', 'desc')
            // и лимит
            ->limit(5);
    });
    $this->onItemsReady(function($e) {
        // коллекция с найденными объектами
        $e['items']
            // найти и удалить объекты, для которых колбэк вернет true
            ->findRemove(function($i) {
                return $i->isNotVeryCool();
            })
            // пересортировать по id 
            ->sort('id');
    });
    return parent::doListInfoblock();
}

</pre>
 
Все "списочные" экшны проходят через общий метод doList, соответственно, если надо поменять поведение всех таких экшнов, можно переопределить doList и навесить события в нем.
<br /><br />
Еще есть экшны для отрисовки форм добавления/редактирования, но они пока не вполне production-ready :)
<br /><br />
Результат работы контроллера собирается из данных, которые вернул метод-экшн и данных, добавленных через $this->assign('key', $data).

<h2>10. Шаблоны</h2>

<h2>11. События</h2>

<h2>12. Полезные инструменты</h2>

Для отладки можно использовать логгер - в любом месте кода пишем:
<pre>
 fx::log($what, $you, $want);
 </pre>

 
И ищем результат во вкладке "Development - Logs". Можно вывести объекты прямо на экран методом fx::debug($a1, $a2);
<br /><br />
Дебаггер делает print_r($data) и затем парсит результат, превращая в кликабельное дерево. Поэтому для больших сильно залинкованных структур данных оно может работать очень долго, сжирать всю память и т.д. Когда-нибудь мы допилим его, чтобы он возвращал json со специальными пометками вместо ссылок.

<br /><br />
Еще есть консоль, где можно вбить кусок кода на PHP и посмотреть, чтоб получится. Искать - "Development - Console". Например, можно быстро найти и посмотреть все разделы сайта, скопипастив в консоль вот такой код:

<pre>
&lt;?php
$pages = fx::data('floxim.nav.section')
            // получить все разделы
            ->all()
            // выдрать значения
            ->getValues( 
                // полей url и name
                array('url', 'name'), 
                // использовать id как ключ
                'id'
             );

fx::debug($pages);
</pre>

Покажет примерно это: http://joxi.ru/BA0v0G6FbOvzmy
<br /><br />
В шапке - тайминг и последний элемент трейса. Развернуть всю ветку можно через Ctrl+click.
<br /><br />
Консоль хорошо подходит для всяких служебных операций и экспериментов.
