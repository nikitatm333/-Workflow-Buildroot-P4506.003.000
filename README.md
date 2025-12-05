# -Workflow-Buildroot-P4506.003.000

<a name="build"></a>

## Используем внешнее дерево <a href="https://gitlab.infratest.local/navigator/br-ext-infratest">br-ext-infratest</a>
**Хост**

Клонируем репозиторий на хост и запускаем сборочный контейнер:
```
tnv@debian:~/BOX_hi3518$ git clone ssh://git@gitlab.infratest.local:2289/navigator/br-ext-infratest.git
tnv@debian:~/BOX_hi3518$ cd br-ext-infratest/
tnv@debian:~/BOX_hi3518/br-ext-infratest$ ./run-docker.sh 
# При необходимости можно взять другую версию buildroot с https://registry-ui.infratest.local/#!/taglist/kuklin_m/build-hisilicon

```
**Docker**

Выбираем defconfig который послужит основой нашего образа, можем заранее его скопировать и переминовать
```
root@e62913301a67:/buildroot# cd ../br-ext-navigator/configs/
root@e62913301a67:/br-ext-navigator/configs# cp tmp47_defconfig P4506.003.000_defconfig 
# Возвращаемся назад 
root@e62913301a67:/br-ext-navigator/configs# cd ../../buildroot/ 
```

## Указываем внешнее дерево через BR2_EXTERNAL:
**Docker**
```
root@35d756da17b4:/buildroot# make BR2_EXTERNAL=/br-ext-navigator P4506.003.000_defconfig
#
# configuration written to /buildroot/.config
#
```

# Поправить P4506.003.000_defconfig
Прежде чем идти дальше нужно обязательно проверить defconfig вашей платы (NAVI_DEV_NAME и тп ...)

# Опционально (если делали изменения) - сохраняем defconfig
**Docker**
```
root@e62913301a67:/buildroot# make BR2_EXTERNAL=/br-ext-navigator savedefconfig BR2_DEFCONFIG=/br-ext-navigator/configs/P4506.003.000_defconfig
```

## На хосте создаем board для нашей платы
Патчи, настройки и тд будем сохранять в конкретную директорию вашей board

**Хост**
```
tnv@debian:~/BOX_hi3518/br-ext-infratest/board$ mkdir P4506.003.000
```

## Сохраняем конфиг ядра и перемещаем в board
**Docker**
```
root@35d756da17b4:/buildroot# make linux-savedefconfig 
по пути output/build/linux-4.9.37 увидим новый файл defconfig (это defconfig ядра)
root@35d756da17b4:/buildroot# ls output/build/linux-4.9.37/
COPYING  Documentation  Kconfig      Makefile        README          System.map  block  crypto     drivers   fs       init  kernel  mm               modules.order  samples  security  tools  virt     vmlinux.o
CREDITS  Kbuild         MAINTAINERS  Module.symvers  REPORTING-BUGS  arch        certs  defconfig  firmware  include  ipc   lib     modules.builtin  net            scripts  sound     usr    vmlinux
```
Копируем этот defconfig на наш board и переминовываем в linux-4.9.config

## Собираем образ
**Docker**
```
root@35d756da17b4:/buildroot# make -j$(nproc)
#
  ...
#
```


## Добавление патчей
Патчи ядра должны лежать по конкретному пути относительно вашей board

**Хост**
```
tnv@debian:~/BOX_hi3518/br-ext-infratest/board/P4506.003.000/patches/linux$ ls
0001-enable-spidev-on-spi_bus1.patch  0002-enable-rs.patch  0003-enable-dac.patch
```
Схема простая:
- Сделали изменения
- git add file1 & git commit -m "enable_new_fun"
- Генерируем патч и кладем в /patches/linux
- В сборочном контейнере: make linux-dirclean(при необходимости) -> make my_board -> make linux-rebuild

# Пример с spi (включение драйвера ЦАП ad5420 сидящего на spi):
## Написание и сохранение патчей
1) Редактируем/добавляем ноду spi в hi3518ev300-demb.dts:
```
&spi_bus1 {
    status = "okay";
    /* AD5420 on SPI*/
    ad5420@0 {
        compatible = "adi,ad5420";    
        reg = <0>;                    
        spi-max-frequency = <30000000>; 
        ldac-gpios = <&gpio_chip7 3 0>;   /* GPIO7_3 — LATCH */
        clear-gpios = <&gpio_chip8 5 0>;  /* GPIO8_5 — CLEAR */
		adi,current-range = <0>; // 0=4-20mA, 1=0-20mA, 2=0-24mA
		status = "okay";
    };
};
```
2) Добавляем драйвер ad5420 (linux-4.9.37/drivers/iio/dac/ad5420.c):
```
Пишем (желательно рабочий ^_^) драйвер
```
3) Чтобы в драйвер собрался и мы его могли включить в сборку нужно изменить Kconfig и Makefile в linux-4.9.37/drivers/iio/dac/:

**Makefile**
```
obj-$(CONFIG_AD5420) += ad5420.o
```
**Kconfig**
```
config AD5420
    tristate "Analog Devices AD5420 DAC driver"
    depends on SPI
    help
      Minimal AD5420 DAC driver.
```

