# Введение
Жил-был один ноутбук с 2-мя операционными системами в режиме Dual Boot. И однажды на нем стала появляться чувствительная информация, так что владелец этого ноутбука стал задумываться о защите своих данных. В результате дум получились следующие требования:
- Диски должны быть полностью зашифрованы.
- Должна быть настроена доверенная загрузка с помощью Secure Boot.
- Загрузчики должны верфицироваться с помощью собственных сертификатов.
- Должна быть возможность гибкой настройки параметров в дальнейшем.
- При настройке текущие системы не должны переустанавливаться.
В результате изучения матеирлов по теме появился следующий гайд.
Присутпим.

# Подготовка

## Требуемые вводные

- Диск использующий таблицу разделов GPT.
- Наличие свободного места на разделе с Linux равного занятому+раздел SWAP+10%
- Флешка с Live CD Linux(Дистрибутив любой который Вам нравится, в моем случае Kali Live CD)
- [Дистрибутив VeraCrypt](https://sourceforge.net/projects/veracrypt/files/)

## Исходный список разделов

|Раздел|Назначение|
|---|---|
|/dev/sda1|Служебный раздел Windows|
|/dev/sda2|Раздел с загрузчиками UEFI|
|/dev/sda3|резервный раздел Windows|
|/dev/sda4|Основной системный раздел Windows на котором расположена система|
|/dev/sda5|Загрузчик GRUB|
|/dev/sda6|Основной системный раздел Linux на котором расположена система|
|/dev/sda7|Раздел файла подкачки Linux Swap|

# Шифруем Windows
> Шифрование с использование VeraCrypt требует добавления Microsoft 3-rd party certificate при настройке SecureBoot с собственными ключами. Если вы хотите использовать SecureBoot и не хотите разрешать загрузку сторонних приложений подписанных Microsoft используйте [Bitlocker](https://learn.microsoft.com/ru-ru/windows/security/operating-system-security/data-protection/bitlocker/) для шифрования Windows. 

VeraCrypt устанавливается без каких-либо проблем по принципу "Далее-Далее-Готово" поэтому здесь я не буду описывать процесс установки. Перейдем сразу к шифрованию.
Запускаем VeraCrypt и выбираем шифроание системного диска:
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/37bc41f6-08a7-4e78-a3d2-cfb32a8ae698)


Выбираем мультизагрузку:

![image](https://github.com/Green-Hamster/feHDD/assets/47595907/99c850cc-142d-4d5d-a094-99c75f37d78b)

Устанавливаем PIM. [Подробнее](https://veracrypt.eu/ru/Personal%20Iterations%20Multiplier%20%28PIM%29.html) о PIM можно почитать на сайте VeraCrypt. Коротко это второй фактор аутентификации, я рекомендую его его не игнарировать и выбирать значение между 485(значение по умолчанию) и 1000:
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/7a1f0b55-44a8-42ab-942b-bbe623cefa24)

Установщик потребует сохранить диск востановления VeraCrypt и записать его на флешку отформатированную в Fat32. Обязательно сделайте это, и положите ее в надежное место:


После создания диска восстановления VeraCrypt попросит перезагрузиться для выполнения предварительного тестирования. После успешного теста можно приступать к шифрованию:
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/f30867e2-8cfc-4827-ab34-cad62e0c4fa2)

В процессе шифрования можно использовать систему как обычно:
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/74273d6e-a6c4-4433-b81d-fa06f1b4fe5f)

# Шифруем Linux

## Создание раздела для переноса ОС

- Загружаемся с Live CD
- Запускаем Gparted
- Отрезаем свободное место от раздела с Linux и получаем неразмеченную область.
- В полученной области создаем новый раздел.

## Создание зашифрованного раздела
```bash
cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 2000 /dev/sda8
```

Опции:  
  
