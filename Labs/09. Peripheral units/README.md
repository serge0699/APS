# Лабораторная работа 7 "Периферийные устройства"

На прошлой лабораторной работе вы реализовали свой собственный RISC-V процессор. Однако пока что он находится "в вакууме" и никак не связан с внешним миром. Для исправления этого недостатка вами будет реализована системная шина, через которую к процессору смогут подключаться различные периферийные устройства.

## Цель

Интегрировать периферийные устройства в процессорную систему.

---

## Допуск к лабораторной работе

Для успешного выполнения лабораторной работы, вам необходимо:

* ознакомиться с [примером описания модуля-контроллера](../../Basic%20Verilog%20structures/Controllers.md);
* ознакомиться с [описанием](#описание-контроллеров-периферийных-устройств) контроллеров периферийных устройств.

## Ход работы

1. Изучить теорию об адресном пространстве
2. Получить индивидуальный вариант со своим набором периферийных устройств
3. Интегрировать контроллеры периферийных устройств в адресное пространство вашей системы
4. Собрать финальную схему вашей системы
5. Проверить работу системы в ПЛИС с помощью демонстрационного ПО, загружаемого в память инструкций

---

## Теория

Помимо процессора и памяти, третьим ключевым элементом вычислительной системы является система ввода/вывода, обеспечивающая обмен информации между ядром вычислительной машины и периферийными устройствами.

Любое периферийное устройство, со стороны вычислительной машины, представляется набором ячеек памяти (регистров). С помощью чтения и записи этих регистров происходит обмен информации с периферийным устройством, и управление им. Например, датчик температуры может быть реализован самыми разными способами, но для процессора он в любом случае ячейка памяти, из которой он считывает число – температуру.

Система ввода/вывода может быть организована одним из двух способов: с **выделенным адресным пространством** устройств ввода/вывода, или с **совместным адресным пространством**. В первом случае система ввода/вывода имеет отдельную шину для подключения к процессору (и отдельные инструкции для обращения к периферии), во втором – шина для памяти и системы ввода/вывода общая (а обращение к периферии осуществляется теми же инструкциями, что и обращение к памяти).

### Адресное пространство

Архитектура RISC-V подразумевает использование совместного адресного пространства, это значит, что в лабораторной работе будет использована единая шина для подключения памяти и регистров управления периферийными устройствами. При обращении по одному диапазону адресов процессор будет
попадать в память, при обращении по другим – взаимодействовать с регистрами управления/статуса периферийного устройства. Например, можно разделить 32-битное адресное пространство на 256 частей, отдав старшие 8 бит адреса под указание конкретного периферийного устройства. Тогда каждое из периферийных устройств получит 24-битное адресное пространство (16 MiB). Допустим, мы распределили эти части адресного пространства в следующем порядке (от младшего диапазона адресов к старшему):

0. Переключатели
1. Светодиоды
2. Клавиатура PS/2
3. Семисегментные индикаторы
4. UART-приемник
5. UART-передатчик

В таком случае, если мы захотим обратиться в четвертый регистр семисегментных индикаторов, мы должны будем использовать адрес `0x03000004`. Старшие 8 бит (`0x03`) определяют выбранное периферийное устройство, оставшиеся 24 бита определяют конкретный адрес в адресном пространстве этого устройства.

На рисунке ниже представлен способ подключения процессора к памяти инструкций и данных, а так же 255 периферийным устройствам.

![../../.pic/Labs/lab_09_periph/fig_01.drawio.png](../../.pic/Labs/lab_09_periph/fig_01.drawio.png)

### Активация выбранного устройства

В зависимости от интерфейса используемой шины, периферийные устройства либо знают какой диапазон адресов им выделен (например, в интерфейсе I²C), либо нет (интерфейс APB). В первом случае, устройство понимает что к нему обратились непосредственно по адресу в данном обращении, во втором случае — по специальному сигналу.

На приведенной выше схеме используется второй вариант — устройство понимает, что к нему обратились по специальному сигналу `req_i`. Данный сигнал формируется из двух частей: сигнала `req` исходящего из процессорного ядра (сигнал о том, обращение в память вообще происходит) и специального сигнала-селектора исходящего из 256-разрядной шины. Формирование значения на этой шине происходит с помощью [унитарного](https://ru.wikipedia.org/wiki/Унитарный_код) ([one-hot](https://en.wikipedia.org/wiki/One-hot)) кодирования. Процесс кодирования достаточно прост. В любой момент времени на выходной шине должен быть **ровно один** бит, равный единице. Индекс этого бита совпадает со значением старших восьми бит адреса. Поскольку для восьмибитного значения существует 256 комбинаций значений, именно такая разрядность будет на выходе кодировщика. Это означает, что в данной системе можно связать процессор с 256 устройствами (одним из которых будет память данных).

Реализация такого кодирования предельно проста:

* Нулевой сигнал этой шины будет равен единице только если `data_addr_o[31:24] = 8'd0`.
* Первый бит этой шины будет равен единице только если `data_addr_o[31:24] = 8'd1`.
* ...
* Двести пятьдесят пятый бит шины будет равен единице только если `data_addr_o[31:24] = 8'd255`.

Для реализации такого кодирования достаточно выполнить сдвиг влево `255'd1` на значение `data_addr_o[31:24]`.

### Дополнительные правки модуля riscv_unit

Ранее, для того чтобы ваши модули могли работать в ПЛИС, вам предоставлялся специальный модуль верхнего уровня, который выполнял всю работу по связи с периферией через входы и выходы ПЛИС. Поскольку в текущей лабораторной вы завершаете свою процессорную систему, она сама должна оказаться модулем верхнего уровня, а значит здесь вы должны и выполнить все подключение к периферии.

Для этого необходимо добавить в модуль `riscv_unit` дополнительные входы и выходы, которые подключены посредством файла ограничений ([nexys_a7_100t.xdc](nexys_a7_100t.xdc)) к входам и выходам ПЛИС.

```SystemVerilog
module riscv_unit(
  input  logic        clk_i,
  input  logic        resetn_i,

  // Входы и выходы периферии
  input  logic [15:0] sw_i,       // Переключатели
  output logic [15:0] led_o,      // Светодиоды
  input  logic        kclk_i,     // Тактирующий сигнал клавиатуры
  input  logic        kdata_i,    // Сигнал данных клавиатуры
  output logic [ 6:0] hex_led_o,  // Вывод семисегментных индикаторов
  output logic [ 7:0] hex_sel_o,  // Селектор семисегментных индикаторов
  input  logic        rx_i,       // Линия приема по UART
  output logic        tx_o        // Линия передачи по UART
);
//...
endmodule
```

Эти входы порты подключены к одноименным портам ваших контроллеров периферии (речь идет только о реализуемых вами контроллерах, остальные порты должны остаться неподключенными).

Обратите внимание на то, что изменился сигнал сброса (`resetn_i`). Буква `n` на конце означает, что сброс работает по уровню `0` (когда сигнал равен нулю — это сброс, когда единице — не сброс).

Помимо прочего, необходимо подключить к вашему модулю `блок делителя частоты`. Поскольку в данном курсе лабораторных работ вы выполняли реализацию однотактного процессора, инструкция должна пройти через все ваши блоки за один такт. Из-за этого критический путь вашей схемы не позволит использовать тактовый сигнал частотой в `100 МГц`, от которого работает отладочный стенд. Поэтому, необходимо создать отдельный сигнал с пониженной тактовой частотой, от которого будет работать ваша схема.

Для этого необходимо:

1. Подключить файл [`sys_clk_rst_gen.v`](sys_clk_rst_gen.v) в ваш проект.
2. Подключить этот модуль внутри `riscv_unit` следующим образом:

```SystemVerilog
logic sysclk, rst;
sys_clk_rst_gen divider(.ex_clk_i(clk_i),.ex_areset_n_i(resetn_i),.div_i(10),.sys_clk_o(sysclk), .sys_reset_o(rst));
```

3. После вставки данных строк в начало описания модуля `riscv_unit` вы получите тактовый сигнал `sysclk` с частотой в 10 МГц и сигнал сброса `rst` с активным уровнем `1` (как и в предыдущих лабораторных). Все ваши внутренние модули (`riscv_core`, `data_mem` и `контроллеры периферии`) должны работать от тактового сигнала `sysclk`. На модули, имеющие входной сигнал сброса (`rst_i`) необходимо подать ваш сигнал `rst`.

---

## Задание

В рамках данной лабораторной работы необходимо реализовать модули-контроллеры двух периферийных устройств, реализующих управление в соответствии с приведенной ниже картой памяти и встроить их в процессорную систему, используя используя схему приведенную выше. На карте приведено шесть периферийных устройств, вам необходимо взять только два из них в зависимости от вашего варианта индивидуального задания (ИЗ) по следующему правилу:

1. Те кому достались варианты ИЗ 1-5 должны реализовать контроллеры переключателей и светодиодов.
2. Те кому достались варианты ИЗ 6-15 должны реализовать контроллеры клавиатуры PS/2 и семисегментных индикаторов.
3. Те кому достались варианты BP 16-29 должны реализовать контроллеры приемника и передатчика UART.

![Карта памяти](../../.pic/Labs/riscv_periph_memory_map.png)

Работа с картой осуществляется следующим образом. Под названием каждого периферийного устройства указана старшая часть адреса (чему должны быть равны старшие 8 бит адреса, чтобы было сформировано обращение к данному периферийному устройству). Например, для переключателей это значение равно `0x01`, для светодиодов `0x02` и т.п.
В самом левом столбце указаны используемые/неиспользуемые адреса в адресном пространстве данного периферийного устройства. Например для переключателей есть только один используемый адрес: `0x000000`. Его функциональное назначение и разрешения на доступ указаны в столбце соответствующего периферийного устройства. Возвращаясь к адресу `0x000000` для переключателей мы видим следующее:

* (R) означает что разрешен доступ только на чтение (операция записи по этому адресу должна игнорироваться вашим контроллером).
* "Выставленное на переключателях значение" означает ровно то что и означает. Если процессор выполняет операцию чтения по адресу 0x01000000 (`0x01` (старшая часть адреса переключателей) + `0x000000` (младшая часть адреса для получения выставленного на переключателях значения) ), то контроллер должен выставить на выходной сигнал RD значение на переключателях (о том как получить это значение будет рассказано чуть позже).

Рассмотрим еще один пример. При обращении по адресу `0x02000024` (`0x02` (старшая часть адреса контроллера светодиодов) + `0x000024` (младшая часть адреса для доступа на запись к регистру сброса) ) должна произойти запись в регистр сброса, который должен сбросить значения в регистре управления зажигаемых светодиодов и регистре управления режимом "моргания" светодиодов (подробнее о том как должны работать эти регистры будет ниже).

Таким образом, каждый контроллер периферийного устройства должен выполнять две вещи:

1. При получении сигнала `req_i`, записать в регистр или вернуть значение из регистра, ассоциированного с переданным адресом (адрес передается с обнуленной старшей частью). Если регистра, ассоциированного с таким адресом нет (например для переключателей не ассоциировано ни одного адреса кроме `0x000000`), игнорировать эту операцию.
2. Выполнять управление периферийным устройством с помощью управляющих регистров.

Подробное описание периферийных устройств их управления и назначение управляющих регистров будет дано после порядка выполнения задания.

---

## Порядок выполнения задания

1. Внимательно ознакомьтесь с [примером описания модуля контроллера](../../Basic%20Verilog%20structures/Controllers.md).
2. Внимательно ознакомьтесь со спецификацией контроллеров периферии своего варианта. В случае возникновения вопросов, проконсультируйтесь с преподавателем.
3. Реализуйте модули контроллеров периферии. Имена модулей и их порты будут указаны в [описании контроллеров](#описание-контроллеров-периферийных-устройств). Пример разработки контроллера приведен [здесь](../../Basic%20Verilog%20structures/Controllers.md).
4. Обновите модуль `riscv_unit` в соответствии с разделом ["Дополнительные правки модуля riscv_unit"](#дополнительные-правки-модуля-riscv_unit).
   1. Подключите в проект файл `sys_clk_rst_gen.sv`.
   2. Добавьте в модуль `riscv_unit` входы и выходы периферии.
   3. Подключите к модулю `riscv_unit` модуль `sys_clk_rst_gen` скопировав приведенный фрагмент кода.
   4. Замените подключение тактового сигнала исходных подмодулей `riscv_unit` на появившийся сигнал `sysclk`. Убедитесь, что на модули имеющие сигнал сброса приходит сигнал `rst`.
5. Интегрируйте модули контроллеров периферии в процессорную систему по приведенной схеме руководствуясь старшими адресами контроллеров, представленными на карте памяти. Это означает, что если вы реализуете контроллер светодиодов, на его входов `req_i` должна подаваться единица в случае если `mem_req_o == 1` и старшие 8 бит адреса равны `0x02`.
   1. При интеграции вы должны подключить только модули-контроллеры вашего варианта. Контроллеры периферии других вариантов подключать не надо.
   2. При этом во время интеграции, вы должны использовать старшую часть адреса, представленную в карте памяти для формирования сигнала `req_i` для ваших модулей-контроллеров.
   3. Даже если вы не используете какие-то входные/выходные сигналы в модуле `riscv_unit` (например по варианту вам не достался контроллер клавиатуры и поэтому вы не используете сигналы `kclk_i` и `kdata_i`), вы все равно должны их описать во входах и выходах модуля `riscv_unit`.
6. Проинициализируйте память инструкций с помощью предоставленной для каждой пары контроллеров программы.
7. Подключите к проекту файл ограничений ([nexys_a7_100t.xdc](nexys_a7_100t.xdc)), если тот еще не был подключен, либо замените его содержимое данными из файла к этой лабораторной работе.
8. Проверьте работу вашей процессорной системы с помощью отладочного стенда с ПЛИС и (при соответствующем варианте) клавиатуры/рабочего компьютера.
   1. Обратите внимание, что в данной лабораторной уже не будет модуля верхнего уровня `nexys_...`, так как ваш модуль процессорной системы уже полностью самостоятелен и взаимодействует непосредственно с ножками ПЛИС через модули, управляемые контроллерами периферии.

---

## Описание контроллеров периферийных устройств

Для того, чтобы избежать избыточности в контексте описания контроллеров периферийных устройств будет использоваться два термина:

1. Под "**запросом на запись** по адресу `0xАДРЕС`" будет пониматься совокупность следующих условий:
   1. Происходит восходящий фронт `clk_i`.
   2. На входе `req_i` выставлено значение `1`.
   3. На входе `write_enable_i` выставлено значение `1`.
   4. На входе `addr_i` выставлено значение `0xАДРЕС`
2. Под "**запросом на чтение** по адресу `0xАДРЕС`" будет пониматься совокупность следующих условий:
   1. На входе `req_i` выставлено значение `1`.
   2. На входе `write_enable_i` выставлено значение `0`.
   3. На входе `addr_i` выставлено значение `0xАДРЕС`

Обратите внимание на то, что **запрос на чтение** должен обрабатываться **синхронно** (выходные данные должны выдаваться по положительному фронту `clk_i`).

При описании поддерживаемых режимов доступа по данному адресу используется интуитивно понятное обозначение:

* R — доступ **только на чтение**;
* W — доступ **только на запись**;
* RW — доступ на **чтение и запись**.

В случае отсутствия **запроса на чтения**, на выход `read_data_o` должно подаваться значение `32'hfa11_1eaf`. Это никак не повлияет на работу процессора, но будет удобно в процессе отладки на временной диаграмме (тоже самое было сделано в процессе разработки памяти данных).

Если пришел **запрос на запись** или **чтение**, это еще не значит, что контроллер должен его выполнить. В случае, если запрос происходит по адресу, не поддерживающему этот запрос (например **запрос на запись** по адресу поддерживающему только чтение или наоборот), данный запрос должен игнорироваться, а на выходе `read_data_o` должно появиться значение `32'hdead_beef`.

К примеру, в случае запроса на чтение по адресу `0x0100004` (четвертый байт в адресном пространстве периферийного устройства "переключатели"), на выходе `read_data_o` должно оказаться значение `32'hdead_beef`. В случае отсутствия запроса на чтение (`req_i == 0` или `write_enable_i == 1`), на выходе `read_data_o` контроллера переключателей должно оказаться значение `32'hfa11_1eaf`.

В случае осуществления записи по принятому запросу, необходимо записать данные с сигнала `write_data_i` в регистр, ассоциированный с адресом `addr_i` (если разрядность регистра меньше разрядности сигнала `write_data_i`, старшие биты записываемых данных отбрасываются).

В случае осуществления чтения по принятому запросу, необходимо по положительному фронту `clk_i` выставить данные с сигнала, ассоциированного с адресом `addr_i` на выходной сигнал `read_data_o` (если разрядность сигнала меньше разрядности выходного сигнала `read_data_o`, возвращаемые данные должны дополниться нулями в старших битах).

### Переключатели

Переключатели являются простейшим устройством ввода на отладочном стенде `Nexys A7`. Соответственно и контроллер, осуществляющий доступ процессора к ним так же будет очень простым. Рассмотрим прототип модуля, который вам необходимо реализовать:

```SystemVerilog
module sw_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic        clk_i,
  input  logic        rst_i
  input  logic        req_i,
  input  logic        write_enable_i,
  input  logic [31:0] addr_i,
  input  logic [31:0] write_data_i,  // не используется, добавлен для
                                     // совместимости с системной шиной
  output logic [31:0] read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение к периферии
*/
  input logic [15:0]  sw_i
);

endmodule
```

По сути, логика работы контроллера сводится к тому, выдавать на шину `read_data_o` данные со входа `sw_i` каждый раз, когда приходит **запрос на чтение** по нулевому адресу. Поскольку разрядность `sw_i` в два раза меньше разрядности выхода `read_data_o` его старшие биты необходимо дополнить нулями.

Адресное пространство контроллера:

|Адрес|Режим доступа|           Функциональное назначение             |
|-----|-------------|-------------------------------------------------|
|0x00 | R           | Чтение значения, выставленного на переключателях|

### Светодиоды

Как и переключатели, светодиоды являются простейшим устройством вывода. Поэтому, чтобы задание было интересней, для их управления был добавлен регистр, управляющий режимом вывода данных на светодиоды.
Рассмотрим прототип модуля, который вам необходимо реализовать:

```SystemVerilog
module led_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic        clk_i,
  input  logic        rst_i
  input  logic        req_i,
  input  logic        write_enable_i,
  input  logic [31:0] addr_i,
  input  logic [31:0] write_data_i,
  output logic [31:0] read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение к периферии
*/
  output logic [15:0]  led_o
);

logic [15:0]  led_val;
logic         led_mode;

endmodule
```

Данный модуль должен выводить на выходной сигнал `led_o` данные с регистра `led_val`. Запись и чтение регистра `led_val` осуществляется по адресу `0x00`. Запись любого значения, превышающего `2¹⁶-1` должна игнорироваться.

Регистр `led_mode` отвечает за режим вывода данных на светодиоды. Когда этот регистр равен единице, светодиоды должны "моргать" выводимым значением. Под морганием подразумевается вывод значения из регистра `led_val` на выход `led_o` на одну секунду (загорится часть светодиодов, соответствующие которым биты шины `led_o` равны единице), после чего на одну секунду выход `led_o` необходимо подать нули. Запись и чтение регистра `led_mode` осуществляется по адресу `0x04`. Запись любого значения, отличного от `0` и `1` должна игнорироваться.

Отсчет времени можно реализовать простейшим счетчиком, каждый такт увеличивающимся на 1 и сбрасывающимся по достижении определенного значения, чтобы продолжить считать с нуля. Зная тактовую частоту, нетрудно определить до скольки должен считать счетчик. При тактовой частоте в 10 МГц происходит 10 миллионов тактов в секунду. Это означает, что при такой тактовой частоте через секунду счетчик будет равен `10⁷-1` (счет идет с нуля).

Обратите внимание на то, что адрес `0x24` является адресом сброса. В случае записи по этому адресу единицы вы должны сбросить регистры `led_val`, `led_mode` и все вспомогательные регистры, которые вы создали. Для реализации сброса вы можете как создать отдельный регистр `led_rst`, в который будет происходить запись, а сам сброс будет происходить по появлению единицы в этом регистре (в этом случае необходимо не забыть сбрасывать и этот регистр), так и создать обычный провод, формирующий единицу в случае выполнения всех указанных условий (условий запроса на запись, адреса сброса и значения записываемых данных равному единице).

Адресное пространство контроллера:

|Адрес|Режим доступа|Допустимые значения|                       Функциональное назначение                                   |
|-----|-------------|-------------------|-----------------------------------------------------------------------------------|
|0x00 | RW          | [0:65535]         | Чтение и запись в регистр `led_val` отвечающий за вывод данных на светодиоды      |
|0x04 | RW          | [0:1]             | Чтение и запись в регистр `led_mode`, отвечающий за режим "моргания" светодиодами |
|0x24 | W           |  1                | Запись сигнала сброса                                                             |

### Клавиатура PS/2

Клавиатура [PS/2](https://ru.wikipedia.org/wiki/PS/2_(порт)) осуществляет передачу [скан-кодов](https://ru.wikipedia.org/wiki/Скан-код), нажатых на этой клавиатуре клавиш.

В рамках данной лабораторной работы вам будет предоставлен модуль, осуществляющий прием данных с клавиатуры. Вам нужно написать лишь модуль, осуществляющий контроль предоставленным модулем. У предоставленного модуля будет следующий прототип:

```SystemVerilog
module PS2Receiver(
    input         clk_i,          // Сигнал тактирования процессора и вашего модуля-контроллера
    input         kclk_i,         // Тактовый сигнал, приходящий с клавиатуры
    input         kdata_i,        // Сигнал данных, приходящий с клавиатуры
    output [15:0] keycode_o,      // Сигнал полученного с клавиатуры скан-кода клавиши
    output        keycode_valid_o // Сигнал готовности данных на выходе keycodeout
    );
endmodule
```

Вам необходимо реализовать модуль-контроллер со следующим прототипом:

```SystemVerilog
module ps2_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic         clk_i,
  input  logic         rst_i,
  input  logic [31:0]  addr_i,
  input  logic         req_i,
  input  logic [31:0]  write_data_i,
  input  logic         write_enable_i,
  output logic [31:0]  read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение к модулю,
    осуществляющему прием данных с клавиатуры
*/
  input  logic kclk_i,
  input  logic kdata_i
);

logic [7:0] scan_code;
logic       scan_code_is_unread;

endmodule
```

В первую очередь, вы должны инстанциировать модуль `PS2Receiver` внутри вашего модуля-контроллера, соединив соответствующие входы. Для подключения к выходам необходимо создать дополнительные провода.

По каждому восходящему фронту сигнала `clk_i` вы должны проверять выход `keycode_valid_o` и если тот равен единице, записать значение с выхода `keycode_o` в регистр `scan_code`. При этом значение регистра `scan_code_is_unread` необходимо выставить в единицу.

В случае, если произошел запрос на чтение по адресу `0x00`, необходимо выставить на выход `read_data_o` значение регистра `scan_code` (дополнив старшие биты нулями), при этом значение регистра `scan_code_is_unread` необходимо обнулить.

В случае запроса на чтение по адресу `0x04` необходимо вернуть значение регистра `scan_code_is_unread`.

В случае запроса на запись по адресу `0x24` со значением `1`, необходимо осуществить сброс регистров `scan_code` и `scan_code_is_unread` в `0`.

Адресное пространство контроллера:

|Адрес|Режим доступа|Допустимые значения|                       Функциональное назначение                                                                   |
|-----|-------------|-------------------|-------------------------------------------------------------------------------------------------------------------|
|0x00 | R           | [0:255]           | Чтение из регистра `scan_code`, хранящего скан-код нажатой клавиши                                                |
|0x04 | R           | [0:1]             | Чтение из регистра `scan_code_is_unread`, сообщающего о том, что есть непрочитанные данные в регистре `scan_code` |
|0x24 | W           |  1                | Запись сигнала сброса                                                                                             |

### Семисегментные индикаторы

Семисегментные индикаторы позволяют выводить арабские цифры и первые шесть букв латинского алфавита, тем самым позволяя отображать шестнадцатиричные цифры. На отладочном стенде `Nexys A7` размещено восемь семисегментных индикаторов. Для вывода цифр на эти индикаторы, вам будет предоставлен модуль `hex_digits`, вам нужно лишь написать модуль, осуществляющий контроль над ним. Прототип модуля `hex_digits` следующий:

```SystemVerilog
module hex_digits(
  input  logic       clk_i,
  input  logic       rst_i,
  input  logic [3:0] hex0_i,    // Цифра, выводимой на нулевой (самый правый) индикатор
  input  logic [3:0] hex1_i,    // Цифра, выводимая на первый индикатор
  input  logic [3:0] hex2_i,    // Цифра, выводимая на второй индикатор
  input  logic [3:0] hex3_i,    // Цифра, выводимая на третий индикатор
  input  logic [3:0] hex4_i,    // Цифра, выводимая на четвертый индикатор
  input  logic [3:0] hex5_i,    // Цифра, выводимая на пятый индикатор
  input  logic [3:0] hex6_i,    // Цифра, выводимая на шестой индикатор
  input  logic [3:0] hex7_i,    // Цифра, выводимая на седьмой индикатор
  input  logic [7:0] bitmask_i, // Битовая маска для включения/отключения
                                // отдельных индикаторов

  output logic [6:0] hex_led_o  // Сигнал, контролирующий каждый отдельный
                                // светодиод индикатора
  output logic [7:0] hex_sel_o  // Сигнал, указывающий на какой индикатор
                                // выставляется hex_led
);
endmodule
```

Для того, чтобы вывести шестнадцатеричную цифру на любой из индикаторов, необходимо выставить двоичное представление этой цифры на соответствующий вход `hex0-hex7`.

За включение/отключение индикаторов отвечает входной сигнал `bitmask_i`, состоящий из 8 бит, каждый из которых включает/отключает соответствующий индикатор. Например, при `bitmask_i == 8'b0000_0101`, включены будут нулевой и второй индикаторы, остальные будут погашены.

Выходные сигналы `hex_led` и `hex_sel` необходимо просто подключить к соответствующим выходным сигналам модуля-контроллера. Они пойдут на выходы ПЛИС, соединенные с семисегментными индикаторами.

Для управления данным модулем, необходимо написать модуль-контроллер со следующим прототипом:

```SystemVerilog
module hex_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic        clk_i,
  input  logic [31:0] addr_i,
  input  logic        req_i,
  input  logic [31:0] write_data_i,
  input  logic        write_enable_i,
  output logic [31:0] read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение к модулю,
    осуществляющему вывод цифр на семисегментные индикаторы
*/
  output logic [6:0] hex_led,
  output logic [7:0] hex_sel
);

  logic [3:0] hex0, hex1, hex2, hex3, hex4, hex5, hex6, hex7;
  logic [7:0] bitmask;
endmodule
```

Регистры `hex0-hex7` отвечают за вывод цифры на соответствующий семисегментный индикатор. Регистр `bitmask` отвечает за включение/отключение семисегментных индикаторов. Когда в регистре `bitmask` бит, индекс которого совпадает с номером индикатора равен единице — тот включен и выводит число, совпадающее со значением в соответствующем регистре `hex0-hex7`. Когда бит равен нулю — этот индикатор гаснет.

Доступ на чтение/запись регистров `hex0-hex7` осуществляется по адресам `0x00-0x1c` (см. таблицу адресного пространства).

Доступ на чтение/запись регистра `bitmask` осуществляется по адресу `0x20`.

При запросе на запись единицы по адресу `0x24` необходимо выполнить сброс всех регистров. При этом регистр `bitmask` должен сброситься в значение `0xFF`.

Адресное пространство контроллера:

|Адрес|Режим доступа|Допустимые значения|                 Функциональное назначение               |
|-----|-------------|-------------------|---------------------------------------------------------|
|0x00 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex0           |
|0x04 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex1           |
|0x08 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex2           |
|0x0C | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex3           |
|0x10 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex4           |
|0x14 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex5           |
|0x18 | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex6           |
|0x1C | RW          | [0:31]            | Регистр, хранящий значение, выводимое на hex7           |
|0x20 | RW          | [0:255]           | Регистр, управляющий включением/отключением индикаторов |
|0x24 | W           |  1                | Запись сигнала сброса                                   |

### UART

[UART](https://ru.wikipedia.org/wiki/Универсальный_асинхронный_приёмопередатчик) — это последовательный интерфейс, использующий для приема и передачи данных по одной независимой линии с поддержкой контроля целостности данных.

Для того, чтобы передача данных была успешно осуществлена, приемник и передатчик на обоих концах одного провода должны договориться о параметрах передачи:

* её скорости (бодрейт);
* контроля целостности данных (использование бита четности/нечетности/отсутствие контроля);
* длины стопового бита.

Вам будут предоставлены модули, осуществляющие прием и передачу данных по этому интерфейсу, от вас лишь требуется написать модули, осуществляющие управление предоставленными модулями.

```SystemVerilog
module uart_rx (
  input  logic            clk_i,      // Тактирующий сигнал
  input  logic            rst_i,      // Сигнал сброса
  input  logic            rx_i,       // Сигнал линии, подключенной к выводу ПЛИС,
                                      // по которой будут приниматься данные
  output logic            busy_o,     // Сигнал о том, что модуль занят приемом данных
  input  logic [15:0]     baudrate_i, // Настройка скорости передачи данных
  input  logic            parity_en_i,// Настройка контроля целостности через бит четности
  input  logic            stopbit_i,  // Настройка длины стопового бита
  output logic [7:0]      rx_data_o,  // Принятые данные
  output logic            rx_valid_o  // Сигнал о том, что прием данных завершен

);
endmodule
```

```SystemVerilog
module uart_tx (
  input  logic            clk_i,      // Тактирующий сигнал
  input  logic            rst_i,      // Сигнал сброса
  output logic            tx_o,       // Сигнал линии, подключенной к выводу ПЛИС,
                                      // по которой будут отправляться данные
  output logic            busy_o,     // Сигнал о том, что модуль занят передачей данных
  input  logic [15:0]     baudrate_i, // Настройка скорости передачи данных
  input  logic            parity_en_i,// Настройка контроля целостности через бит четности
  input  logic            stopbit_i,  // Настройка длины стопового бита
  input  logic [7:0]      tx_data_i,  // Отправляемые данные
  input  logic            tx_valid_i  // Сигнал о старте передачи данных
);
endmodule
```

Для управления этими модулями вам необходимо написать два модуля-контроллера со следующими прототипами

```SystemVerilog
module uart_rx_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic          clk_i,
  input  logic          rst_i
  input  logic [31:0]   addr_i,
  input  logic          req_i,
  input  logic [31:0]   write_data_i,
  input  logic          write_enable_i,
  output logic [31:0]   read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение передающему,
    входные данные по UART
*/
  input  logic          rx_i
);

  logic busy;
  logic [15:0] baudrate;
  logic parity_en;
  logic stopbit;
  logic data;
  logic valid;

endmodule
```

```SystemVerilog
module uart_tx_sb_ctrl(
/*
    Часть интерфейса модуля, отвечающая за подключение к системной шине
*/
  input  logic          clk_i,
  input  logic          rst_i
  input  logic [31:0]   addr_i,
  input  logic          req_i,
  input  logic [31:0]   write_data_i,
  input  logic          write_enable_i,
  output logic [31:0]   read_data_o,

/*
    Часть интерфейса модуля, отвечающая за подключение передающему,
    выходные данные по UART
*/
  output logic          tx_o
);

  logic busy;
  logic [15:0] baudrate;
  logic parity_en;
  logic stopbit;

endmodule
```

У обоих предоставленных модулей схожий прототип, различия заключаются лишь в направлениях некоторых сигналов:

Управление сигналами этого модуля достаточно просто.

Сигналы `clk_i` и `rx_i`/`tx_i` подключаются напрямую к соответствующим сигналам модуля-контроллера.

Сигнал `rst_i` модулей `uart_rx` / `uart_tx` должен быть равен единице при запросе на запись единицы по адресу `0x24`, а так же в случае когда сигнал `rst_i` модуля-контроллера равен единице.

Выходной сигнал `busy_o` на каждом такте `clk_i` должен записываться в регистр `busy`, доступ на чтение к которому осуществляется по адресу `0x08`.

Значение входных сигналов `baudrate_i`, `parity_en_i`, `stopbit_i` берутся из соответствующих регистров, доступ на запись к которым осуществляется по адресам `0x0C`, `0x10`, `0x14` соответственно, но только в моменты, когда выходной сигнал `busy_o` равен нулю. Иными словами, изменение настроек передачи возможно только в моменты, когда передача не происходит. Доступ на чтение этих регистров может осуществляться в любой момент времени.

В регистр `data` модуля `uart_rx_sb_ctrl` записывается значение одноименного выхода модуля `uart_rx` в моменты положительного фронта `clk_i`, когда сигнал `rx_valid_o` равен единице. Доступ на чтение этого регистра осуществляется по адресу `0x00`.

В регистр `valid` модуля `uart_rx_sb_ctrl` записывается единица по положительному фронту clk_i, когда выход `rx_valid_o` равен единице. Данный регистр сбрасывается в ноль при выполнении запроса на чтение по адресу `0x00`. Сам регистр доступен для чтения по адресу `0x04`.

На вход `tx_data_i` модуля `uart_tx` подаются данные из регистра `data` модуля `uart_tx_sb_ctrl`. Доступ на запись в этот регистр происходит по адресу `0x00` в моменты положительного фронта `clk_i`, когда сигнал `busy_o` равен нулю. Доступ на чтение этого регистра может осуществляться в любой момент времени.

На вход `tx_valid_i` модуля `uart_tx` подается единица в момент выполнения запроса на запись по адресу `0x00` (при сигнале `busy` равном нулю). В остальное время на вход этого сигнала подается `0`.

В случае запроса на запись единицы по адресу `0x24` (адресу сброса), все регистры модуля-контроллера должны сброситься. При этом регистр `baudrate` должен принять значение `9600`, регистр `parity` должен принять значение `1`, регистр, `stopbit` должен принять значение `1`. Остальные регистры должны принять значение `0`.

Адресное пространство контроллера `uart_rx_sb_ctrl`:

|Адрес|Режим доступа|Допустимые значения|                 Функциональное назначение                                                               |
|-----|-------------|-------------------|---------------------------------------------------------------------------------------------------------|
|0x00 | R           | [0:255]           | Чтение из регистра `data`, хранящего значение принятых данных                                           |
|0x04 | R           | [0:1]             | Чтение из регистра `valid`, сообщающего о том, что есть непрочитанные данные в регистре `data`          |
|0x08 | R           | [0:1]             | Чтение из регистра `busy`, сообщающего о том, что модуль находится в процессе приема данных             |
|0x0C | RW          | [0:65535]         | Чтение/запись регистра `baudrate`, отвечающего за скорость передачи данных                              |
|0x10 | RW          | [0:1]             | Чтение/запись регистра `parity`, отвечающего за включение отключение проверки данных через бит четности |
|0x14 | RW          | [0:1]             | Чтение/запись регистра `stopbit`, отвечающего за длину стопового бита                                   |
|0x24 | W           |  1                | Запись сигнала сброса                                                                                   |

Адресное пространство контроллера `uart_tx_sb_ctrl`:

|Адрес|Режим доступа|Допустимые значения|                 Функциональное назначение                                                               |
|-----|-------------|-------------------|---------------------------------------------------------------------------------------------------------|
|0x00 | RW          | [0:255]           | Чтение и запись регистра `data`, хранящего значение отправляемых данных                                 |
|0x08 | R           | [0:1]             | Чтение из регистра `busy`, сообщающего о том, что модуль находится в процессе передачи данных           |
|0x0C | RW          | [0:65535]         | Чтение/запись регистра `baudrate`, отвечающего за скорость передачи данных                              |
|0x10 | RW          | [0:1]             | Чтение/запись регистра `parity`, отвечающего за включение отключение проверки данных через бит четности |
|0x14 | RW          | [0:1]             | Чтение/запись регистра `stopbit`, отвечающего за длину стопового бита                                   |
|0x24 | W           |  1                | Запись сигнала сброса                                                                                   |