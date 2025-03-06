# cIti


Для настройки сети на **Alt Server**, чтобы все машины (включая балансировщик нагрузки) работали корректно, нужно следовать определённой логике. Давайте разберём пошагово, как настроить сеть для вашей топологии.

---

## **Топология сети**

На основе вашего задания, топология сети выглядит следующим образом:

```
ControlVM  
INTERNET  

Балансировщик нагрузки  
Web1  
Web2
```

---

## **1. Подготовка сети**

### a) **Создание виртуальных сетей**

1. **Создайте виртуальную сеть для внутренних машин (Web1, Web2):**

   ```bash
   sudo nmcli connection add type bridge ifname br-internal
   sudo nmcli connection modify br-internal ipv4.addresses 192.168.1.1/24
   sudo nmcli connection modify br-internal ipv4.method manual
   sudo nmcli connection up br-internal
   ```

   - Эта сеть будет использоваться для связи между балансировщиком нагрузки и веб-серверами (Web1, Web2).

2. **Создайте виртуальную сеть для внешнего доступа (ControlVM, балансировщик):**

   ```bash
   sudo nmcli connection add type bridge ifname br-external
   sudo nmcli connection modify br-external ipv4.addresses 10.0.0.1/24
   sudo nmcli connection modify br-external ipv4.method manual
   sudo nmcli connection up br-external
   ```

   - Эта сеть будет использоваться для внешнего доступа к балансировщику нагрузки и ControlVM.

---

### b) **Настройка IP-адресов**

1. **Настройте IP-адреса для каждой машины:**

   - **ControlVM:**
     - Внешний IP: `10.0.0.2`
     - Шлюз: `10.0.0.1`
   - **Балансировщик нагрузки:**
     - Внешний IP: `10.0.0.3`
     - Внутренний IP: `192.168.1.2`
   - **Web1:**
     - Внутренний IP: `192.168.1.10`
   - **Web2:**
     - Внутренний IP: `192.168.1.11`

2. **Настройте IP-адреса на каждой машине:**

   Например, для **Web1**:

   ```bash
   sudo nmcli connection modify eth0 ipv4.addresses 192.168.1.10/24
   sudo nmcli connection modify eth0 ipv4.gateway 192.168.1.1
   sudo nmcli connection modify eth0 ipv4.method manual
   sudo nmcli connection up eth0
   ```

   Повторите аналогичные действия для других машин.

---

### c) **Настройка маршрутизации**

1. **Настройте маршрутизацию на балансировщике нагрузки:**

   Балансировщик должен уметь перенаправлять трафик между внешней и внутренней сетями.

   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   sudo iptables -t nat -A POSTROUTING -o br-external -j MASQUERADE
   ```

2. **Настройте маршруты на веб-серверах (Web1, Web2):**

   Убедитесь, что веб-серверы используют балансировщик как шлюз для выхода в интернет.

   ```bash
   sudo ip route add default via 192.168.1.2
   ```

---

## **2. Настройка балансировщика нагрузки**

### a) **Установка и настройка Nginx**

1. **Установите Nginx на балансировщике нагрузки:**

   ```bash
   sudo apt-get install nginx -y
   ```

2. **Настройте балансировку нагрузки:**

   Отредактируйте конфигурационный файл Nginx:

   ```bash
   sudo nano /etc/nginx/nginx.conf
   ```

   Добавьте следующий код:

   ```nginx
   http {
       upstream backend {
           server 192.168.1.10;
           server 192.168.1.11;
       }

       server {
           listen 80;
           server_name balance.example.com;

           location / {
               proxy_pass http://backend;
           }
       }
   }
   ```

3. **Перезапустите Nginx:**

   ```bash
   sudo systemctl restart nginx
   ```

---

### b) **Настройка безопасности**

1. **Разрешите трафик по HTTP и HTTPS:**

   ```bash
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

2. **Запретите все остальные порты:**

   ```bash
   sudo ufw default deny incoming
   sudo ufw enable
   ```

---

## **3. Настройка веб-серверов (Web1, Web2)**

### a) **Установка веб-сервера**

1. **Установите веб-сервер (например, Apache или Nginx):**

   ```bash
   sudo apt-get install apache2 -y
   ```

2. **Создайте тестовую страницу:**

   Например, для **Web1**:

   ```bash
   echo "Hello from Web1" | sudo tee /var/www/html/index.html
   ```

   Для **Web2**:

   ```bash
   echo "Hello from Web2" | sudo tee /var/www/html/index.html
   ```

3. **Запустите веб-сервер:**

   ```bash
   sudo systemctl start apache2
   sudo systemctl enable apache2
   ```

---

## **4. Проверка работы**

1. **Проверьте доступность веб-серверов через балансировщик:**

   Откройте браузер и перейдите по адресу `http://<IP-адрес балансировщика>`. Вы должны увидеть ответ от одного из веб-серверов (Web1 или Web2).

2. **Проверьте балансировку:**

   Обновите страницу несколько раз. Вы должны видеть, что ответы чередуются между Web1 и Web2.

---

## **5. Настройка VPN для WebAdm**

1. **Установите OpenVPN на WebAdm:**

   ```bash
   sudo apt-get install openvpn -y
   ```

2. **Настройте VPN-туннель:**

   Создайте конфигурационный файл для VPN и настройте подключение к Web1 и Web2.

3. **Проверьте подключение:**

   Убедитесь, что WebAdm может подключаться к Web1 и Web2 через VPN.

---

## **Итог**

Теперь у вас настроена сеть, балансировщик нагрузки и веб-серверы. Все машины взаимодействуют друг с другом, а балансировщик распределяет трафик между Web1 и Web2. Если у вас возникнут вопросы или ошибки, напишите, и я помогу вам разобраться! 😊

https://dzen.ru/a/Z8oUcB8cdyPBusex

