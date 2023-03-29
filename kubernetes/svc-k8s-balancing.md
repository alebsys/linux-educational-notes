# Балансировка в Kubernetes на примере инцидента с coreDNS

Опишу недавний инцидент, который повлек за собой всплеск ошибок на клиентских сервисах. 

### Исходные данные

Есть kubernetes кластер c набором микросервисов на борту. Команда инфра-платформы вывела пару Нод друг за другом на обслуживание. 

На клиентских приложениях были замечены всплески http таймаутов, но спустя пару минут все пришло в норму.

### Анализ

Триггер проблемы простой - Ноды, которые выводились не были “задрейнены”, а на обоих было по экземпляру **coreDNS**. 

Соответственно те запросы, которые продолжали на них попадать, пока кубовый **svc** не перестроил правила (как выяснилось помимо этого были и другие причины), повисали и в итоге отбивались приложениями по таймауту.

Все логично, но оставался вопрос почему DNS ретраи, если таковые вообще были, не попадали на живые Поды **coreDNS**. 

Давайте разбираться. 

### SVC и iptables

Кубовый сервис это виртуальная сущность, что по факту является просто набором iptables правил, которые в итоге будут транслироваться в конкретный адрес Пода. [Подробнее на примере nodePort](https://ronaknathani.com/blog/2020/07/kubernetes-nodeport-and-iptables-rules/).

Этот набор правил управляется через **kube-proxy** компонент, причем так, что всегда будет некий рассинхрон в моменте между реальной картиной (часть Подов стартовала/завершилась) и содержанием в правилах. Последнее будет незначительно (или как настроите) отставать.

```bash
$ iptables -t nat -L KUBE-SERVICES -n  | column -t
Chain                      KUBE-SERVICES  (2   references)
target                     prot           opt  source       destination
KUBE-SVC-TCOU7JCQXEZGVUNU  udp            --   0.0.0.0/0    10.96.0.10    /*  kube-system/kube-dns:dns      cluster  IP          */     udp   dpt:53
...
$ iptables -t nat -L KUBE-SVC-TCOU7JCQXEZGVUNU -n  | column -t
Chain                      KUBE-SVC-TCOU7JCQXEZGVUNU  (1   references)
target                     prot                       opt  source        destination
...
KUBE-SEP-732S4P25776SCG6Z  all                        --   0.0.0.0/0     0.0.0.0/0    /*  kube-system/kube-dns:dns  ->       192.168.132.67:53  */  statistic  mode    random  probability  0.50000000000
KUBE-SEP-FKMZ2ZAICZ6XU5GU  all                        --   0.0.0.0/0     0.0.0.0/0    /*  kube-system/kube-dns:dns  ->       192.168.132.68:53   */ 
$ iptables -t nat -L KUBE-SEP-732S4P25776SCG6Z -n  | column -t
Chain           KUBE-SEP-732S4P25776SCG6Z  (1   references)
target          prot                       opt  source          destination
...
DNAT            udp                        --   0.0.0.0/0       0.0.0.0/0    /*  kube-system/kube-dns:dns  */  udp  to:192.168.132.67:53
```

В разрезе **coreDNS** сервиса с двумя Подами под капотом видим, что в 50% случаев трафик пойдет на `192.168.132.67:53`, в остальных в `192.168.132.68:53`. Вот и вся балансировка.

### Воспроизведение отказа DNS

Напишем правило дропающее входящий udp трафик (ответы) для определенного Пода:

```bash
$ iptables -I INPUT -p udp --sport 53 -d 192.168.49.84 # 192.168.49.84 адрес Пода c запросом на резолв
```

Кинем из Пода запрос на резолв и параллельно снимем трафик через `tcpdump`

```bash
$ kubectl exec -it dnsutils -- sh # Под уже присутствовал в кластере
/ # dig ya.ru +short

; <<>> DiG 9.11.6-P1 <<>> ya.ru +short
;; global options: +cmd
;; connection timed out; no servers could be reached
/ # command terminated with exit code 9

tcpdump udp -i calib3c61c3cba9
10:35:46.189943 IP 192.168.49.84.39249 > 10.96.0.10.domain: 28311+ [1au] A? ya.ru. (46)
10:35:51.189677 IP 192.168.49.84.39249 > 10.96.0.10.domain: 28311+ [1au] A? ya.ru. (46)
10:35:56.189785 IP 192.168.49.84.39249 > 10.96.0.10.domain: 28311+ [1au] A? ya.ru. (46)
```
| подробнее как найти интерфейс контейнера писал [тут](https://alebsys.github.io/posts/how_to_view_container_interface/) или можно воспользоваться утилиткой [clink](https://github.com/alebsys/clink) или любым удобным для вас способом

Под отправил 3 запроса, где 2 из них судя по всему ретраи через 5с каждый. Копаем дальше. 

### /etc/resolv.conf

Резолв Пода происходит согласно файлу `/etc/resolve.conf`, выглядит он так (подробнее я писал про него [тут](https://alebsys.github.io/posts/kube-dns-resolve/)):

```
search namespace.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

То есть из опций только **`ndots`**, а остальное - стандартные настройки. 

Меня интересуют кол-во ретраев и таймауты. Если посмотреть в [manpage **`resolv.conf`**](https://man7.org/linux/man-pages/man5/resolv.conf.5.html), то увидим, что за

- ретрай отвечает опция **`attempts`**, по дефолту она равна 2 попыткам;
- таймаут между ретраями отвечает **`timeout`**, по умолчанию 5 секунд.

Именно это поведение мы и наблюдаем. Осталось понять куда летят ретраи относительно **coreDNS** Подов. 

### conntrack и upd

Ядро поддерживает таблицу коннектов для отслеживания их состояний, что позволяет реализовывать более “умные” правила на фаерволе. Содержание таблицы можно посмотреть утилитой [conntrack](https://manpages.debian.org/testing/conntrack/conntrack.8.en.html)

Посмотрим что есть в таблице в момент резолва:

```bash
$ while sleep 1; do conntrack -L -p udp 2>/dev/null | grep 192.168.49.84; done
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 28 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 27 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 26 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 25 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 28 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 27 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 26 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 25 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 28 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 27 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 26 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 25 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 24 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 23 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 22 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 21 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
udp      17 20 src=192.168.49.84 dst=10.96.0.10 sport=58165 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=58165 mark=0 use=1
...
```

**Наблюдения**

- Третий столбец отображает TTL записи в секундах, значение по умолчанию 30с (`net.netfilter.nf_conntrack_udp_timeout`);
- Каждый ретрай “сбрасывает” таймер до изначальных 30 секунд;
- Столбец `[UNREPLIED]` намекает, что между клиент-сервером не происходит обмена сообщениями;
- Все запросы, включая ретраи, падают на один и тот же адрес Пода - `192.168.132.68`. И балансировки не происходит. 

Хотя что мешает по теории вероятности попасть 3 раза на один и тот же Под?

Проверим.

Увеличим кол-во ретраев до 10 и снизим величину таймаутов до 1 секунды прописав в `/etc/resolv.conf`:

```yaml
search default.svc.cluster.local svc.cluster.local cluster.local samokat.io
nameserver 10.96.0.10
options ndots:5
options attempts:10
options timeout:1
```

Запустим резолв вновь:

```yaml
$ tcpdump udp -i calib3c61c3cba9
13:59:25.155250 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:26.154921 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:27.155042 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:28.156243 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:29.156364 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:30.156549 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:31.156658 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:32.156776 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:33.156942 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:34.157061 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)
13:59:35.157206 IP 192.168.49.84.47495 > 10.96.0.10.domain: 25380+ [1au] A? ya.ru. (46)

$ while sleep 1; do conntrack -L -p udp 2>/dev/null | grep 192.168.49.84; done
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 29 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 28 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 27 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 26 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 25 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
udp      17 24 src=192.168.49.84 dst=10.96.0.10 sport=47495 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=47495 mark=0 use=1
```

Страшное подтвердилось - все запросы летят в один и тот же Под **coreDNS** в рамках записи в таблице **conntrack**. 

Давайте докрутим и выставим параметр `net.netfilter.nf_conntrack_udp_timeout`, отвечающий за TTL записи в таблице, в 1с, а таймаут к ретраю в 2с (значение не может быть меньше 1):

```bash
sysctl -w net.netfilter.nf_conntrack_udp_timeout=1
$ cat /etc/resolv.conf | grep timeout
options timeout:2
```

Провернем все заново резолв и посмотрим содержимое **conntrack**:

```bash
$ while sleep 1; do conntrack -L -p udp 2>/dev/null | grep 192.168.49.84; done
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.68 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
udp      17 0 src=192.168.49.84 dst=10.96.0.10 sport=53579 dport=53 [UNREPLIED] src=192.168.132.67 dst=192.168.49.84 sport=53 dport=53579 mark=0 use=1
```

А вот и долгожданная балансировка между Подами - наблюдаем в src как `.67` так и `.68` адреса Подов **coreDNS**.

### Итоги

Кубовая балансировка не так-то проста как кажется и с дефолтными настройками может не дать той надежности, которую от нее ожидаешь.

Стоит ли идти в оптимизацию сетевого стека подобным образом и с какими параметрами еще только предстоит узнать, похожие кейсы в интернетах не были замечены, потому этот момент ждет отдельного исследования.