4) Делаем коммит и сохраняем патч
```
tnv@debian:~/BOX_hi3518/linux-4.9.37$ git add .
tnv@debian:~/BOX_hi3518/linux-4.9.37$ git commit -m "enable-dac-driver" 
tnv@debian:~/BOX_hi3518/linux-4.9.37$ # Генерируем и сохраняем патч в /br-ext-infratest/board/my_board
tnv@debian:~/BOX_hi3518/br-ext-infratest/board/P4506.003.000$ ls patches/linux/
0003-enable-dac-driver.patch

```
### getpatch.sh
**Полезный скрипт при написании большого количества патчей**
```
#!/bin/bash

# Директория с patch'ами
PATCHDIR=../br-ext-infratest/board/P4506.003.000/patches/linux

# Убедимся, что директория существует
ls -ld "$PATCHDIR" || ( echo "Папка $PATCHDIR не найдена"; exit 1 )

# Посчитаем следующую нумерацию 
NUM=$(ls "$PATCHDIR" | grep -E '^[0-9]{4}-' | wc -l)
NEXT=$(printf "%04d" $((NUM+1)))

# Возьмём хеш последнего коммита (или указать нужный хеш)
HASH=$(git rev-parse --verify HEAD)

# Берем текст последнего коммита
COMMIT_SUBJECT=$(git show -s --format=%s "$HASH")

# Заменим пробелы на дефисы для читаемости
FILE_SUBJECT=$(echo "$COMMIT_SUBJECT" | tr ' ' '-' | tr -dc '[:alnum:]-._')

#  Общий вид
PATCH_FILE="$PATCHDIR/${NEXT}-${FILE_SUBJECT}.patch"

# Патчим
git format-patch -1 "$HASH" --stdout > "$PATCH_FILE"

# Убедимся, что файл создан
ls -l "$PATCH_FILE"

```
## Включение патчей в сборку
**Docker**

Дальше нужно применить изменения в defconfig вашей платы:
(Предварительно лучше очистить исходники ядра: make linux-dirclean)
```
root@e62913301a67:/buildroot# make P4506.003.000_defconfig
root@e62913301a67:/buildroot# # Так как мы хотим еще включить драйвер ad5420, то перед пересборкой ядра:
root@e62913301a67:/buildroot# make linux-menuconfig
```
В меню конфигурации ядра Linux переходим Device Drivers > Industrial I/O support > Digital to analog converters и выбираем как включим драйвер:
- <*> Analog Devices AD5420 DAC driver (непосредственно в основное тело образа ядра Linux) 
- <M> Analog Devices AD5420 DAC driver (как отдельный загружаемый модуль ядра .ko)

Сохраняемся и выходим из menuconfig

```
root@e62913301a67:/buildroot# make linux-rebuild 
```
### Сохранить новый defconfig ядра
Чтобы каждый раз при очистке ядра не делать вышеперечисленные действия в меню конфигурации ядра, нужно по новой сохранить defconfig ядра.

**<a href="https://gitlab.infratest.local/-/snippets/14#%D1%81%D0%BE%D1%85%D1%80%D0%B0%D0%BD%D1%8F%D0%B5%D0%BC-%D0%BA%D0%BE%D0%BD%D1%84%D0%B8%D0%B3-%D1%8F%D0%B4%D1%80%D0%B0-%D0%B8-%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D1%89%D0%B0%D0%B5%D0%BC-%D0%B2-board">Это мы делали тут</a>**

На выходе получим свежий UImage

# Создание в NetworkManager постоянного сетевого профиля (Если вдруг плата не пингуется по 10.0.0.1)
**Хост**
```
tnv@debian:~/BOX_hi3518/emmc-burn-script$ ls /sys/class/net | grep ^enx
enx8e125865d3de <-- Копируем
tnv@debian:~/BOX_hi3518/emmc-burn-script$ sudo nmcli connection modify hisilicon-usb ifname enx8e125865d3de
tnv@debian:~/BOX_hi3518/emmc-burn-script$ sudo systemctl restart NetworkManager
tnv@debian:~/BOX_hi3518/emmc-burn-script$ sudo nmcli device connect enx8e125865d3de
# Затем можно проверить статус
tnv@debian:~/BOX_hi3518/emmc-burn-script$ nmcli device status
enx8e125865d3de  ethernet  подключено            hisilicon-usb 
```
# Прошивка платы 
**Хост**
Прошиваем с помощью репозитория <a href="https://gitlab.infratest.local/navigator/emmc-burn-script">emmc-burn-script</a>
```
tnv@debian:~/BOX_hi3518/emmc-burn-script$ python3 -m venv .venv
tnv@debian:~/BOX_hi3518/emmc-burn-script$ source .venv/bin/activate
(.venv) tnv@debian:~/BOX_hi3518/emmc-burn-script$ pip install -r requirments.txt
# Зажимаем boot, включаем плату и запускаем скрипт
(.venv) tnv@debian:~/BOX_hi3518/emmc-burn-script$ sudo .venv/bin/python3 emmc-burn.py -p ../br-ext-infratest/out/partition.xml
```

# Замена uImage
**Таргет**
```
dd if=uImage of=/dev/mmcblk0p2
```
# Поменять регистры
**Таргет**
```
tnv@debian:~/BOX_hi3518/br-ext-infratest/host_utils/uboot_tools/hiregbin-v5.0.1$ ./hiregbin ../../../../br-ext-infratest/board/P4506.003.000/boot/reg_sirius_hi3518ev300.xlsx ../../../../br-ext-infratest/board/P4506.003.000/boot/reg_info_hi3518ev300.bin 
```
# Замена uboot
**Таргет**
```
dd if=u-boot-hi3518ev300.bin of=/dev/mmcblk0p1
```
# Кросскомпилятор
**Docker**
```
output/host/bin/arm-buildroot-linux-uclibcgnueabi-gcc -O2 -static -o file file.c
```
