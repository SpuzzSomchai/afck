# Система Сборки Android Прошивок
*(Android Firmware Construction Kit)*

AFCK это набор скриптов, используемых для (пере-)сборки прошивок ОС Android. Целью данного набора является изготовление тюнингованных прошивок, используя готовую прошивку производителя в качестве основы.

В настоящий момент данные проект может использоваться для сборки прошивок под следующие устройства:

* Android TV приставка X96 Max

Для (пере)сборки прошивки используется сценарий для программы GNU Make и набор POSIX-совместимых утилит. Наберите "make" без параметров чтобы получить описание целей, которые можно собрать сценарием.

# Подготовка к сборке
## Базовые файлы

Файлы, из которых собирается прошивка, не входят в GIT. Они слишком большие, и они часто меняются. Вам придётся найти их в сети самостоятельно. При запуске make Вам сообщат имена отсутствующих файлов.

Можно попробовать запустить файл сценария ingredients/01-get-it-all. Он попытается скачать необходимые файлы из сети. Это не обязательно получится т.к. ссылки на подобные ресурсы могут часто изменяться.

## apktool
Вам необходимо установить свежий apktool отсюда:
	https://bitbucket.org/iBotPeaches/apktool/downloads/

Создайте скрипт apktool, который будет запускать `java -jar apktool-версия.jar $*`. Если Вы раньше устанавливали в apktool файлы фреймворка, очистите его:
```
apktool empty-framework-dir --force
```

# Использование системы сборки

Система сборки имеет встроенную систему помощи. Запуск `make` без параметров выведет полный список целей для сборки. Чтобы инициировать сборку любой цели, запустите `make <цель>`. Чтобы не захламлять основной экран помощи, многие дополнительные цели находятся на отдельных экранах. Для вывода дополнительных экранов помощи, наберите `make help-<экран>`, список имеющихся дополнительных экранов приведён на основном экране помощи, например, `make help-mod` выведет список целей для накладывания модификаций на распакованную прошивку.

Переменная `TARGET` задаёт целевую платформу прошивки, например, `TARGET=x96max/beelink` задаёт сборку прошивки под аппаратную платформу x96max на базе прошивки beelink. Значение переменной TARGET задаётся либо в Makefile верхнего уровня, либо может быть переопределено в файле local-config.mak, который необходимо создать самостоятельно.

Чтобы собрать конечные файлы из исходных, запустите команду `make deploy`. Если всё пройдёт успешно, Вы получите конечные файлы дистрибутива в подкаталоге out/$(TARGET)/deploy/.

Если что-то не получается, можно собирать прошивку шаг за шагом. Сперва попробуйте просто распаковать исходную прошивку: `make img-unpack`. Затем накладываем модификации, либо все сразу командой `make mod`, либо по одной, используя названия целей из `make help-mod`. В конце можно собрать файлы прошивок: `make ubt-img` и `make upd`. После этого остался один маленький шажок: `make deploy` и конечные файлы готовы.


## Разработка собственных модификаций
Все модификации находятся в отдельных каталогах внутри подкаталога целевой платформы `build/$(TARGET)/` (например, build/x96max/beelink/). Каждая модификация находится в своём индивидуальном подкаталоге.

Если Вы разрабатываете свою модификацию, Вам необходимо:

* Создать подкаталог для модификации внутри каталога build/$(TARGET)/. В принципе, название может быть любым, но принято составлять название из двух цифр, тире и названия модификации. Число в начале названия подкаталога помогает отсортировать подкаталоги по порядку обработки, таким образом, модификации с более высоким номером могут пользоваться переменными, заданными модификациями с более низкими номерами.

* Создать в нём файл `название-модификации.mak`, в котором определить необходимые правила наложения модификации. Будем называть содержимое этого файла *правилами модификации*.

Следует учитывать, что все файлы с правилами модификации загружаются в единое пространство имён. Это значит, что если Вы определите переменную с определённым названием в одном файле, а затем переменную с таким же названием в другом файле правил модификации, второе присваивание отменит первое и модификация может произойти не так, как Вам этого хочется. Поэтому, если вводите дополнительные переменные в правила модификации, используйте название модификации как основание для название переменной. Например, если Ваша модификация называется `boom`, используйте названия переменных вида `BOOM_APK`, `BOOM_FILES` и так далее.

Единственное исключение делается для переменных с фиксированными именами, которые задают каким именно образом будет накладываться модификация:

* `HELP` содержит описание модификации. Этот текст выдаётся пользователю по команде `make help-mod`.
* `DISABLED` - если эта переменная получает непустое значение, мод будет выключен. Данную переменную можно использовать для временного отключения определённых модов (без необходимости удалять или переименовывать файл .mak).
* `INSTALL` содержит, собственно, последовательность инструкций по накладыванию модификации на распакованный образ.
* `MOD` содержит название мода
* `DIR` содержит название подкаталога мода. Используйте для доступа к дополнительным файлам, например `$(DIR)/boom.apk` и так далее.
* `DEPS` содержит список файлов, от которых зависит мод. При изменении любого из этих файлов система сборки сочтёт, что мод необходимо переналожить. По умолчанию в переменную `DIR` добавляются все файлы из подкаталога мода (*но не из вложенных подкаталогов*).
* `/` - переменная с таким смешным названием очень полезна, т.к. содержит базовый каталог, в котором находятся распакованные образы файловых систем. В GNU Make переменные с названием из одного символа можно не брать в круглый скобки: `$/` эквивалентно `$(/)`. Используйте эту переменную для удобной работы с файлами распакованного образа, например: `cp $(DIR)/boom.apk $/system/app`

Также для некоторых задач существуют полезные функции, которые можно вызвать:

* `$(call IMG.UNPACK.EXT4,<раздел>)` создаст зависимость текущего мода от распакованного образа указанного раздела. Наприемр, `$(call IMG.UNPACK.EXT4,system)` создаст такую зависимость, чтобы перед началом наложения мода раздел `system` уже был распакован в каталог `$/`, таким образом, Вы сможете сразу писать правила вида `rm $/system/build.prop` и т.п. Если не вызвать этой функции, существование подкаталога `$/system` не гарантируется на момент начала выполнения правил мода, особенно при многопоточной сборке.
* `$(call MOD.APK,<раздел>,<файл.apk>,<описание>)` создаёт полный набор правил для установки APK файла в подкаталог app/ указанного раздела (обычно system или vendor). Простейший мод для добавления определённого приложения может состоять из одной строки вызова этой функции.

В принципе, сказанного выше должно хватать для начала работы. Если же возникнет желание узнать подробнее, как работает систему сборки изнутри, см. следующий раздел.

# Как работает система сборки

Работа системы сборки подробнее описана в файле doc/HOWITWORKS.md. Если что-то пошло не так, или Вы собираетесь разрабатывать на базе afck, рекомендуем этот файл к прочтению.
