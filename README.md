# UKNC_upgrades
Hardware extensions for UKNC (Soviet PDP-11 compatible machine)

Коллекция аппаратных дополнений для ПК [Электроника МС 0511(УКНЦ)](https://ru.wikipedia.org/wiki/%D0%AD%D0%BB%D0%B5%D0%BA%D1%82%D1%80%D0%BE%D0%BD%D0%B8%D0%BA%D0%B0_%D0%9C%D0%A1_0511).

**[mc0511_05.pdf](http://micklab.ru/MC0511.htm)** - схема самого ПК, взято с сайта [micklab.ru](http://micklab.ru/MC0511.htm)

## ay_and_dac_vm1
Реализация подключеня звукогенератора AY-3-8910 и двухканального ЦАП'а.

[Тема](http://zx-pk.ru/threads/29020-zvukovoj-kontroller-dlya-uknts.html) на форуме zx-pk.ru где обсуждалась данная реализация.
## sram_cpu_two_windows
[Модуль SRAM.](https://zx-pk.ru/threads/29492-alternativnyj-form-faktor-dlya-uknts-rasshirenie/)

После того, как я неожиданно заметил, что у ЦП свободно 7.5КБ адресного пространства (при его ширине в 64КБ!), было решено немедленно заполнить их дополнительной памятью. В результате возникла данная реализация.

На модуле установлен 1МБ памяти. Доступ к ним производится через два окна `0160000-0167777` и `0170000-0175777`, т.е. первое окно 4КБ, второе 3КБ. Через каждое окно можно получить доступ ко всему объему памяти, но в верхнем окне доступы только нижние 3КБ каждой страницы.

Переключение страниц осуществляется записью в регистр по адресу `0176000`, младший байт переключает страницу в первом, 4-х килобайтном окне, старший во втором.

По сигналу сброс, регистры страниц обнуляются, но т.к. регистр верхнего окна инверсный, получается что отсчёт страниц идёт с противоположных границ 1МБ-айтного диапазона. Т.е. после записи в один регистр 0, а в другой 255, в оба окна будет подставлена одна и таже страница.

Несмотря на то что RT-11 может использовать дополнительную память (по умолчанию верхняя граница `0170000`), драйвера устройств подключенных к ПП не работают(`MZ` и `WD`), из за того что со стороны ПП, адреса ОЗУ ЦП выше `0160000` это ОЗУ режима HALT.

Для использования таких драйверов необходимо установить вехний адрес RT-11 равным `0160000` (должен быть кратен `04000`).
Можно воспользоваться системной утилитой `SIPP` (предварительно найдя в файле `RTSJ.MAP` адрес на который указывает символ `..28KW`, подробнее в [RT-11 Installation Guide](http://bitsavers.informatik.uni-stuttgart.de/pdf/dec/pdp11/rt11/v5.6_Aug91/AA-H376F-TC_RT-11_Installation_Guide_Aug91.pdf) п. 2.6.20). После этого необходимо будет вновь прописать загрузчик коммандой `COPY/BOOT:HX HX0:RT11SJ MZ0:`
Либо с помощью сторонней утилиты [`SYSTOP`](https://zx-pk.ru/threads/10718-soft-dlya-dvk-pdp11.html?p=932386&viewfull=1#post932386). 

### Тест скорости записи в память:
```
xxxx00     012700  MOV    #100, R0 ;
xxxx02     000100                  ;
xxxx04     005001  CLR    R1       ; <---+
xxxx06     012702  MOV    #1100,R2 ;     |
xxxx10     001100/170000           ;     |
xxxx12  0(1)10112  MOV(B) R1, (R2) ; <-+ |
xxxx14  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx16  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx20  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx22  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx24  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx26  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx30  0(1)10112  MOV(B) R1, (R2) ;   | |
xxxx32     077111  SOB    R1, 11   ;---+ |
xxxx34     077015  SOB    R0, 15   ;-----+
xxxx36     000000  HALT            ;
```
Время выполнения теста:

length |  8 Mhz  |   10MHz   |8 MHz SRAM| 10MHz SRAM
-------|---------|-----------|---------|-----------
word   | 152 sec | 120 sec   | 97 sec  | 77 sec
byte   | 193 sec | 162 sec   | 121 sec | 97 sec
