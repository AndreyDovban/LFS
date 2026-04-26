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

Монтирование диска

```bash
sudo mount -v -t ext4 /dev/vdb1 $LFS
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
export LFS LC_ALL LFS_TGT PATH
EOF
```

Применение настроек

```bash
source ~/.bash_profile
```
