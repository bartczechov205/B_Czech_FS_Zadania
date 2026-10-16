1. Opis projektu:

Aby wykonać zadanie użyłem programu STM32CubeMX, który poprzez interafce graficzny pozwala
szybciej skonfigurować m.in. MCU, timery czy też peryferia. Generuje on część kodu
odpowiedzialną za konfiguracje sprzętową i inicjalizacje. 

Zapisany kod służy do obsługi STM32G473CET3, który pracuje z zewnętrznym źródłem taktowania
o częstotliwości 16Mhz. Z racji na to, że w specyfikacji nie było określone czy wspominane
taktowanie pochodzi z rezonatora kwarcowego czy tez z generatora, przyjąłem jako założenie
zewnętrznego rezonatora kwarcowego podłączonego do HSE, w rzeczywistym projekcie należałoby to ustalić. 

Program realizuje miganie
diodą LED na pinie PA5 poprzez podłączanie na ten pin timera 1 kanału timera 2 (TIM2_CH1) który generuje syganł
PWM o częstotliwości 5hz i wypełnieniu 50%. Pełen cykl trwa 200ms, przez 100ms dioda się świeci, 
a przez pozostałe 100ms pozostaje zgaszona.

Dodałem także watchdog IWDG taktowany z osobnego oscylatora (LSI) o częstotliwości 32Khz, jego
reset przy obecnych ustawieniach nastąpi po 512 ms.

W przypadku błędów w kluczowych sytuacjach takich jak m.in startu interfaceu CAN wywoływana 
jest funkcja Error_Handler() która wprowadza program w pustą pętle i doprowadza do resetu IWDG.	
 
Zaimplementowałem transmisję danych przez interfejs FDCAN1 jako klasyczne CAN z szybkością 1 000 000 bit/s.
Piny PA11 i PA12 stanowią odpowiednio piny do odbioru i transmisji dla transciver'a. Do podłączania
MCU do rzeczywistej magistrali CAN potrzebny był jeszcze transciver. 

UpTime jest liczbą 2 bajtową, jednak z racji na ograniczenia rozmiaru danych podzielony został na
dwie 1 bajtowe liczby wysyłane przez interface CAN 

2. Konfiguracja w STM32CubeMX

2.1 Taktowanie

Jak wspomniałem w opisie, w specyfikacji zadania nie ma sprecyzowane czy taktowanie 
pochodzi z rezonatora kwarcowego czy też z generatora, jest to ważne w kwestii tego
jakie piny powinny być skofigurowane pod odbiór taktowania. Przyjąłem że jest to
rezonator kwarcowy co skonfigurowałem przy użyciu RCC.
Ustawiłem HSE oraz HCKL na 16 Mhz. HCLK można by było 
zwiększyć, ale skoro kod jest dość prosty lepiej pozostawić taktowanie równe
temu z rezonatora kwarcowego, aby nie zwiększać niepotrzebnie zużycia energii. 

Oba APB pozostały równe 1 dlatego PCLK też został 1 więc bazowo na peryferiach
również mam 16 Mhz.

2.2 Timer i PWM

Żeby obsłużyć mruganie diodą użyłem Timera który generuje sygnał PWM.
Wybrałem Timer2 oraz generowanie PWM na kanale 1, ponieważ pin PA5 to obsługuje.
Prescaler ustwaiłem na 1599 przez co dostałem 10000 taktów na sekunde,
z tego wynika, że jeden takt przypada co 0,1 ms. Counter Period dałem jako
1999 czyli okres równy 200 ms. Nastepnie w ustwaniach PWM wybrameł Mode 1,
CH Polarity High oraz Pulse 1000 czyli 100 ms, dzieki temu przez 100ms dioda się świeci, 
a przez pozostałe 100ms pozostaje zgaszona.

2.3 CAN

W ustwieniach CAN wybrałem Clasicc CAN. Nastepnie wybrałem FIFO mode
tak aby interface obsługiwał dane hronologicznie. Ustwaiłem Auto Retransmioion na 
enable aby w razie niepowdzenia transmisji kontroler autmoatycznie ponawiał 
próbę wysłania ramki. 

Aby wyznaczyć odpowedni bitrate posłużyłem się wzorem:

