# Wysokowydajny Python - Przetwarzanie Rozproszone

W codziennej pracy z pythonem zdarzają się sytuacje w której napisany program działa zbyt długo. W takim przypadku kod najprawdopodobniej został napisany nieefektywnie lub wykonywane są długotrwałe operacje I/O, które należy zoptymalizować. Użytkownik pozostawiony bez informacji zwrotnej może błędnie założyć, że program wpadł w nieskończoną pętlę lub jego żądnie (np. HTTP) spowodowało zbyt duże obciążenie serwera i należy przerwać operację. W obu przypadkach sytuacja ta jest niekorzystna i powinna zostać naprawiona.  Do rozwiązania tych problemów będziemy wykrozystywać techniki opisane w niniejszej instrukcji.

Wymagania do zadań:
- `python >= 3.6`
- `multiprocessing` package
- `threads` package
- `asyncio` package


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
def linear_search(mylist, find):
    for x in mylist:
        if x == find:
            return True
    return False
    
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

## Multiprocessing 

Identyfikacja fragmentów kodu z użyciem metod pomiaru czasu jest dopiero pierwszym krokiem w kierunku przyspieszenia działania całego programu. Znalezienie fragmentu kodu powodującego opóźnienia wskazuje tylko problem, który należy w kolejnym etapie rozwiązać. W naiwnym podejściu operacje trwające zbyt długo możemy rozdysponować pomiędzy wszystkie dostępne logiczne procesory. Biblioteka `multiprocessing` dostarcza funkcjonalności uruchomienia wielu procesów w obrębie jednego programu. Informacja o liczbie procesorów logicznych może zostać uzyskana w następujący sposób:

```python
import multiprocessing as mp
print("Number of processors: ", mp.cpu_count())
```

Aby rozproszyć opracje na dostępne procesory logiczne wykorzystuje się klasę `Pool` (lub `Process`) i metody `apply()` lub `map()`. Zarówno `apply()`, jak i `map()` przyjmują funkcję, która ma być zrównoleglona, jako główny argument. Różnica polega na tym, że `apply()` przyjmuje argument `args`, który akceptuje parametry przekazane do "funkcji do zrównoleglenia" jako argument, podczas gdy funkcja map może przyjąć jako argument obiekt typu `iterable` (np. listę).

Poniższy przykład ilustruje problem zliczenia elementów znajdujących się w zadanym zakresie. Metoda `howmany_within_range(row, minimum, maximum)` przyjmuje parametr `row` będący listą elementów np. `[1, 5, 7, 3, 7, 8, 10, 22, 0]`, parametr `minimum` i `maximum` definiuje górny i dolny zakres do zliczania.

```python
def howmany_within_range(row, minimum, maximum):
    """Returns how many numbers lie within `maximum` and `minimum` in a given `row`"""
    count = 0
    for n in row:
        if minimum <= n <= maximum:
            count = count + 1
    return count
```

Uruchomienie powyższej metody dla tablicy 2-D `data` na wielu procesach realizowane jest z użyciem:

```python
pool = mp.Pool(mp.cpu_count())

results = [pool.apply(howmany_within_range, args=(row, 4, 8)) for row in data]

pool.close()    
```

Podejście to ma jednak wady. Po pierwsze procesy uruchamiane są synchroniczne co oznacza, że drugi wiersz z `data` zostanie sprawdzony dopiero gdy poprzedni proces uruchomiony z użyciem `pool.apply(...)` zakończy swoje działanie na pierwszym wierszu. Aby naprawić ten problem i w pełni wykorzystać możliwości biblioteki `multiprocessing` należy użyć metod asynchronicznych, tj. metod, które nie blokują dalszego działania programu. Program można zmodyfikować do postaci:

```python
def collect_result(result):
    global results
    results.append(result)


pool.apply_async(howmany_within_range, args=(row, 4, 8), callback=collect_result)

print(f'This line executes immediately')
do_some_stuff()

pool.close()
pool.join() # Here we collect all applied tasks 

```

W ten sposób zadania do procesów są niejako tylko "dodawane" do pool'a, gdzie następnie są dystrybuowane do worker processów. Ich wykonanie następuje w tle (w kontekście głównego programu). Wyniki poszczególnych procesow możemy zbierać z użyciem argumentu `callback` będącego funkcją lub pobierając wyniki z użyciem `.get()` na obiekcie `ApplyResult`.

```python
result_objects = [pool.apply_async(howmany_within_range, args=(row, 4, 8)) for row in data]

results = [r.get()[1] for r in result_objects]
```

W przypadku użycia `map()` zamiast funkcji `apply()` z powyższego przykładu argument `data` jest automatycznie dzielony na tzw. `chunki`, które wysyłane są procesów roboczych w `pool`. Podział elementu iteracyjnego na porcje działa lepiej niż przekazywanie każdego elementu po jednym elemencie na raz — szczególnie jeśli element iteracyjny jest duży. Dodatkowo, kod jest czytelniejszy i nie ma wymagania składania listy wyników. Przekształcenie obiektu `iterable` w listę w celu podzielenia go na `chunki` może mieć jednak bardzo wysoki koszt pamięciowy, ponieważ cała lista będzie musiała być przechowywana w pamięci.


> Zadanie (2 pkt):  
>   
> Zmodyfikuj zliczanie elementów w zakresie `minimum` i `maximum` uzywając metody `map_async()` z biblioteki `multiprocessing`.