* luksFormat -инициализация LUKS заголовка;  
* --cipher - выбранный алгоритм шифрования;
* --key-size - длина ключа для алгоритма шифрования;
* --hash - выбранный алгоритм хеширования
* --iter-time - время в миллисекундах, затрачиваемое на генерацию ключа из вводимого пароля функцией PBKDF2;
* /dev/sda8 - раздел для переноса ОС созданный в предыдущем пункте.


## Монтируем криптоконтейнер

Открываем криптоконтейнер сопоставляя имя защифрованного раздела и имя криптоконтейнера.
Имя криптоконтейнера выбирается самостоятельно, необходимо запомнить его, в дальнейшем оно будет использовано 

```shell
cryptsetup open /dev/sda8 sda8_crypt
#выполнение данной команды запрашивает ввод секретной парольной фразы.
```

ОПЦИИ:
* open -сопоставить раздел «с
именем»
* /dev/sda8 - имя созданного ранее раздела на котором расположен криптоконтейнер
* sda8_crypt - выбранное имя криптоконтейнера, которое используется для монтирования зашифрованного раздела или его инициализации при загрузке ОС.

## Создание логических томов на зашифрованном разделе

Инициализируем физический раздел LVM поверх /dev/mapper/sda8_crypt:
```bash
pvcreate /dev/mapper/sda8_crypt
```

Внутри этого физического раздела создадим группу томов с именем kali:
```bash
vgcreate kali /dev/mapper/sda8_crypt
```
Теперь мы можем создавать логические тома внутри этой группы. Первым делом создадим том для раздела подкачки и инициализируем его. Рекомендуемый размер — от sqrt(RAM) до 2xRAM в гигабайтах.
```bash
#создать логический том с меткой swap размером 4 ГБ в группе kali
lvcreate -n swap -L 4G kali
#Отформатировать раздел для swap
mkswap /dev/ubuntu/swap
```

Добавим том для корня и создадим в нём файловую систему ext4. Хорошей практикой считается оставлять свободное место и расширять тома по мере необходимости, поэтому выделим для корня 70 ГБ. Выделить всё свободное место для тома можно с помощью параметра `-l 100%FREE`.
```bash
#создать логический том с меткой root размером 70GB в группе kali
lvcreate -n root -L 70G kali
#Создать на логическом томе root файловую систему ext4
mkfs.ext4 /dev/kali/root
```

## Перенос системы на зашифрованный раздел
Монтируем созданный виртуальный том в систему  

```bash
mount /dev/mapper/kali-root /mnt
```


Создаем папку /mnt2 _(Примечание — мы все еще работаем с live usb, в точку /mnt смонтирован sda8\_crypt)_, монтируем наш текущий GNU/Linux в созданную папку:  
```bash
mkdir /mnt2
mount /dev/sda6 /mnt2
```

С помощью программы rsync синхронизируем каталоги:
`rsync -avlxhHX --progress /mnt2/ /mnt`

Опции:
* -а -режим архива  
* -v -вербализация.
* -l — копировать символьные ссылки
* -x — работать только в этой файловой системе
* -H -копировать хардлинки, как есть; 
* -g -сохранить группы;  
* -P --progress — статус времени работы над файлом;  

Далее, **необходимо** провести дефрагментацию раздела логического диска  

```shell
#после проверки, e4defrag выдаст, что степень дефрагментации раздела ≈ 0, это заблуждение, которое может вам стоить существенной потери производительности!
e4defrag -c /mnt
#проводим дефрагментацию шифрованной GNU/Linux/ 
e4defrag /mnt/
```
## Настройка системы на зашифрованном разделе
### Смена корневого каталога
Создаем файлы-маркеры на шифрованной и нешифрованной ОС
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/6a34b4fa-f6cf-48de-ad05-97b2efd6f5b3)

