Źródło: https://www.youtube.com/watch?v=XrlrbfGZo2k  
* Redford
* q3k
* MrTick/Pan Kleszcz

### Wprowadzenie

Jest rok 2016. Koleje Dolnośląskie kupują 11 pociągów imupls (45WE) od Newagu.  
Pociągi osiągają milion kilometrów przebiegu - po takiej ilości kilometrów należy zrobić "duży przegląd" pociągów - serwis. Gwarancja producenta wygasła więc Koleje Dolnośląskie tworzą przetarg.  
W 2021 roku przetarg wygrywa polska firma SPS Mieczkowski - niezależny (niepowiązany z producentem) serwisant pociągów.  

Na początku roku 2022 pierwszy Impuls 45WE trafia do serwisu SPS.

![Impuls KD](https://upload.wikimedia.org/wikipedia/commons/f/f3/Impuls_45WE-024.jpg)

##### Odkrycia serwisu SPS

W kwietniu 2022 pociągowi 45WE-024 został wykonany serwis, ale... Nie można było go uruchomić.  
Podobnie miała się sytuacja z innymi pociągami. W czerwcu 2022 roku inne Impulsy napotykają podobne problemy (z Polregio), serwisowane w zupełnie innym warsztacie.  
Media interesują się sprawą - brak pociągów już jest zauważalny, flota jest coraz mniejsza, a w lipcu 2022 roku Newag obwinia SPS - serwis miał aktywować systemy bezpieczeństwa pociągu, co uniemożliwiało jego start.  

Warsztat zauważa, że ta usterka jest po prostu dziwna. Komputery sprawdzające informują, że pociąg jest w pełni sprawny i gotowy do uruchomienia, ale z dzwinych, nieznanych im powodów się nie uruchamia - tak jakby ktoś po prostu go "odpiął od prądu".
Mieli też podejrzenia, które kierowały całą sprawę w stronę prodcuenta i możliwych jakiś nieczystych zagrywek z ich strony.  

Aby rozwikłać zagadkę, Koleje Dolnośląskie i SPS zatrudniły trzech specjalistów z grupy Dragon Sector (Redford, q3k i MrTick).

### Jak odblokować pociąg

Zablokowany pociąg w wypadku impulsów:  
1. HMI raportuje "gotowy do ruchu" - w takim razie pociąg wydaje się w pełni sprawny;  
2. Zwiększamy prędkość manetką;  
3. Zwolnienie hamulców;  
4. Pociąg stoi;  

Badacze otrzymali do dyspozycji jeden zablokowany pociąg, dwa kontrolery sterujące (PLC) z dostępem do oprogramowania deweloperskiego oraz dokumentację serwisową pojazdu.

![Pociąg](https://github.com/user-attachments/assets/445bbbfb-f7e5-41b6-b242-9d3b0684335d)


Magistrala CANopen traffic - różnica w trafficu dla działającego pociągu i uziemionego:

|Pociąg|CANopen traffic|
|------|---------------|
|45WE-022, działa|ID: ``0x0212`` <br> Playload: ``01 1f 0b 0b 0b 0b 52 00`` <br> Sent by CPU on CAN1|
|54WE-023, uziemiony|ID: ``0x0212`` <br> Playload: ``01 0f 00 00 00 00 52 00`` <br> Sent by CPU on CAN1|

Widzimy różnicę w przesyłanych bitach. Hakerzy uznali że różnica leży gdzieś pomiędzy PLC a Inverterem.  

<img width="940" alt="Screenshot 2025-06-08 at 13 31 15" src="https://github.com/user-attachments/assets/fe29897f-4568-4ee4-bbb4-815cd36e96c9" />  

*Źródło: https://www.youtube.com/watch?v=XrlrbfGZo2k timestamp: 9:16*  

Trudne okazało się dla hakerów pobranie kodu z modułów PLC. Jak możemy się domyślić (zazwyczaj) na sprzęcie nie ma magicznego przycisku "pobierz kod źródłowy".
Musieli korzystać z technik inżynierii wstecznej. Skorzystali z wbudowanego debuggera w sterownikach PLC (typ CAP1131) pracującego na protokole SysCom. 
Po zalogowaniu do sterownika (domyślnie user_0, hasło sele0) mogli ustawiać zakresy adresów pamięci do podglądu i okresowo odpytywać ich zawartość.  

Po jakimś czasie SPS dał badaczom deadline - tydzień *(ten deadline z kolei KD dało SPS, by składy wróciły na tory - inaczej zwróciliby się do producenta)*. Zespół musiał znaleźć
rozwiązanie i to szybko.  
 
Kluczowe było zidentyfikowanie ukrytego kodu blokującego działanie pojazdu. Dragon Sector ustalił, że w większości przebadanych pojazdów (24 z 29 analizowanych) 
znajdował się „złożony system blokad”. Mechanizm ten sprawdzał kilka warunków, a gdy któryś z nich został spełniony, kontroler wyłączał napęd tak, że pociąg cofał hamulce, 
ale falowniki (siła napędowa) pozostawały wyłączone.  

Z początku te warunki wyglądały niegroźnie - sprawdź jeśli oraz "jakaś duża liczba" - pewnie chodziło o milion kilometrów! Jednak kwestia powtarzania się owych warunków 
wzbudziła w badaczach wątpliwości. I słusznie! Okazało się bowiem, że gdy warunki te to właśnie systemy zaporowe, które zindentyfikowano:
* Na początku gdy pociąg nie ruszał się przez 10 dni (tzn. gdy pociąg nie osiągnął prędkości conajmniej 60km/h na 3 minuty). Po skontaktowaniu z Newagiem, firma naprawiła pociągi - zmienili brak ruchu z 10 dni na 21 dni oraz... dodali sprawdzenie geolokalizacyjne, by wykryć konkurencyjne serwisy! <img width="698" alt="Screenshot 2025-06-08 at 13 54 54" src="https://github.com/user-attachments/assets/a480c2ef-b2e3-471b-9cbe-3fc6c0b9e9ff" />  
Można zauważyć specjalną flagę ``&& (var3 & 1)``, która służyła firmie do... Testowania czy blokada geolokalizacyjna działa, sprawdzając ją na terenie swojego warsztatu!

* Wymiana konkretnego podzespołu (sygnał CAN831 systemu pneumatyki) bez autoryzowanej części zamiennych;

Dodatkowo odkryto wariant oprogramowania zgłaszający fałszywą awarię sprężarki pomocniczej przy zbliżającym się terminie przeglądu P3 lub po osiągnięciu miliona kilometrów przebiegu. 
W co najmniej jednym egzemplarzu blokada mogła zostać wywołana także poprzez urządzenie podłączone sieciowo (UDP) do magistrali CAN pociągu. 
Wszystkie te funkcje blokujące nie były nigdzie udokumentowane w oficjalnej instrukcji.  

Co zabawne jeden z warunków firmie chyba nie wyszedł...

<img width="775" alt="Screenshot 2025-06-08 at 13 57 13" src="https://github.com/user-attachments/assets/fefd2f7c-5504-41fd-aefb-f4372bcadfd7" />
<img width="329" alt="Screenshot 2025-06-08 at 13 57 46" src="https://github.com/user-attachments/assets/9105ddd6-19df-4f9b-81df-2e2cb5f23406" />

*źródło: https://www.youtube.com/watch?v=XrlrbfGZo2k timestamp: 32:00 - 34:00*





Analiza kodu odsłoniła również ukrytą procedurę odblokowania: w niezmienionym fabrycznym oprogramowaniu znajdowała się tajna kombinacja przycisków w kabinie maszynisty, która resetowała blokady. Po jej naciśnięciu nawet „uziemiony” Impuls odzyskiwał napęd. Jak podkreślają źródła prasowe, Newag mógł ponownie uruchomić pociąg „praktycznie jednym kliknięciem”. Jednak w nowszych wersjach oprogramowania mechanizm ten usunięto, pozostawiając jedynie warunki blokujące. Ponieważ nie było innego legalnego sposobu pobrania kodu sterownika, 
hakerzy wykorzystali właśnie interfejs SysCom do debugowania: jest to prosty, bazujący na protokole UDP, zestaw komend służących do odczytu pamięci sterownika. Zalogowali się jako „user_0” (hasło „sele0”), następnie zakresowo monitorowali pamięć i analizowali przebieg programu. Dzięki temu etapowi udało im się odtworzyć działanie funkcji blokad i potwierdzić warunki logiczne ukryte w kodzie.  

Badacze z Dragon Sector zwrócili uwagę, że całość systemu sterowania blokadami przypomina technikę geofencingu. W prezentacji na konferencji 37C3 opisali, że oprogramowanie Newagu sprawdzało położenie GPS i czas postoju, by egzekwować wirtualne granice: Impulsy miały „zapadać w hibernację”, gdy tylko opuściły dozwolone obszary serwisowe lub długo stały poza autoryzowanymi miejscami. Jedyną oficjalną metodą „odratowania” tak zablokowanego składu była interwencja autoryzowanego serwisu Newagu – w praktyce najem technika producenta, który „z deka” ponownie aktywował układ. Hakerzy określili to jako rodzaj okupu wymuszanego na przewoźnikach, by ograniczyć możliwość napraw w niezależnych warsztatach



Cała sprawa unikalnie łączy inżynierię sterowników przemysłowych ze zjawiskiem DRM (Digital Rights Management). Stosowane przez Newag „sim-locki” na pociągach mają cel handlowy – monopolu na serwis – i były celowo zaimplementowane w kodzie sterowników. Jednocześnie badacze stanowczo podkreślają, że nie zmieniali oryginalnego oprogramowania: wszystkie analizowane składy jeżdżą na niezmodyfikowanym kodzie producenta.








