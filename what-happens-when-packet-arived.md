# What happens when... или что происходит когда прилетает сетевой пакет

Путь входящего TCP-сегмента с точки зрения OS Linux.

1. Сетевой пакет прибывает на интерфейс сетевой карты (далее **NIC**);
2. **NIC** сохраняет его в **RAM** через механизм **DMA** и помещает указатель на пакет в `RX` очередь **NIC**.

    * `RX` очередь представляет собой [кольцевой буфер (ring buffer)](https://en.wikipedia.org/wiki/Circular_buffer) заранее определенной длины:
        ```bash
        $ ethtool -g eth0
        Ring parameters for eth0:
        Pre-set maximums:
        RX:		8192 # максимально допустимая длина очереди
        RX Mini:	0
        RX Jumbo:	0
        TX:		8192
        Current hardware settings:
        RX:		1024 # текущая длина очереди
        RX Mini:	0
        RX Jumbo:	0
        TX:		1024
        ```
        * `TX` очередь для исходящих сетевых пакетов;
        * через механизм [Receive-Side Scaling (RSS)](https://www.kernel.org/doc/Documentation/networking/scaling.txt) возможно использовать множественные очереди распределяя обработку пакетов среди доступных ядрах CPU. Это "аппаратная фича".

    * Максимально возможное и текущее кол-во очередей:

        ```bash
        $ ethtool -l eth0
        Channel parameters for eth0:
        Pre-set maximums:
        RX:		0
        TX:		0
        Other:		0
        Combined:	12 # максимальное кол-во очередей
        Current hardware settings:
        RX:		0
        TX:		0
        Other:		0
        Combined:	12 # текущее кол-во очередей
        ```
    * Статистика по обработке пакетов на уровне `RX`/`TX` очередей:

        ```bash
        ethtool -S eth0
        NIC statistics:
            rx_queue_0_packets: 103250162
            rx_queue_0_bytes: 8915117437
            rx_queue_0_drops: 0
            ...

        cat /sys/class/net/eth0/statistics/rx_packets
        72972698163

        cat /sys/class/net/eth0/statistics/rx_dropped
        0
        ```

3. **NIC** оповещает систему о поступившем пакете через механизм [Hardware Interrupt Request (IRQ)](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture))
4. Отрабатывает обработчик прерывания;

    Под капотом обработчика реализована минимальная логика:

        1. инкрементировать счетчик hardware прерываний
        2. проверить активизирована ли система по дальнейшей обработке пакетов **softIRQ poll loop** (уже вне контекста обработчика), если нет - запустить ее. 
        3. Выйти из обработчика.
    Иначе из-за максимального приоритета на исполнение с ростом объема трафика система будет то и дело прерываться на процессинг сетевого трафика, негативно влияя на остальную ее часть.

5. **softIRQ poll loop** вычитывает пакеты из `RX-queue` с помощью [`ksoftirqd/X`](https://www.opennet.ru/man.shtml?topic=ksoftirqd&category=9&russian=2) треды.
    Подробнее про механизм ассинхронной обработки пакетов [NAPI](https://en.wikipedia.org/wiki/New_API).
* (если [Receive packet steering (RPS)](https://lwn.net/Articles/362339/) включен:
    * `ksoftirqd/X` тред перекладывает пакет из `RX` в `input_pkt_queue` очередь.
        Емкость `input_pkt_queue` задается в `net.core.netdev_max_backlog`.
    * Пакет вычитывается из `input_pkt_queue` для продвижения вверх по сетевому стеку
6. Пакет передается на обработку сетевым стеком linux
7. TODO Тут будет про L2 OSI
8. TODO Тут будет про L3 OSI
9. Пакет поступает на уровень TCP протокола

    1. если соединение еще не установлено ([3-way handshake](https://www.geeksforgeeks.org/tcp-3-way-handshake-process/))

        * Получая `SYN` (соединение переходит в статус **SYN-RECV**), сервер отправляет в ответ `SYN/ACK` и помещает пакет в `syn-queue`, ее размер задается на уровне системы - `net.ipv4.tcp_max_syn_backlog` (на самом деле нет, подробнее [тут](https://www.alibabacloud.com/blog/599203#:~:text=Maximum%20Length%20Control%20of%20SYN%20Queue)). 
        
            Если очередь переполнена, пакет отбрасывается и TCP стек клиента будет ретрансмитить.

        * Дождавшись завершающего `ACK` от клиента (соединение переходит в статус **ESTABLISHED**) и ядро добавляет пакет в `accept-queue`, где он будет ожидать вычитки через `accept()`.

            Размер очереди высчитывается как `min(somaxconn, backlog)`, где:

                * `somaxconn` - `/proc/sys/net/core/somaxconn`, 4096 по дефолту;
                * `backlog` - параметр при syscall `int listen(int sockfd, int backlog)`.

            Если очередь переполнена - пакет отбрасывается и ожидается ретрансмит `ACK` от клиента.
    
11. Пакет заезжает в **TCP Buffer** установленного соединения;
12. Пакет вычитывается целевым приложением через `read()`.
