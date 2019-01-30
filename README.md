# Docker-образ Home Assistant + BLE

Этот образ специальным образом подготовлен для установки Home Assistant на Orange Pi Zero Plus 2 H5 в виде Docker-контейнера, что существенно упрощает установку и обновление ПО. В образе выполняется установка необходимого дополнительного ПО, решаются конфликты зависимостей и т.п. Docker-образ изолирует Home Assistant от другого установленного ПО, таким образом, на одном Orange Pi вы можете разместить совершенно различный софт, который не будет мешать друг другу.

## Использование образа для установки Home Assistant

После установки ОС по [этой инструкции](http://cutecare.readthedocs.io/ru/master/%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0/), необходимо подключиться по SSH (для Windows можно использовать Putty) и выполнить команды.

```
sudo -s
apt-get -y update && apt-get -y install docker.io=18.06.1-0ubuntu1~18.04.1
mkdir /home/home-assistant
docker run -d --name hass --restart unless-stopped -p 80:8123 -p 8080:8080 --cap-add=SYS_ADMIN --cap-add=NET_ADMIN --net=host -v /home/home-assistant:/config -v /dev:/dev -v /etc/localtime:/etc/localtime:ro cutecare/opi-home-assistant:latest
```

Параметры контейнера (hass) указаны таким образом, чтобы Home Assistant запускался при старте ОС, веб-интерфейс открывался по стандартному порту 80, конфигурационные файлы находились в каталоге /home/home-assistant

## Включение Bluetooth

По умолчанию в выбранном вами образе модуль Bluetooth может быть отключен. Выполните команды ниже, чтобы активировать модуль после перезагрузки микромпьютера.

```
sudo -s
apt-get -y install devmem2
```

Откройте на редактирование файл сервиса, обслуживающего Bluetooth, при помощи команды:

```
vi /etc/init.d/ap6212-bluetooth
```

Добавьте команды, управляющие пинами для включения Bluetooth в контроллере AP6212, чтобы текст демона выглядел следующим образом:

```
# Start patching

# ==> activate Bluetooth module
devmem2 0x1f00060 b 1
echo 10 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio10/direction
echo 0 > /sys/class/gpio/gpio10/value
echo 1 > /sys/class/gpio/gpio10/value
# <== activate Bluetooth module

rfkill unblock all
echo "0" > /sys/class/rfkill/rfkill0/state
echo "1" > /sys/class/rfkill/rfkill0/state
echo " " > /dev/$PORT
hciattach /dev/$PORT bcm43xx 115200 flow bdaddr $MAC_OPTIONS
hciconfig hci0 up
```

Сохраните изменения и перезапустите сервис:

```
systemctl daemon-reload
service ap6212-bluetooth restart
```

После этого Bluetooth модуль должен запуститься:

```
# hciconfig
hci0:   Type: BR/EDR  Bus: UART
        BD Address: 43:29:B1:55:01:02  ACL MTU: 1021:8  SCO MTU: 64:1
        UP RUNNING
        RX bytes:738 acl:0 sco:0 events:44 errors:0
        TX bytes:1750 acl:0 sco:0 commands:44 errors:0
```

## Просмотр логов Home Assistant

```
sudo docker logs hass
```

Чтобы логи не съедали все место на диске необходимо ограничить их размер.
Сделать это можно следующим образом:

```
cat > /etc/docker/daemon.json << EOF
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "2m",
        "max-file": "5"
    }
}
EOF
service docker restart
```

## Выполнение команд в образе Home Assistant

```
sudo docker exec hass echo 'hello from hass image'
```


# Сборка и публикация образа

Вы можете доработать образ, добавив туда необходимых компонентов. Сборка образа выполняется на Orange Pi. Перед публикацией образа в Docker Hub, необходимо в нем зарегистрироваться, а затем залогиниться командой:

```
sudo -s
apt-get -y install jq curl git
git clone https://github.com/cutecare/opi-home-assistant.git
cd opi-home-assistant
docker login
./build.sh
```
