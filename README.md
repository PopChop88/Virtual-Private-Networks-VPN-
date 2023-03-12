# ****«Виртуальные частные сети (VPN)»****

В данном задании мы будем практиковаться в использовании самого популярного Open Source решения для организации VPN - OpenVPN.

Первое с чего мы начнём - немного теории. 
OpenVPN предлагает два виртуальных устройства: `TUN` (только IP-трафик) и `TAP` (любой трафик). Соответственно, для приложений всё выглядит так, как будто оно использует не обычный Ethernet-интерфейс, а другой, направляя через него трафик. OpenVPN же "шифрует" данный трафик и перенаправляет его через Ethernet-интерфейс.

Поднимите две виртуальные машины:

1. Ubuntu с Адаптер 1 - NAT и Адаптер 2 - Internal Network (10.0.0.1 - вручную)

2. Kali с Адаптер 1 - NAT и Адаптер 2 - Internal Network (10.0.0.2 - вручную)

3. Удостоверьтесь, что машины видят друг друга по адресам 10.0.0.1 и 10.0.0.2 соответственно (команда ping)

4. Установите на обеих машинах OpenVPN:
~~~
sudo apt update
sudo apt install openvpn
~~~

5. Дополнительно на Ubuntu установите сервер openssh:
~~~
sudo apt install openssh-server
~~~

6. А на Kali - mc:
~~~
sudo apt install mc
~~~
### ***P2P***
Начнём с режима P2P (Point-to-Point).

**PlainText**

Первое, что необходимо сделать, попробовать создать туннель без всяких механизмов шифрования и аутентификации.

*Ubuntu*
~~~
sudo openvpn --ifconfig 10.1.0.1 10.1.0.2 --dev tun
Где, 10.1.0.1 - это локальный VPN endpoint, 10.1.0.2 - удалённый VPN endpoint
~~~

*Kali*
~~~
sudo openvpn --ifconfig 10.1.0.2 10.1.0.1 --dev tun --remote 10.0.0.1
~~~

В данном случае адреса меняются местами и мы указываем к какому адресу нужно подключиться (режим P2P).

Откройте в Kali Wireshark и выберите интерфейс `eth1`.

Для тестирования мы будем использовать утилиту `netcat` (она позволит прослушивать на сервере определённый порт, а с клиента подключаться к этому порту).

Вам нужно и на Ubuntu и на Kali открыть ещё по одному терминалу (или вкладке терминала) и не завершая openvpn проделать остальные команды.

Ubuntu (прослушиваем порт 3000):

~~~
nc -l 3000
~~~

Kali (подключаемся через туннель к порту 3000 сервера):
~~~
nc 10.1.0.1 3000
~~~

Передаём любой текст, он будет отображаться на сервере в консоли
Удостоверьтесь в Wireshark, что данные передаются в открытом виде (Follow UDP Stream).
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/P2P_01.png?raw=true)
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/P2P_02.png?raw=true)

Завершите работу `openvpn` на сервере и на клиенте (Ctrl + C).

**Shared Key**

В этом режиме мы будем использовать один ключ для клиента и сервера.

*Ubuntu* (генерация ключа):
~~~
openvpn --genkey --secret vpn.key
cat vpn.key
~~~
Ключ будет выглядеть следующим образом:
~~~
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END OpenVPN Static key V1-----
~~~

Теперь необходимо безопасно передать ключ с сервера на клиент. Проще всего это сделать, воспользовавшись mc (файловый менеджер). 
Запускаем его, набрав в терминале mc:
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/mc01.png?raw=true)

Нажмите клавишу F9 после чего Enter, чтобы попасть в выпадающее меню. С помощью стрелок переместитесь на пункт `Shell link...` и нажмите Enter:
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/mc02.png?raw=true)

В строке подключения введите ubuntu@10.0.0.1, где ubuntu - это ваш логин на машине с Ubuntu (будет другой, если вы использовали образ с OSBoxes), после чего нажмите Enter:
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/mc03.png?raw=true)

Согласитесь с подключением (и сохранением отпечатка) к 10.0.0.1 (введите yes после чего нажмите Enter и введите пароль от учётной записи на машине Ubuntu):
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/mc04.png?raw=true)

Вы попадёте в корневой каталог вашего Ubuntu Server'а, по которому сможете перемещаться с помощью стрелок вверх вниз, заходить в каталоги - с помощью Enter, выходить - с помощью Enter на каталоге ...

Копирование в правую панель (где находится файловая система вашего локального компьютера с Kali) осуществляется с помощью клавиши F5 с подтверждением клавишей Enter:
![ufw](https://github.com/PopChop88/Virtual-Private-Networks-VPN-/blob/main/рис/mc05.png?raw=true)

Выход из mc осуществляется с помощью клавиши F10.

*Ubuntu*
~~~
sudo openvpn --ifconfig 10.1.0.1 10.1.0.2 --dev tun --secret vpn.key
~~~

*Kali*
~~~
sudo openvpn --ifconfig 10.1.0.2 10.1.0.1 --dev tun --remote 10.0.0.1 --secret vpn.key --providers legacy default
~~~

*Ubuntu* (прослушиваем порт 3000):
~~~
nc -l 3000
~~~

*Kali* (подключаемся через туннель к порту 3000 сервера):
~~~
nc 10.1.0.1 3000
~~~

Передаём любой текст, он будет отображаться на сервере в консоли
Удостоверьтесь в Wireshark, что данные не передаются в открытом виде (Follow UDP Stream).
