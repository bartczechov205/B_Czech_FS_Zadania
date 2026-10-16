SDC to obwód bezpieczeństwa, który ma za zadanie odłączyć zasilanie na AIR-ach, które z kolei odcinają 
zasilanie z akumulatora trakcyjnego w pojeździe. Składa się on z wielu przycisków i przekaźników – jeśli 
choć jeden przerwie obwód, AIR-y zostają odłączone. Przerwanie obwodu SDC może nastąpić m.in. przez wciśnięcie 
przycisku bezpieczeństwa w kokpicie, ale także przez układ BSPD, który wykryje nieprawidłowość.

W specyfikacji zadania nie ma ustalonego progu, który w naszym bolidzie byłby uznawany za mocne hamowanie 
albo który blokowałby koła, więc mam pewną swobodę w jego ustalaniu, jednak z regulaminu wyczytałem, że maksymalny 
próg mocnego hamowania to ciśnienie 30 barów. Jednocześnie ciśnienie to nie może doprowadzić do zablokowania kół. 
Założyłem więc, że progiem mocnego hamowania będzie wartość 25 barów. W regulaminie jest także mowa o tym, że pomiędzy 
czujnikiem a BSPD nie może być żadnego innego urządzenia, czyli formalnie układ, który zaprojektowałem 
(pomijając czujnik), jest fragmentem BSPD.

(W obliczeniach posługuję się wartościami z zaokrągleniem do 2. miejsca po przecinku).

Napięcie, jakie pojawi się na czujniku przy 25 barach, obliczyłem jako:

Ucz = 0,5 V + 25 * 0,02857 V = 1,21 V

Następnie musiałem dobrać rezystory do dzielnika napięcia, który z 5 V ze źródła zasilania 
dałby mi policzoną wartość 1,21 V. Jako dolny rezystor przyjąłem wartość 100 omów, a następnie posłużyłem się wzorem:

Uwys = Uwej * R2/(R1+R2)

Po podstawieniu znanych wartości:

1,21 = 5 * 100/(100+R1) => R1 = 379/1,21 = 313,22

Zatem za drugi rezystor przyjąłem wartość 313,22 oma.

Później zastosowałem komparator, którego wejście nieodwracające podłączyłem do czujnika, a wejście odwracające 
do ustalonej stałej wartości 1,21 V. W przypadku, kiedy czujnik przyjmie wartość większą niż 1,21 V, to na wyjściu 
komparatora pojawi się stan wysoki, który (takie przyjąłem założenie, bo to też nie zostało określone) 
potwierdza wystąpienie silnego hamowania.