**Тестовое задание**

###### Вольное отступление:
По моему мнению делать контейнеры необходимо не плейбуками ансибла (по крайней мере не в таком формате), но сделать костыль для демонстрации тестового задания вполне допустимо.<br>

###### Лимиты:
```
ansible 2.8.1
python version = 3.7.9 (default, Feb  1 2021, 20:09:54) [Clang 12.0.0 (clang-1200.0.32.29)]
Docker version 20.10.2, build 2291f61
Не требует никаких дополнительных модулей из galaxy или других источников.

```
Для запуска плейбуков на удаленных хостах необходимо корректно заполнить инвентори.<br>
Для работы с локалхостом все плейбуки готовы без дополнительных настроек.<br>
Все файлы с настройками снабжены комментариями.<br>

_builder.yml_<br>
Плейбук не работает без указания inventory файла. Это искусственное ограничение.<br>
Плейбук сделан для создания образа (образов) на основе настроек и темплейтов, указанных в роли builder.<br>
Плейбук рассчитан на запуск на хосте, в котором уже установлен докер.<br>
Плейбук не рассчитан на push образа в репозиторий, но достаточно легко может быть доработан под данную функциональность.<br>
Плейбук, к сожалению, не приведен к такому состоянию, при котором повторный запуск плейбука ничего не изменяет в системе, так как из-за невозможности установить дополнительные модули широко используются модули command и shell.<br><br>
<br>
_runner.yml_<br>
Плейбук не работает без указания inventory файла. Это искусственное ограничение.<br>
Плейбук сделан для запуска образа (образов) на основе настроек, указанных в роли runner.<br>
Плейбук рассчитан на запуск на хосте, на котором уже доступен образ, указанный в настройках плейбука.<br>
Плейбук рассчитан на запуск на хосте, в котором уже установлен докер.<br>
Плейбук не рассчитан на pull образа из репозитория, но достаточно легко может быть доработан под данную функциональность.<br>
Плейбук изначально рассчитан на запуск одной копии контейнера на хосте в своей группе.<br>
Плейбук ожидает группу в инвентори, при отсутствии группы образ поднят не будет, но плейбук завершит работу состоянием SUCCESS. Это сделано для некоторых странных случаев, например чтобы запуском плейбука both.yml можно было сгенерить образ без установки контейнера.<br>
Хотя функционально плейбук может быть запущен на хостах, отличных от локалхоста, но по факту требуются дописать работу с регистри и модифицировать работу с docker network под свою логику.
<br>
_both.yml_<br>
Пример полной инсталляции, продукта, хотя может использоваться с тегами для расширения функционала.

##### Краткое описание
###### Инвентори
`inventory` Директория настроек стенда. В данный момент является излишне сложной, но рассчитана на доработанные роли (работа с регистри, работа с сетями).<br>

`inventories/DEV/inventory`
Сделана заготовка под запуск контейнеров на удаленных хостах. 

#### Теги и работа с плейбуком
##### builder
Вспомогательный тег `build` идентичен запуску плейбука без тегов и сделан для поддержки тега `run_only`<br>
Вспомогательный тег `run_only` сделан для раздельной сборки образов.<br>
<br>
Пример:<br>
Команда<br>
`ansible-playbook -i inventories/DEV/inventory builder.yml -t run_only,sample_2,build`<br>
соберет только образ sample_2.<br>
Можно указывать несколько образов, например `-t run_only,sample_1,sample_2,undefined_image,build`. Образ _undefined_image_ не будет собран, потому как для него нет настроек в роли.

##### runner
Вспомогательный тег `run_only` сделан для раздельной работы с контейнерами.<br>
Тег `network` создает все сети, указанные в настройках контейнеров. При использовании тега `run_only` создает только те сети, которые подключаются к указанным контейнерам.<br>
Тег `stop` по факту не является остановкой контейнера, а является остановкой и удалением контейнеров по имени, но может легко быть переделан под другие признаки, например label. Может использоватся с тегом `run_only`<br>
Тег `start` является созданием, подключением указанных сетей и запуском контейнера с проверкой запуска по строке в логе. Ожидает существующих сетей, созданных тегом `network`. Может использоватся с тегом `run_only`<br>
Тег `restart` является компиляцией тегов `start` и `stop`<br>
<br>
Пример:<br>
Команда<br>
`ansible-playbook -i inventories/DEV/inventory runner.yml -t stop`<br>
удалит все образы с именами sample_1,sample_2,sample_3.<br><br>
Команда<br>
`ansible-playbook -i inventories/DEV/inventory runner.yml -t restart,run_only,sample_3`<br>
удалит все образы с именем sample_3 и поднимет новый образ sample_3.<br><br>
Команда<br>
`ansible-playbook -i inventories/DEV/inventory runner.yml -t network,run_only,sample_3`<br>
не создаст ни одной сети, потому как в настройках sample_3 не указано ни одной сети.
```
runner:
  app_list:
    sample_3:
      image: sample_1 # необязательное, наименование образа, если не указано то берется название элемента
      port: "5002:5000"
      correct_start_line: "Debug mode: on"
```

##### both
Плейбук является компиляцией плейбуков `builder` и `runner`.
Запуск плейбука приведет к созданию на локалхосте двух образов
```
REPOSITORY                                                                 TAG                                                     IMAGE ID       CREATED         SIZE
sample_2                                                                   latest                                                  345feb9af37c   3 hours ago     442MB
sample_1                                                                   latest                                                  2195c8d9591d   3 hours ago     442MB
```
двух сетей
```
NETWORK ID     NAME              DRIVER    SCOPE
1f8e23e38163   test              bridge    local
0266ec59ccc9   test1             bridge    local
```
и трех контейнеров
```
CONTAINER ID   IMAGE             COMMAND           CREATED              STATUS              PORTS                    NAMES
3395c00dab93   sample_1:latest   "python app.py"   About a minute ago   Up About a minute   0.0.0.0:5002->5000/tcp   sample_3
310ebd97bd81   sample_2:latest   "python app.py"   About a minute ago   Up About a minute   0.0.0.0:5001->5000/tcp   sample_2
ca65f0d047b0   sample_1:latest   "python app.py"   About a minute ago   Up About a minute   0.0.0.0:5000->5000/tcp   sample_1
```
Контейнер sample_1 подключен к сетям test и test1, контейнер sample_2 подключен к сети test, контейнер sample_1 не подключен к сетям.<br>
При создании контейнера sample_3 используется образ sample_1:latest<br>
Пример команды запуска плейбука:
```
ansible-playbook -i inventories/DEV/inventory both.yml
```

Команда запуска
```
ansible-playbook -i inventories/DEV/inventory both.yml -t build,run_only,sample_1,sample_3,start
```
приведет к созданию образа sample_1:latest, поднятию двух образов sample_1 и sample_3 из образа sample_1:latest или к падению, если сети test, test1 не существуют или образы с такими названиями уже подняты.
