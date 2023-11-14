- [[#Введение]]
- [[#Подготовка]]
- [[#Шифруем Windows]]
- [[#Шифруем Linux]]
- [[#Загрузка системы]]
- [[#Установка и настройка rEFInd]]
- [[#Настройка UEFI Secure Boot with own machine key]]
- [[#Ссылки]]

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
![[Pasted image 20231113005049.png]]

Выбираем мультизагрузку:
![[Pasted image 20231113005243.png]]

Устанавливаем PIM. [Подробнее](https://veracrypt.eu/ru/Personal%20Iterations%20Multiplier%20%28PIM%29.html) о PIM можно почитать на сайте VeraCrypt. Коротко это второй фактор аутентификации, я рекомендую его его не игнарировать и выбирать значение между 485(значение по умолчанию) и 1000:
![[Pasted image 20231113005810.png]]

Установщик потребует сохранить диск востановления VeraCrypt и записать его на флешку отформатированную в Fat32. Обязательно сделайте это, и положите ее в надежное место:


После создания диска восстановления VeraCrypt попросит перезагрузиться для выполнения предварительного тестирования. После успешного теста можно приступать к шифрованию:
![[Pasted image 20231113011224.png]]

В процессе шифрования можно использовать систему как обычно:
![[Pasted image 20231113011351.png]]

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

С помощью программы rsync синхронизируем кталооги:
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
![[Pasted image 20231113231850.png]]

![[Pasted image 20231113232034.png]]

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
![[Pasted image 20231113235619.png]]
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
В fstab прописываем созданные виртуаальные тома

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


# Загрузка системы
Процесс подготовки загрузчика отличается для разных дистрибутивов, в остальном процесс идентичен. Настройки характерные для конкретного семейства дистрибутивов выведены в отдельные подразделы.
В качестве загрузчика будем использовать [Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image), который представляет собой .efi приложение в которое вмонтированы ядро системы и образ initramfs.
В пакете `systemd` (ранее в `gummiboot`) находится _linuxx64.efi.stub_ — заготовка UEFI-приложения, в которую можно встроить ядро, initramfs и аргументы, передаваемые ядру.
В /boot хранятся образы ядра и initramfs. 
## Kali
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

### Установка make-initrd и создание initramfs
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

![[Pasted image 20230322201529.png]]

### Создание UKI

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

## Arch
### Настройка mkinitcpio и создание Unified Kernel Image 
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

![[Pasted image 20230325100336.png]]
# Установка и настройка rEFInd
*Все команды требуют привелегий root пользователя*
Устанавливаем пакет:
```bash
pacman -S refind
```

![[refind_install.png]]

Монтируем загрузочный раздел:
```bash
mount /dev/sda2 /boot/efi
```

Создаем папку на загрузочном разделе:
```bash
mkdir -p /boot/efi/EFI/refind
```

Копируем наш boot manager:
```
cp /usr/share/refind/refind_x64.efi /boot/efi/EFI/refind/refind_x64.efi
```

Копируем конфигурационный файл:
```bash
cp /usr/share/refind/refind.conf-sample /boot/efi/EFI/refind/refind.conf
```

Устанаввливаем efibootmgr:
```bash
sudo pacman -S efibootmgr
```

![[efibootmgr_without_args.png.png]]

Настраиваем точку загрузки:
```bash
efibootmgr --create --disk /dev/sda --part 2 --loader /EFI/refind/refind.efi --label "rEFInd Boot Manager" --unicode
```

Опции:
- `--create` - Создать новую точку загрузки и добавить в список загрузок
- `--disk` - Диск на котором расположен загрузчик
- `--part` - Раздел диска на котором расположен загрузчик
- `--loader` - Сам загрузчик
- `--label` - Отображаемое имя
- `--unicode` - Кодировка параметров конмандной строки загрузки ядра

![[created_boot_entry.png]]

## Настраиваем конфигурацию
### Экран загрузки после установки
![[refind_default_view.png]]
#### Измененный конфиг
```bash
# Ожидание в секундах перед авто-выбором ОС
timeout 20
use_nvram false
# фоновый рисунок
banner /EFI/refind/m.png
# отключить автоматическое обнаружение загрузчиков в директориях
dont_scan_dirs /EFI/VeraCrypt
dont_scan_dirs /EFI/Microsoft

# Пункт для загрузки windows
menuentry Windows {
    icon \EFI\refind\icons\blue.png
    loader \EFI\Boot\bootx64.efi
}
# Пункт для загрузки Arch
menuentry Arch {
    icon /EFI/refind/icons/os_arch.png
    loader /EFI/arch/arch.efi
}
# Пунккт для загрузки Kali
menuentry kali {
    icon /EFI/refind/icons/red.png
    loader /EFI/crypt_kali/crypt_kali.efi
}
```

## Финальный вид

Содержимое папки:

![[final_refind_folder.png]]

Экран загрузки:

![[final_refind.png]]


# Настройка UEFI Secure Boot with own machine key
## Подготовка
### Настройка uefi

- Переводим Secure Boot в режим настроек
- Очищаем текущие ключи
- Монтируем загрузочный раздел: mount /dev/sda2 /boot/efi
- Делаем бекап загрузочного раздела: 
	`sudo cp -r /boot/efi /boot/efi_backup`
- Устанавливаем необходимые утилиты: 
	`sudo pacman -S efitools sbsigntool`

![](../_resources/UEFI_clear_keys.png)
![](../_resources/reset_to_setup.png)
### Бекап старых ключей
Текущие ключи можно просмотреть утилитой efi-readvar:

![[current_efi_keys.png]]

Сохраняем текущие ключи
```bash
for var in PK KEK db dbx; do efi-readvar -v $var -o old_${var}.esl; done
```

![[backup_keys_output.png]]
### Бекап загрузочного раздела

```bash
sudo cp -r /boot/efi /boot/efi_backup_121023
```

### Устанавливаем необходимые утилиты
```bash
sudo pacman -S efitools sbsigntool
```

## Создание ключей
### Создаем ключи

Создаем папку для наших ключей:
```bash
cd ~
mkdir uefi-keys
cd uefi-keys
```

Создаем Platform key:
```bash
openssl req -newkey rsa:4096 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=My Platform Key/" -out PK.crt

openssl x509 -outform DER -in PK.crt -out PK.cer
```

Создаем Key Echange Key:
```bash
openssl req -newkey rsa:4096 -nodes -keyout KEK.key -new -x509 -sha256 -days 3650 -subj "/CN=My Key Exchange Key/" -out KEK.crt
```

Создаем database signing key:
```bash
openssl req -newkey rsa:4096 -nodes -keyout db.key -new -x509 -sha256 -days 3650 -subj "/CN=My Signature Database key/" -out db.crt

openssl x509 -outform DER -in db.crt -out db.cer
```

![[gen_keys_output.png]]

### Конвертируем ключи в понятный EFI формат ESL

Создаем уникальный id:
```bash
uuidgen -r > guid.txt
```

Конвертируем ключи, используя созданный UID:
```bash
cert-to-efi-sig-list -g "$(< guid.txt)" PK.crt PK.esl
cert-to-efi-sig-list -g "$(< guid.txt)" KEK.crt KEK.esl
cert-to-efi-sig-list -g "$(< guid.txt)" db.crt db.esl
```

### Подписываем списки сертификатов

Подписываем PK самим собой:
```bash
sign-efi-sig-list -g "$(< guid.txt)" -k PK.key -c PK.crt PK PK.esl PK.auth
```

Подписываем KEK ключом PK:
```bash
sign-efi-sig-list -g "$(< guid.txt)" -k PK.key -c PK.crt KEK KEK.esl KEK.auth
```

Подписываем db ключом KEK:
```bash
sign-efi-sig-list -g "$(< guid.txt)" -k KEK.key -c KEK.crt db db.esl db.auth
```

Подписываем dbx ключом KEK:
```bash
sign-efi-sig-list -g "$(< guid.txt)" -k KEK.key -c KEK.crt dbx old_dbx.esl dbx.auth
```


#### Добавляем сертификат Microsoft для загрузки windows
Скачиваем сертификат
```bash
wget --user-agent="Mozilla" https://www.microsoft.com/pkiops/certs/MicWinProPCA2011_2011-10-19.crt
```

Добавляем GUID Microsoft в файл
```bash
echo "77fa9abd-0359-4d32-bd60-28f4e78f784b" > msguid.txt
```

Конвертируем в формат .esl
```bash
sbsiglist --owner 77fa9abd-0359-4d32-bd60-28f4e78f784b --type x509 --output ms_win_db.esl MicWinProPCA2011_2011-10-19.crt
```

Подписываем esl:
```bash
sign-efi-sig-list -a -g 77fa9abd-0359-4d32-bd60-28f4e78f784b -k KEK.key -c KEK.crt db ms_win_db.esl add_ms_db.auth
```

#### Добавляем 3-rd party 3-rd party 3-rd party сертификат Microsoft для загрузки VeraCrypt
Скачиваем сертификат
```bash
wget --user-agent="Mozilla" https://www.microsoft.com/pkiops/certs/MicCorUEFCA2011_2011-06-27.crt
```

Конвертируем в формат .esl
```bash
sbsiglist --owner 77fa9abd-0359-4d32-bd60-28f4e78f784b --type x509 --output ms_win_uef_db.esl MicCorUEFCA2011_2011-06-27.crt
```

```bash
sign-efi-sig-list -a -g 77fa9abd-0359-4d32-bd60-28f4e78f784b -k KEK.key -c KEK.crt db ms_win_uef_db.esl add_ms_uef_db.auth
```

![[Pasted image 20231013030629.png]]



## Добавление ключей в систему
Если у вас установлен пароль суперюзера для BIOS(а если не установлен, бросьте всё и немедленно установите), то для загрузки ключей необходимо его ввести. Из ОС его можно ввести записав в специальный файл. Расположение файла зависит от производителя оборудования, я использую Lenovo Thinkpad и в моем случае это выглядит так:
```bash
cat > /sys/class/firmware-attributes/thinklmi/authentication/Admin/current_password 
my-super-secret-password
^D
```

Создаем структуру папок для ключей и копируем ключи:
```bash
mkdir -p /etc/secureboot/keys/{db,dbx,KEK,PK}
cp db.auth /etc/secureboot/keys/db
cp add_ms_db.auth /etc/secureboot/keys/db
cp add_ms_uef_db.auth /etc/secureboot/keys/db
cp PK.auth /etc/secureboot/keys/PK
cp KEK.auth /etc/secureboot/keys/KEK
cp dbx.auth /etc/secureboot/keys/dbx
```

С помощью утилиты sbkeysync из ранее устанрвленного пакета sbsigntool добавляем ключи в систему:
```bash
sbkeysync --verbose
```

sbkeysync не добавляет Platform key, возможно это связано с тем что именно добавление Platform key переводит UEFI из режима настройки "Setup mode" в режим использования "User mode".

Для добавления Platform key воспользумся утилитой efi-updatevar из набора efitools:
```bash
efi-updatevar -f /etc/secureboot/keys/PK/PK.auth PK
```

## Подписание загрузчиков
Монитруем ESP раздел:
```bash
sudo mount /dev/sda2 /boot/efi
```

Подписываем загрузчики:
```bash
sudo sbsign --key db.key --cert db.crt --output /boot/efi/EFI/arch/arch.efi /boot/efi/EFI/arch/arch.efi
sudo sbsign --key db.key --cert db.crt --output /boot/efi/EFI/refind/refind.efi /boot/efi/EFI/refind/refind.efi
sudo sbsign --key db.key --cert db.crt --output /boot/efi/EFI/Boot/bootx64.efi /boot/efi/EFI/Boot/bootx64.efi
```

## Завершение настройки
Перезагружаемся, проверяем что UEFI находится в User mode, а  Secure boot в Custom mode.
Включаем Secure boot:
![[Pasted image 20231114133435.png]]

Проверяем невозможность загрузки неподписанных приложений:
![[Pasted image 20231114133518.png]]

Мы получили полностью зашифрованную систему с доверенной загрузкой и проверкой загрузчиков на основе собственных ключей электронной подписи. Но есть один нюанся)) О нём мы поговорим далее.

# Ссылки
[Используем Secure Boot в Linux на всю катушку](https://habr.com/ru/articles/308032/)
[О безопасности UEFI, часть пятая](https://habr.com/ru/articles/267953/)
[Укрощаем UEFI SecureBoot](https://habr.com/ru/articles/273497/)
[Полнодисковое шифрование Windows и Linux установленных систем. Зашифрованная мультизагрузка](https://habr.com/ru/articles/482696/)
[Hardware-and-Firmware-Security-Guidance](https://github.com/nsacyber/Hardware-and-Firmware-Security-Guidance)
[Документация VeraCrypt](https://www.veracrypt.fr/en/Documentation.html)