## Multithreading

Wykorzystanie wieloprocesowości w pythonie jest metodą z największym potencjałem do przyspieszania obliczeń poprzez ich zrównoleglanie. Ma jednak ona swoje wady, np. konieczność tworzenia kopii przestrzeni danych dla każdego procesu czy też zapewnienie sposobów komunikacji pomiędzy procesami. Drugim i przystępniejszym sposobem na przyspieszenie obliczeń jest wykorzystanie wielowątkowści. Biblioteka `threading` zawiera metody i klasy niezbędne do implementacji wielowątkowości.

Wątek budowany jest z użyciem klasy bazowej `threading.Thread` lub bezpośrednio wskazując funkcję do wykonania:

```python
def print_cube(num):
    print(f"Cube: {num * num * num}")
    
t1 = threading.Thread(target=print_square, args=(10,))
t1.start()
t1.join()
```

```python

class SomeWorker(Thread):

    def __init__(self, num):
        super(SomeWorker, self).__init__()
        self.num = num
        
    def run(self):
        return num * num * num
```

Komunikacja międzywątkowa odbywa się najczęściej z wykorzystaniem tzw. `thread-safe` kolejek: `Queue`. Poniżej został zaprezentowany przykład użycia kolejki do komunikacji pomiędzy wątkiem głównym a jednym z 10 utworzonych wcześniej wątków.

```python
import threading
from queue import Queue

q = Queue()

def job(worker):
    with open('messages.txt') as f:
        for line in f:
            print(line)


def reader():
    while True:
        worker = q.get()
        job(worker)
        q.task_done()
        
def main():
    for x in range(10):
        t = threading.Thread(target=reader)
        t.daemon = True
        t.start()

    for worker in range(1):
        q.put(worker)

    q.join()
```

Property `.deamon = True` pozwoli głównemu programowi zakończyć pracę nawet jeżeli jego wątki ciągle działają. Aplikacje zwykle czekają, aż wszystkie wątki podrzędne zostaną zakończone, zanim same zakończą swoje działanie.

> Zadanie (2 pkt):  
>   
> Zmodyfikuj klasę `SomeWorker` w taki sposób aby wątki cyklicznie wypisywały swoje ID (ustawiane w konstruktorze) i aby możlwie było zakończenie ich działania z poziomu wątku głównego (wykorzystując Queue).

## Programowanie Asynchroniczne (Coroutines and Tasks)

Programowanie asynchroniczne jest ostatnim elementem omawianych współbieżności. Biblioteka `asyncio` jest to jednowątkowa wielozadaniowość oparta na pojedynczym procesie dzięki temu rzadziej występują problemy z synchronizacją czy wyścigiem.

W przeciwieństwie do zwykłej funkcji wątku z pojedynczym punktem wyjścia, `coroutine` (korutyna) może wstrzymać i wznowić swoje wykonanie (wykorzystując `await`). Tworzenie korutyny polega na użyciu słowa kluczowego `async` przed deklaracją funkcji. Poniżej zaprezentowana przykład aplikacji asynchronicznej wykorzystującej bibliotekę `asyncio`

```python
import asyncio

num_word_mapping = {1: 'ONE', 2: 'TWO', 3: "THREE", 4: "FOUR", 5: "FIVE", 6: "SIX", 7: "SEVEN", 8: "EIGHT",
                   9: "NINE", 10: "TEN"}

async def delay_message(delay, message):
    logging.info(f"{message} received")
    await asyncio.sleep(delay) # time.sleep is blocking call. Hence, it cannot be awaited and we have to use asyncio.sleep
    logging.info(f"Printing {message}")
    
async def main():
    logging.info("Main started")
    logging.info(f'Current registered tasks: {len(asyncio.all_tasks())}')
    logging.info("Creating tasks")
    task_1 = asyncio.create_task(delay_message(2, num_word_mapping[2])) 
    task_2 = asyncio.create_task(delay_message(3, num_word_mapping[3]))
    logging.info(f'Current registered tasks: {len(asyncio.all_tasks())}')
    await task_1 # suspends execution of coroutine and gives control back to event loop while awaiting task completion.
    await task_2
    logging.info("Main Ended")

if __name__ == '__main__':

    asyncio.run(main()) # creats an envent loop
```

```
07:35:32:MainThread:Main started
07:35:32:MainThread:Current registered tasks: 1
07:35:32:MainThread:Creating tasks
07:35:32:MainThread:Current registered tasks: 3
07:35:32:MainThread:TWO received
07:35:32:MainThread:THREE received
07:35:34:MainThread:Printing TWO
07:35:35:MainThread:Printing THREE
07:35:35:MainThread:Main Ended
```

Uzyskano tą samą funkcjonalność co w przypadku użycia biblioteki `threading`. W tym przypadku została zbudowana jednowątkowa aplikacja wykorzystująca tzw. Event Loop i korutyny dające możliwości uruchomienia współbieżnego kodu. Do komunikacji pomiędzy korutynami wykorzystuje się obiekt `asyncio.Queue`, natomiast do synchronizacji `asyncio.Lock` czy `asyncio.Event`.

> Zadanie (2 pkt):  
>   
> Zmodyfikuj powyższy program w którym czas opóźnienia jest losowany z przedziały 0-4 sekund. Uruchom 100 korutyn i udowodnij poprawne działanie programu. (Tip: do "zebrania" 100 korutyn wykorzystaj asyncio.gather())
