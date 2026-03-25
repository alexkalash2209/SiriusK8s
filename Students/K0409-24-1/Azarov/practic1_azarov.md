# Практическая работа 1 Азаров

## Блок 1 - namespaces

Требуется просмотреть namespace-ы текущего shell-процесса. Это делается командой "ls -la /proc/$$/ns/"
![alt text](https://github.com/Dubrovsky18/SiriusK8s/Students/K0409-24-1/images/Screenshot from 2026-03-24 23-50-01.png)

Описание namespace-ов:

mnt (Mount): 4026531832 \
Отвечает за точки монтирования. Процесс видит свой собственный корень файловой системы и смонтированные устройства. Если сделать chroot или запустить в контейнере, то иноды будут другими

pid и pid_for_children: 4026531836 \
Отвечает за изоляцию дерева процессов. Процесс видит только своих «детей» и процессы в своем неймспейсе.
pid_for_children показывает, в каком неймспейсе PID окажутся будущие дочерние процессы (обычно совпадает с pid, если процесс не вызвал unshare(CLONE_NEWPID) без передачи флага).

net (Network): 4026531833 \
Изолирует сетевой стек: интерфейсы, IP-адреса, маршруты, правила iptables. У вас сейчас инод 1833 — это, скорее всего, «глобальный» или «корневой» сетевой неймспейс хоста.

ipc (Inter-Process Communication): 4026531839 \
Изолирует механизмы межпроцессного взаимодействия System V и POSIX message queues. Если бы процессы находились в разных ipc, они не могли бы общаться через разделяемую память (shared memory) или семафоры.

uts (UNIX Time Sharing): 4026531838 \
Изолирует hostname и domainname. Благодаря этому в контейнере можно задать имя web-server-1, не меняя имя хостовой машины.

user (User): 4026531837 \
Изолирует пользователей и группы. Позволяет процессу внутри неймспейса иметь root (UID 0) для операций внутри этого неймспейса, но при этом на хосте этот процесс является обычным непривилегированным пользователем. Это основа безопасности контейнеров (rootless containers).

cgroup: 4026531835 \
Позволяет процессу иметь свой собственный иерархический путь в cgroups. Это нужно для изоляции ограничений ресурсов (CPU, память) для групп процессов.

time и time_for_children: 4026531834 \
Позволяет изолировать системные часы (монотонные, boot-time). Нужен для того, чтобы процессы внутри неймспейса «не знали», как долго работает хост, или чтобы корректно работали паузы/возобновления работы контейнеров.

Просмотрим все типы неймспейсов в системе командой "lsns"
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-05-05.png>)

В целом lsns выступает как некий паспортный стол для неймспейсов, потому что эта команда показывает все активные пространства неймспейсов. 

Структура вывода lsns 

 NS TYPE — тип неймспейса (mnt, net, pid, user, ipc, uts, cgroup, time)\
 NPROCS — количество процессов, использующих этот неймспейс\
 PID — PID одного из процессов в этом неймспейсе (обычно первого попавшегося)\
 USER — владелец процесса\
 COMMAND — команда, которая запустила процесс

Запустим bash в новом PID namespace. Для этого нам понадобится команда sudo unshare --pid --fork --mount-proc /bin/bash

Описание каждой части команды:\
sudo - Запуск от root (нужен для создания некоторых типов неймспейсов)\
unshare - Утилита для запуска программы в новых неймспейсах\
--pid - Создать новый PID namespace (изоляция процессов)\
--fork - Создать новый процесс (fork) перед запуском программы\
--mount-proc - Автоматически примонтировать /proc в новом неймспейсе\
/bin/bash - Какую программу запустить внутри изоляции\
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-12-13.png>)

Убедимся что мы PID 1
echo "Мой PID: $$"
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-13-18.png>)

Просмотрим запущенные процессы командой ps aux
![alt text](<.Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-14-38.png>)

Запущено два процесса всего, потому что мы запустили баш в контейнере. При выходе этот контейнер просто напросто исчезнет.

Выйдем в глобальное окружение командой exit

Создадим изолированное сетевое окружение командой sudo unshare --net /bin/bash
просмотрим список сетевых интерфейсов командой ip link show. Мы увидим, что есть только loopback-интерфейс, и тот неактивен, а значит изоляция работает.
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-18-24.png>)

Выйдем из окружения так же командой exit

## Блок 2 - cgroups

Найдем иерархию cgroup v2: ls /sys/fs/cgroup/

Cgroups - это важнейший механизм для ограничения и учёта ресурсов процессов.
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-24-25.png>)

Создадим свою cgroup и ограничим CPU до 20%:\
sudo mkdir /sys/fs/cgroup/mytest \
echo "20000 100000" | sudo tee /sys/fs/cgroup/mytest/cpu.max
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-25-53.png>)

Запустим нагрузку и поместим ее в cgroup:\
stress-ng --cpu 2 --timeout 30s &
echo $! | sudo tee /sys/fs/cgroup/mytest/cgroup.procs
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-27-08.png>)

Проверим, что лимит применился и посмотрим в top:\
cat /sys/fs/cgroup/mytest/cpu.stat\
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-37-56.png>)
top
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-34-42.png>)
Скриншот лимита (cat /sys/fs/cgroup/mytest/cpu.max):
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-40-42.png>)

Уберем нагрузку, убив процесс: \
sudo kill $(cat /sys/fs/cgroup/mytest/cgroup.procs) \
Также можно просто подождать 30 секунд, по истечении времени процесс умрет сам

## Блок 3 - chroot

создадим минимальный rootfs:\
mkdir -p /tmp/myroot/{bin,lib,lib64,proc,dev}\
Создаёт директорию /tmp/myroot, которая станет "корнем" новой системы\
Внутри создаёт поддиректории:\
 bin — для исполняемых файлов\
 lib и lib64 — для библиотек\
 proc — точка монтирования procfs\
 dev — для устройств
 
 Скопируем баш и его зависимости:\
cp /bin/bash /tmp/myroot/bin/\
cp /bin/ls   /tmp/myroot/bin/\
Скопируем нужные библиотеки:\
ldd /bin/bash\
Создадим нужные директории для библиотек:\
mkdir -p /tmp/myroot/lib/x86_64-linux-gnu\
mkdir -p /tmp/myroot/lib64\
Затем скопируем их все:\
cp /lib/x86_64-linux-gnu/libtinfo.so.6 /tmp/myroot/lib/x86_64-linux-gnu/\
cp /lib/x86_64-linux-gnu/libc.so.6 /tmp/myroot/lib/x86_64-linux-gnu/\
cp /lib64/ld-linux-x86-64.so.2 /tmp/myroot/lib64/

Скопируем все необходимые зависимости одной командой:
for dep in $(ldd /bin/bash | awk '/=>/ {print $3}'); do\
    cp --parents "$dep" /tmp/myroot/\
done\
cp /lib64/ld-linux-x86-64.so.2 /tmp/myroot/lib64/\
скопируем также динамический загрузчик\
Копирует динамический загрузчик (ELF interpreter)\
 Это самая важная библиотека — она запускает все динамически слинкованные программы
Почему отдельно:\
 ldd может не показать загрузчик в выводе (он в первой строке linux-vdso.so.1)\
 Без него программы не запустятся

![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 00-52-28.png>)

Войдем в chroot-окружение : sudo chroot /tmp/myroot /bin/bash\
![alt text](<Students/K0409-24-1/Azarov/images/Screenshot from 2026-03-25 01-04-45.png>)

Итог: Вот как работает контейнер — namespace + cgroup + chroot. Docker это автоматизирует.