![image](https://github.com/Green-Hamster/feHDD/assets/47595907/ab06f0a8-2523-43d3-8516-3d68e5c1123b)


связываем служебные файлы и меняем корневой кталог с помощью `chroot`:
```bash
mount /dev/ubuntu/root /mnt
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /proc /mnt/proc
mount --bind /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt
```

### Настройка /etc/crypttab
Запускаем `blkid`
Находим строку, которую строку которая начинается с имени раздела на котором был создан криптоконтейнер:
![image](https://github.com/Green-Hamster/feHDD/assets/47595907/4d5e8122-289c-4ded-81dc-af4d15bdbf0f)

Копируем значение UUID
Добавляем запись в crypttab
```bash
nano /etc/fstab
```

```toml
# Configuration for encrypted block devices.
# See crypttab(5) for details.

# NOTE: Do not list your root (/) partition here, it must be set up
#       beforehand by the initramfs (/etc/mkinitcpio.conf).

# <name>       <device>                                     <password>              <options>
# home         UUID=b8ad5c18-f445-495d-9095-c9ec4f9d2f37    /etc/mypassword1
# data1        /dev/sda3                                    /etc/mypassword2
# data2        /dev/sda5                                    /etc/cryptfs.key
# swap         /dev/sdx4                                    /dev/urandom            swap,cipher=aes-cbc-essiv:sha256,size=256
# vol          /dev/sdb7                                    none
sda5_crypt UUID=470e92ff-6faf-4f19-8927-15426210f5e1 none luks
```

### Настройка /etc/fstab
В fstab прописываем созданные виртуальные тома

```bash
nano /etc/fstab
```

```bash
# «file system» «mount poin» «type» «options» «dump» «pass»  
# / was on /dev/mapper/kali-root during installation  
/dev/mapper/kali-root / ext4 errors=remount-ro 0 1

#swap
/dev/mapper/kali-swap none swap sw 0 0
```
## Загрузка системы
Процесс подготовки загрузчика отличается для разных дистрибутивов, в остальном процесс идентичен. Настройки характерные для конкретного семейства дистрибутивов выведены в отдельные подразделы.
В качестве загрузчика будем использовать [Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image), который представляет собой .efi приложение в которое вмонтированы ядро системы и образ initramfs.

В пакете `systemd` (ранее в `gummiboot`) находится _linuxx64.efi.stub_ — заготовка UEFI-приложения, в которую можно встроить ядро, initramfs и аргументы, передаваемые ядру.
В /boot хранятся образы ядра и initramfs. 
### Kali
Для подготовки образа initramfs будем использовать make-initrd.

Проверяем конигурацию cryptsetup:
```
nano /etc/initramfs-tools/conf.d/cryptsetup
```  

должно соответствовать:
```bash
#/etc/initramfs-tools/conf.d/cryptsetup  
CRYPTSETUP=yes  
export CRYPTSETUP
```

#### Установка make-initrd и создание initramfs
Устанавливаем необходимые пакеты:
```bash
apt-get install systemd-boot binutils make automake pkg-config udev libkmod-dev libz-dev libbz2-dev liblzma-dev libzstd-dev libelf-dev libtirpc-dev libcrypt-dev help2man gcc opensc pcscd libpcsclite1 byacc bison
```

Клонируем репозиторий:
```bash
git clone https://github.com/osboot/make-initrd --recursive
```

Переходим в папку проекта:
```bash
cd make-initrd
```

Собираем проект:
```bash
./autogen.sh 
./configure
make
```


Делаем бекап образа initramfs:
```bash
cp /boot/initrd.img-6.1.0-kali5-amd64{,.backup1}
```


Создаем новый образ:
```bash
make-initrd
```

![Pasted image 20230322201529](https://github.com/Green-Hamster/feHDD/assets/47595907/345f85ae-9f69-4240-ab35-89b4012a847f)


#### Создание UKI

Запишем в /boot/cmdline аргументы, которые будут передаваться ядру:
```bash
echo -n "quite splash" > /tmp/cmdline
```

Ниже представлен пример скрипта для создания UKI. В скрипте вычисляются сдвиги для корректного встраивания данных в заготовку приложения и непосредственно создание загрузчика.

>При использовании скрипта необходимо заменить названия файлов ядра и initramfs в соответствии с вашей текущей версией.

```bash
#!/bin/bash

# Extract the section alignment value
align=$(objdump -p /usr/lib/systemd/boot/efi/linuxx64.efi.stub | awk '/SectionAlignment/ { print $2 }')
align=$((16#$align))

# Helper function to calculate aligned offset
calculate_aligned_offset() {
    local current_offset=$1
    echo $((current_offset + align - current_offset % align))
}

# Calculate offsets for various sections
osrel_offs=$(objdump -h "/usr/lib/systemd/boot/efi/linuxx64.efi.stub" | \
    awk 'NF==7 {size=strtonum("0x"$3); offset=strtonum("0x"$4)} END {print size + offset}')
osrel_offs=$(calculate_aligned_offset $osrel_offs)

cmdline_offs=$(calculate_aligned_offset $(($osrel_offs + $(stat -Lc%s "/usr/lib/os-release"))))
initrd_offs=$(calculate_aligned_offset $(($cmdline_offs + $(stat -Lc%s "/boot/cmdline"))))
linux_offs=$(calculate_aligned_offset $(($initrd_offs + $(stat -Lc%s "initrd.img-6.1.0-kali5-amd64"))))

# Add and change sections in the EFI stub
objcopy \
    --add-section .osrel="/etc/os-release" --change-section-vma .osrel=$(printf 0x%x $osrel_offs) \
    --add-section .cmdline="/tmp/cmdline" --change-section-vma .cmdline=$(printf 0x%x $cmdline_offs) \
    --add-section .initrd="/boot/initrd.img-6.1.0-kali5-amd64" --change-section-vma .initrd=$(printf 0x%x $initrd_offs) \
    --add-section .linux="/boot/vmlinuz-6.1.0-kali5-amd64" --change-section-vma .linux=$(printf 0x%x $linux_offs) \
/usr/lib/systemd/boot/efi/linuxx64.efi.stub /boot/efi/EFI/crypt_kali/crypt_kali.efi

```

### Arch
#### Настройка mkinitcpio и создание Unified Kernel Image 
В случае с Arch linux для создания UKI мы будем использовать mkinitcpio и файл linux.preset


Монтируем ESP раздел:
```bash
mount /dev/sda2 /boot/efi
```

Создаем папку для нашего загрузчика:
```bash
mkdir /boot/efi/EFI/arch
```

>*Важно!* Для Arch Linux в качестве параметра ядра обязательно должны быть переданы параметры шифрованного раздела. 

Запишем в /boot/cmdline аргументы, которые будут передаваться ядру:
```toml
rw cryptdevice=UUID=470e92ff-6faf-4f19-8927-15426210f5e1:sda5_crypt root=/dev/arch/root resume=/dev/arch/swap
```

Конфигурационный файл mkinitcpio находится по пути  `/etc/mkinitcpio.conf`

Пример рабочей конфигурации mkinitcpio:
```toml
MODULES=()
BINARIES=()
FILES=()
HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

> *Важно!* Обратите внимание на последовательность объявления hook'ов. Она имеет значение, encrypt и lvm должны быть указаны между block и filesystems и именно в такой последовательности.

Пример рабочего конфига linux.preset:
```toml
# mkinitcpio preset file for the 'linux' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
#ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/boot/efi/EFI/arch/arch.efi"
default_options="--cmdline /boot/cmdline --splash=/usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
fallback_image="/boot/initramfs-linux-fallback.img"
#fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Создаем загрузчик:
```bash
mkinitcpio -P
```

## Итоговая структура разделов

![Pasted image 20230325100336](https://github.com/Green-Hamster/feHDD/assets/47595907/347cadf9-8113-4f66-bdbc-b803e9d80a9e)
