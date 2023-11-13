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
kkkk

