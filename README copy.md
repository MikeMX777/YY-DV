          :--     :-:                                           .---.        .:--:.    :-:.    :-:                        :-:                           
          :#%=   =%*                                          .**+-+*+.     :**==**:   .#%+   -%%.                        #%*                           
           :%%= =%#. .:=-.   .:..  .:. .:..:--.    .--..:.    -**. -**.     **=  **-    .#%*.:%#. .-=-:.  .:..:-:.   .:=:.#%*   .-=:.  .:.  ..:.        
            .%%*%#..*%#=+%%= -%%:  #%+ -%%%**%%= :%%*=*%%#     +**=**:      .**++*=.     .#%*%#..##+=*%%: +%%%+#%%: -%%+=#%%* .#%*=#%* :#%*.+%*.        
             :%%*. +%#.  -%%.-%%:  #%+ -%%: .#%+.*%*   #%#   .-****=. :*-  .+****:  =*    .%%%:   ..::%%= +%%   %%-.%%-   #%*.*%*...%%:  *%%%+          
             .*%+..+%*.  :%%:-%%:  #%+ -%%: .#%+.#%+   #%#   =**..=**+**: :**- :***=*=     #%%  .#%#--%%= +%%   %%--%%:   *%*.#%%%%%%%-  =%%%:          
             .*%+. -%%- .*%#.-%%:.:#%+ -%%: .#%+ =%%=:-%%#   -**:  :***-. .**=.  =***:     #%%  -%%. .%%= +%%   %%- #%*. :%%* =%%:  .:..+%*:%%=.        
             .*%+.  .#%%%%+. .*%%%%#%+ -%#: .#%+  :#%%*%%*    :******=+**- .=*****+-***.   #%#  .*%%%#%%= +%#   %%- .*%%%*+%*  -#%%%%%:+%*. :#%+        
                                                 :#*==*%%-                                                                                              
                                                  .-===:.                                                                                               

Привет ребята, вот моя сопроводительная записка:

1) качаем бинарник
2) Запускаем просто и с ключем -h, help есть - отлично!
3) Видим что при ./bingo print_current_config не находит конфиг, хорошо, тогда используем strace и смотрим где ищет конф:
strace ./bingo print_current_config
------///
openat(AT_FDCWD, "/opt/bingo/config.yaml", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
------///
4) кладем свой конф, пример есть в help
5) Поковырял autocompl ничего нового там не нашел( А было бы прикольно пасхалку 
6) Запускаем БД через docker-compose
7) наполняем БД через ./bingo prepare_db
8) Не запускается run_server, смотрим strace^
openat(AT_FDCWD, "/opt/bongo/logs/516abe8e27/main.log", O_WRONLY|O_CREAT|O_APPEND|O_CLOEXEC, 0666) = -1 EACCES (Permission denied)
9) Создаем папки с правами, лог файл
10) Сервер поднялся:
   yoohoo_server_launched
11) Слушает не на 80 порту, а на 14822, видим из netstat
12) ping -> pong)
13) Собираем image на distroless, трабла, что там нет curl, wget для healthcheck. Копируем бинарник wget из busybox (Dockerfile тут - \ansible\files\bingo\image)
14) Проблема, что dockerd сам не следит за healthcheck контейнера, поэтому нужен костыль, я нашел целых 3:
	1) Отслеживам состояние контейнеров по крону каждую минуту, и отправляем в перезапуск (минута - очень много)
	2) В самом HEALHCHECK при unheathy прописываем kill процесса в контейнере, тогда контейнер перезагрузится (в distroless нет kill, можно было пересобрать, но время поджимало)
	3) Использовать сторонний сервис willfarrell/autoheal, он быстро видит что сервис болеет и грохает контейнер (удобно, но небезопасно он цепляется к docker.sock)
	Остановился на 3м варианте.
15) Сервис генерит много логов -> настраиваем ротацию логов каждый час
16) Переходим на развертывание. У меня все сделано на обычных виртуалках без облака. Почему не стал лезть в облако - много народу, банят в docker hub, еще какие то проблемы от наплыва людей.  
Были свободные виртуалки их и использовал, тем более рядом локальный GITLAB - в него и заливал всё. Деплой прописан на ansible, можно запускать тупо через ansible-playbook, но тк у меня gitlab - запускаю через него. Через Run pipeline - > ACTION и название Тэга из плэйбука(Можно сразу все задеплоить через тэг deploy, можно по частям)
################################################################################################################################################
Postgres
################################################################################################################################################
1) БД Postgres об этом намекало имя пользователя в конфиге
2) Все сервисы разворачиваются через docker-compose 
3) Базу на проде решено было не наполнять из bingo(долго), а при ините восстановить дамп.
4) В БД не было индексов -> из-за этого медленно отдавал запросы. Особенно спец составленные для тяжести):
SELECT sessions.id, sessions.start_time, customers.id, customers.name, customers.surname, customers.birthday, customers.email, movies.id, movies.name, movies.year, movies.duration FROM sessions INNER JOIN customers ON sessions.customer_id = customers.id INNER JOIN movies ON sessions.movie_id = movies.id WHERE sessions.id IN 25 ORDER BY movies.year DESC, movies.name ASC, customers.id, sessions.id DESC LIMIT 100000
################################################################################################################################################
BINGO
################################################################################################################################################
1) Собирать или нет image для bingo? - конечно собирать) Это удобно и безопаснее, чем запускать локально. Особенно если настроен userns-remap
2) Образ на Distroless c вкаченным wget, залил на локальный registry, ну их эти докерхабы
3) logrotate.conf и crontab файлы для ротации логов (логи замаплены, ротация на хостовой машине) В кроне на разные ноды bingo разное время запуска({{ 59 | random }}), чтоб не аффектить сервис
################################################################################################################################################
NGINX
################################################################################################################################################
1) NGINX на unpreveledged образе, для безопасности)
2) Эксортер настроил, но он отдает стандартный stub_status. По эндпоинтам с простым сложновато сделать. С ходу в голову приходит обрабатывать логи и по ним статистику писать.
3) HTTP3 пытался, но не успел
4) /long_dummy закэширован - видно в тестах
5) Доступ для Пети открыл по ip подсетям в nginx
------------------------------------------------------------------------------------------------------------------------------------------------
Если GET сделать в корень - выйдет еще CODE - index_page_is_awesome
Все страдания вырезаны из письма, а подумать было над чем. Почему это не работает или то. Делал за 2 дня, поэтому так сумбурно все.

Хочу сказать спасибо всем кто причастен к этой тренировке, итоговый проект - это просто огонь, как квест проходишь. Удачи всем! Пишите, звоните. А, хотелось бы брелок))

Михаил
markov-1810@ya.ru