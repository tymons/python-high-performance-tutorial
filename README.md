# Wysokowydajny Python - Przetwarzanie Rozproszone

W codziennej pracy z pythonem zdarzają się sytuacje w której napisany program działa zbyt długo. W takim przypadku kod najprawdopodobniej został napisany nieefektywnie lub wykonywane są długotrwałe operacje I/O, które należy zoptymalizować. Użytkownik pozostawiony bez informacji zwrotnej może błędnie założyć, że program wpadł w nieskończoną pętlę lub jego żądnie (np. HTTP) spowodowało zbyt duże obciążenie serwera i należy przerwać operację. W obu przypadkach sytuacja ta jest niekorzystna i powinna zostać naprawiona.  Do rozwiązania tych problemów będziemy wykrozystywać techniki opisane w niniejszej instrukcji.

Wymagania do zadań:
- `python >= 3.6`
- `multiprocessing` package (`pip install multiprocessing`)
- `threads` package (`pip install threads`)
- `asyncio` package (`pip install asyncio`)


## Pomiary czsu w pythonie 

Do identyfikacji długo trwających operacji konieczny jest pomiar czasu. Pomiary powinny być realizowane kilkakrotnie a następnie wyniki uśredniane aby zminimalizować wpływ różnego rodzaju optymalizacji pythona. Istnieją dwa sposoby na pomiar czasu trwania wykonywanych operacji. 

### Pomiary jednostkowe

Do sprawdzenia czasu trwania fragmentu kodu wykorzystywana jest biblioteka `time`. Przed wykonaniem fragmentu kodu aktualny czas zegara
jest zapisywany do zmiennej. Po wykonaniu opracji, których wydajność chcemy zmierzyć, ponownie sprawdzamy czas zegarowy. Te dwie wartości są od siebie odejmowane i otrzymujemy czas trwania operacji. 

Biblioteka `time` udostępnia kilka sposobów na ekstrakcję danych "zegarowych". Pierwszym i najbardziej popularnym sposobem otrzymania czasu UNIXowego (sekundy od 00:00:00 UTC 01/01/1970) jest funkcja modułu `time.time()`

```python
import time

start = time.time()
...
print(f'Time: {time.time() - start}')
```

Nowoczesne systemy zwykle zapewniają rozdzielczość mili- lub mikrosekundową. Czas ten jest utrzymywany przez dedykowany sprzęt w komputerach, obwód RTC (zegar czasu rzeczywistego). Ten „czas rzeczywisty” podlega również modyfikacjom na podstawie Twojej lokalizacji (strefy czasowe) i pory roku (czas letni) lub jest wyrażony jako przesunięcie względem czasu UTC (znanego również jako czas GMT lub Zulu).

Z uwagi na ograniczenia zegara RTC do pomiarów wydajnościowych (głównie słaba rozdzielczość) sugerowane jest użycie innego źrodła zegarowego. Metoda `time.perf_counter()` dostarcza relatywnego czasu wyznaczonego na podstawie wewnętrznego zegara CPU. Wartości są mierzone za pomocą licznika procesora i, jak określono w dokumentacji, ten sposób powinien być używany do pomiaru odstępów czasu. W tym przypadku nie uzyskujemy jednak interpretowalnej daty i godziny jak w przypadku `time.time()`.

> Zadanie (2 pkt):  
>   
> Napisz skrypt zapisujący i odczytujcy z dysku plik test.txt z Twoim imieniem i numerem indeksu. Porównaj czasy zapisu i odczytu z użyciem pomiarów bazujących na zegarze CPU i rozdzielczości nanosekundowej.  

### Pomiary powtarzalne

Opisywana powyżej metoda nie jest precyzyjna, ponieważ mogą zdarzyć się sytuacje w której egzekucja danego fragmentu kodu zostanie chwilowo zawieszona przez system operacyjny lub sam intepreter pythona. W takim wypadku pomiary mogą być zakłamane i słabo odpowiadać rzeczywistości. Odpowiedzią na ten problem jest biblioteka wbudowana w pythona o nazwie `timeit`. Metody z tego modułu uruchamiają fragment kodu miliony razy (wartość domyślna to 1000000), aby uzyskać statystycznie najbardziej odpowiedni pomiar czasu wykonania.

Prototyp funkcji jest postaci: `timeit.timeit(stmt, setup, timer, number)`, gdzie:

- `stmt` to string zawierający kod, który ma zostać zmierzony.
- `setup` to string zawierający kod uruchamiany przed uruchomieniem `stmt`. Zwykle używa się tego do importowania wymaganych modułów.
- `timer` obiekt `timeit.Timer`
- `number` liczba egzekucji `stmt`

Przykład liniowego przeszukiwania:

```python
def linear_time():
    SETUP_CODE = '''from __main__ import linear_search
                    from random import randint'''
     
    TEST_CODE = '''mylist = [x for x in range(10000)]
                   find = randint(0, len(mylist))
                   linear_search(mylist, find)'''
                   
    # timeit.repeat statement
    times = timeit.timeit(setup = SETUP_CODE,
                          stmt = TEST_CODE,
                          number = 10000)
```

> Zadanie (2 pkt):  
>   
> Rozszerz napisany wyżej skrypt do pomiaru czasu zapisu i odczytu Twojego imienia/indeksu z pliku o pomiar wydajności funkcji `is_name_index(name: str, index: int)` sprawdzający czy cyfra odpowiadająca liczbie liter z imienia % 10 jest zawarta w indeksie. Przykład: `Tymoteusz, 144298` zawiera 9 liter, cyfra 9 znajduje się w `144298` więc zwracana jest wartość `True`.

2. Multiprocessing 
3. Multithreading
4. Programowanie Asynchroniczne (Python Asynchronous Programming)
