# Инструкция для создания дисрибутива linux

## 1 Подготовка хост системы

### 1.1 Проверка системы

Запустить скрипт для проверки версий инструментов разработки и компиляции version-check.sh. Установить недостаующие программы и настоить ссылки

```bash
./version-check.sh
```

### 1.2 Добавить оборудование в виртуальную машину

Через графический интерфейс virt-manager добавить оборудование

- оборудование: "хранилище"
- размер: 40Gb
- тип: virtIO

### 1.3 Подготовка места на диске

Создание таблицы разделов

```bash
sudo fdisk /dev/vdb
```

- Нажмите g (создать таблицу GPT).
- Нажмите n (новый раздел).
- Нажмите Enter (номер раздела 1).
- Нажмите Enter (первый сектор).
- Нажмите Enter (последний сектор).
- Нажмите w (записать и выйти).

Проверить успешность создания таблицы разделов

```bash
lsblk /dev/vdb
```

Создать файловую систему

```bash
sudo mkfs.ext4 /dev/vdb1
```

> Для процедур, выполняемых от пользователей root и lfs необходимо установить переменную среды LFS

```bash
export LFS=/mnt/lfs
echo "LFS variable is set to: $LFS"
```

Создание папки для LFS, здесь будет происходить сборка

```bash
sudo mkdir -pv $LFS
```

> В системах типа ubuntu может понадобиться отредактировать файл /etc/fstab, что бы после презагрузки операционная система понимала какие диски, разделы или сетевые папки нужно подключать (монтировать) автоматически при включении компьютера

Узнать UUID раздела для монтирования

```bash
sudo blkid /dev/sdb1
```

или

```bash
lsblk -dno UUID /dev/vdb1
```

> Добавить в конец файла /etc/fstab строку с UUID из вывода прошлой команды "/dev/disk/by-uuid/[UUID] /mnt/lfs ext4 defaults 0 2"

Монтирование диска

```bash
sudo mount -v -t ext4 /dev/vdb1 $LFS
```

Проверить успешное примонтирование

```bash
df -h
```

Создание структуры папок

```bash
sudo mkdir -v $LFS/sources
sudo mkdir -v $LFS/tools
sudo chmod -v a+wt $LFS/sources
```

### 1.4 Создание пользоваткля lfs

```bash
sudo groupadd lfs
sudo useradd -s /bin/bash -g lfs -m -k /dev/null lfs
```

- -s /bin/bash: устанавливает bash оболочкой по умолчанию
- -g lfs: добавляет пользователя в только что созданную группу
- m: создает домашнюю директорию /home/lfs
- -k /dev/null: предотвращает копирование лишних файлов из /etc/skel

```bash
sudo passwd lfs
```

Передать пользователю lfs права на папки

```bash
sudo chown -v lfs $LFS/tools
sudo chown -v lfs $LFS/sources
```

### 1.5 Настройка окружения пользователя lfs

```bash
su - lfs
```

Добваить в .bash_profile

```bash
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

Добавить в .bashrc

```bash
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF
```

Применение настроек

```bash
source ~/.bash_profile
```

## 2. Скачивание архивов пакетов

Перейти под пользователем lfs

```bash
su - lfs
cd $LFS/sources
```

Получение спиков пакетов

```bash
wget https://www.linuxfromscratch.org/lfs/view/stable/wget-list-sysv
```

Получение файда проверки целостности пакетов

```bash
wget https://www.linuxfromscratch.org/lfs/view/stable/md5sums
```

Скачивание ресурсов, выполнить на хост машине и потом переложить в виртуалку, т.к. могут быть проблемы из за NAT

```bash
wget --input-file=wget-list-sysv --continue --directory-prefix=$LFS/sources
# wget -4 -i wget-list-sysv -c -T 10 -t 5 --no-check-certificate
```

Проверка целостности

```bash
pushd $LFS/sources
  md5sum -c md5sums
popd
```

Перед распаковкой передать права пользователю root

```bash
chown root:root $LFS/sources/*
```

## 3. Подготовка среды выполнения

### 3.1 Создание ограниченной структуры каталогов в файловой системе LFS

> Следующие команды выполнять из под root

Создание структуры каталогов

```bash
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}
```

Создние симлинков (LFS следует современному стандарту Usr-merge, где /bin, /lib и /sbin — это ссылки на их аналоги в /usr)

```bash
for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done
```

Для 64-битной системы (создаем lib64, хотя в примечании и сказано, что мы его не будем активно использовать, по инструкции он нужен на этом этапе)

```bash
case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac
```

Предоставить lfs полный доступ ко всем каталогам в каталоге $LFS, сделав lfs владельцем

```bash
chown -v lfs $LFS/{usr{,/*},var,etc,tools}
case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64 ;;
esac
```

### 3.2 Настройка среды ваполнения

> Переход под пользователя lfs

```bash
su - lfs
```

Разрешить make запускать до 32 заданий сборки

```bash
cat >> ~/.bashrc << "EOF"
export MAKEFLAGS=-j$(nproc)
EOF
```

```bash
source ~/.bash_profile
```
