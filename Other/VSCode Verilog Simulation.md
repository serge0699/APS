# Руководство по интеграции Icarus Verilog в VSCode

Данное руководство позволяет интегрировать бесплатный Verilog-симулятор с редактором VSCode. После выполнения всех этапов, будет возможно запускать симуляцию прямо из редактора, а затем просматривать временную диаграмму (в ограниченном режиме, об этом ниже).

Пример использования:

![../.pic/Other/VSCode%20Verilog%20Simulation/icarus_verilog_integration.png](../.pic/Other/VSCode%20Verilog%20Simulation/icarus_verilog_integration.png)

Для получения данного результата, необходимо выполнить следующие шаги:

## Установка ПО

1. Скачать и установить [Icarus Verilog](https://bleyer.org/icarus/) (при установке убедитесь, что стоят галочки `Install GTKWave (x64)`, `Add executable folder(s) to the user PATH`)
2. Скачать, установить и запустить [Visual Studio Code](https://code.visualstudio.com/Download)
3. После запуска VSCode, необходимо перейти на вкладку "Расширения" (можно открыть через `Ctrl+Shift+X`)
4. Устанавливаем следующие расширения:
   1. `Verilog-HDL/SystemVerilog/Bluespec SystemVerilog support for VS Code`
   2. `Verilog Testbench Runner`
   3. `WaveTrace`

Расширение `Bluespec SystemVerilog support for VS Code` добавляет подсветку синтаксиса языка Verilog.  
Расширение `Verilog Testbench Runner` добавляет на панель Verilog-файла кнопку `▶`, позволяющую запустить симуляцию без вызова команд `Icarus Verilog` в терминале.  
Расширение `WaveTrace` позволяет просматривать временные диаграммы прямо в VSCode.

На этом все настройки завершены.

## Изменение Verilog-файлов

Для того, чтобы при компиляции тестбенча, подхватывались файлы описанных модулей, необходимо в самый верх тестбенча дописать директиву (см. пример использования, 1):

```Verilog
`include "имя файла, содержащего тестируемый модуль"
```

А внутрь модуля `testbench` необходимо добавить блок:

```Verilog
initial begin
  $dumpfile("tb.vcd");
  $dumpvars(0, fulladder32_tb); // fulladder32_tb <- имя модуля тестбенча
end
```

(см. пример использования, 2)

---

Кроме того, необходимо убедиться, что в модуле тестбенча нет системных функций: `$stop` (проверьте через поиск, если есть — удалите). Если в процессе симуляции произойдет вызов этой системной функции, её не удастся продолжить (в отличие от Vivado).

---

После этого нажимаем на зеленую кнопку `▶` (пример использования, 3), в логе будут либо сообщения об ошибках компиляции, либо результатах симуляции (пример использования, 4).  
Если симуляция будет завершена успешно, в подпапке build появится файл с расширением `.vcd` (пример использования, 5). При открытии этого файла, появится вкладка расширения `WaveTrace` (вкладку можно расположить в соседней панели, перетянув ее).  
Чтобы увидеть временную диаграмму, необходимо добавить отображаемые сигналы (пример использования, 6). Ограничение бесплатной версии `WaveTrace` заключается в том, что можно отобразить изменения только восьми сигналов одновременно (если понадобятся еще сигналы, придется сперва удалить какие-то предыдущие).  
После всех этих действий вы сможете увидеть временную диаграмму (пример использования, 7).

## Как запустить симуляцию с предоставленными мне файлами?

Если вы пропустили, или не сделали какую-то из лаб, вам потребуется взять готовые модули из ветки [Я-не-смог](https://github.com/MPSU/APS/tree/%D0%AF-%D0%BD%D0%B5-%D1%81%D0%BC%D0%BE%D0%B3). Модули в этих ветках являются нетлистами (описанием модуля на языке Verilog, полученным [после этапа синтеза](../Vivado%20Basics/Implementation%20steps.md)).  
Для того, чтобы симулятор Icarus Verilog мог работать с этим файлом, необходимо предоставить ему библиотеку примитивов из которых был собран нетлист. Для этого, необходимо выполнить следующие шаги:

1. [Скачать](../../Я-не-смог/unisims.zip) библиотеку примитивов.
2. Распаковать её в папку вашего проекта.
3. Добавить еще одну директиву `ˋinclude` в тестбенч: `ˋinclude "glbl.v"`
4. Добавить в настройки расширения `Verilog Testbench Runner` указание использовать библиотеку `unisims`. Для этого:
   1. Откройте `Файл->Настройки->Параметры` (`File->Preferences->Settings`).
   2. В поле поиска введите: `verilog.icarusCompileArguments`.
   3. В поле найденной опции введите `-y unisims/`.

### Что делать, если мне нужно отобразить больше, чем 8 сигналов?

В этом случае, необходимо воспользоваться внешней программой по отображению временных диаграмм: `GTK Wave`. Она должна была быть установлена вами на этапе установке `Icarus Verilog` (см. [Установка ПО](#установка-по)).

В теории, после завершения симуляции, расширение `Verilog Testbench Runner`
должно предлагать запустить GTK Wave. Если этого не происходит, выполните команду в терминале VSCode:

```bash
gtkwave build/tb.vcd
```

где `tb.vcd` имя временной диаграммы, которое вы указали в блоке `initial` (см. [Изменение Verilog-файлов](#изменение-verilog-файлов)).  
Откроется окно GTK Wave. Внутри этого окна, слева, есть вкладка `SST`, где будет расположен модуль вашего тестбенча. Нажав на кнопку `+` слева от имени модуля вы увидите объект `DUT` (имя сущности тестируемого модуля). Если нажать по этому объекту `ПКМ -> Recurse Import -> Append`, вы добавите все внутренние сигналы этого модуля в область просмотра временной диаграммы.