Bitrate = zegar FDCAN / (Prescaler * (1 + TimeSeg1 + TimeSeg2))

zegar FDCAN wynosił 16M

Dla lepszego dzielnia Prescaler pozostał 1

TimeSeg1 wyzancza kwanty czasu opóźniajce czas próbkowania 
wzgledem czasu synchornizacji

TimeSeg2 z kolei to kwanty czasu opóźniajce nastepną
synchronizcaje 

Ich odpowiednie ustwanie jest zleżne od sytułacji, więc 
przyoisałem im zbliżone wartości: TimeSeg1 7, a
TimeSeg1 8, co zapewniło Bitrate równy 1000000 bit/s.
Dla CAN i innych popularych interfaców jak SPI, I2C czy
UART bitrate i boundrate są równe.  

2.4 IWDG

W programie zastosowałem niezależny watchdog IWDG, taktowany wewnętrznym oscylatorem 
LSI o nominalnej częstotliwości 32 kHz. Prescaler ustwiony jest na 4 wiec IWDG
wykonuje 8000 taktów na sekundę. Down Counter ustwiony jest na 4095, zatem
reset nastąpi po 4096/8000 * 100 = 512 ms.

3. Działanie Programu

Część kodu odpowiadająca za konfigurację sprzętową i inicjalizację 
peryferiów została wygenerowana na podstawie ustawień w STM32CubeMX. 
Na początku programu wykonywana jest inicjalizacja biblioteki HAL, 
konfiguracja zegara procesora oraz inicjalizacja GPIO, FDCAN1, 
watchdoga IWDG i timera TIM2.

Następnie tworzę nagłówek wiadomości dla FDCAN. Ustawiam w nim m.in. identyfikator ramki, 
standardowy format identyfikatora 11-bitowego, długość danych wynoszącą 2 bajty i format Classic CAN.

Potem stworzyłem bufor txData typu uint8_t, zawierający dwa elementy, gdzie
każdy element zajmuje jeden bajt. W buforze zapisuję później dwubajtową 
liczbę upTime, rozdzieloną na starsze i młodsze bajty. Robię to, ponieważ funkjca:

(HAL_FDCAN_AddMessageToTxFifoQ( , , const uint8_t * pTxData)

w ostatnim argumencie przyjmuje jedynie dane o formacie 1 bajtowym.

Następnie uruchamiam interfejs FDCAN1 oraz generowanie PWM na kanale TIM2_CH1. 
Jeśli któraś z tych operacji zwróci błąd, program przechodzi do funkcji Error_Handler().

W następnej części kodu tworzę też dwie zmienne:

- lastTime typu uint32_t, początkowo równą 0, która służy do odmierzania kolejnych okresów jednej sekundy.

- upTime typu uint16_t, początkowo równą 0, która przechowuje liczbę zliczonych sekund.

Przed wejściem do głównej pętli odświeżam watchdog.

W głównej pętli programu najpierw odczytuję przez HAL_GetTick(), czyli liczbę milisekund 
zliczonych od uruchomienia zegra. Zapisuję ją w zmiennej currentTime.

Następnie sprawdzam warunek:

currentTime - lastTime >= 1000

Jeżeli upłynęło co najmniej 1 sekunda, zwiększam upTime o 1 i przygotowuję dane do wysłania.

Aby zapisać liczbę 2 bajtową jako dwa osobne bajty stosuje:
 
- najpierw przesuięcie o przesuniecie bitowe o 8 pozcyji, aby wydobyc starsze bity (txData[0] = upTime >> 8;)

- nastepnie, dzięki bitowemu AND z wartoscią 255 tworzę bitmaskę i wydobywam 8 najmłodszych bitów (txData[1] = upTime & 255)

Po przygotowaniu danych sprawdzam, czy w kolejce nadawczej FDCAN jest wolne miejsce. 
Jeżeli jest, wywołuję HAL_FDCAN_AddMessageToTxFifoQ(), która wysyła dane do transmisji. 
Jeżeli funkcja zwróci błąd, program przechodzi do Error_Handler(). 

Następnie zwiększam lastTime o 1000, a na końcu każdego głównej pętli odświeżam watchdog.