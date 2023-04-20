# Wymagania

> Sprawozdanie musi zawierać opis problemu oraz algorytmu w postaci słownej oraz w postaci pseudokodu (najlepiej jako maszyna stanów). Algorytm powinien pokazywać możliwe stany procesów, wiadomości wysyłane w każdym stanie, reakcje na odbiór wiadomości danych typów w zależności od stanu procesów, oraz warunki przejścia między stanami. Sprawozdanie musi również zawierać opis złożoności czasowej oraz komunikacyjnej.

> Opis podany w sprawozdaniu powinien być kompletny, szczegółowy i nie powinien wymagać dodatkowych wyjaśnień podczas obrony projektu. Należy przyjąć, że dowolna osoba powinna być w stanie zaimplementować algorytm na podstawie jego opisu w sprawozdaniu. Chociaż zmniejsza to czytelność, dla wygody odnoszenia się do algorytmu podczas dyskusji proszę o numerowanie linii.

# Treść projektu
## Rozważany problem

W bajkowej krainie krasnali głównym źródłem dochodu są fuchy na imprezach ludowych. Skanseny generują zamówienia na usługi. Po zdobyciu zlecenia krasnale dobierają jeden z portali, lecą na fuchę, po czym wracają.

Danych jest S skansenów generujących zlecenia oraz K krasnali. Krasnale *decydują między sobą*, kto weźmie zlecenie (skanseny nie mają nic do powiedzenia). Później zajmują jeden z P portali, udają się na fuchę, po pewnym czasie wracają.

```
           składają
Skanseny ------------> Zlecenia
                          |
                          | są wykonywane
          są używane      V
Portale  ----------->  Krasnale
         <-----------
           zwracają
```

## Opis szczegółowy
### Rodzaje Zasobów
#### Zlecenie
Produkowane przez Skansen, konsumowane przez Krasnale

#### Portal
Zajmowane i zwalniane przez krasnale wykonujące zlecenia

### Rodzaje jednostek przetwarzania
#### Skansen
Każdy skansen generuje zlecenia co pewien losowy czas. Po wygenerowaniu nowego zlecenia wysyła komunikat o nowym zleceniu do wszystkich krasnali, skansen nie ma wpływu na to, który krasnal wykona zlecenie.

#### Krasnal
Krasnale otrzymują od skansenów komunikaty o wszystkich nowych zleceniach. Krasnale w stanie `Ready` uzgadniają między sobą podział dostępnych zleceń **TODO: potrzebne szczegóły**. Krasnal posiadający zgodę na wykonanie zlecenia przechodzi do stanu `WaitPortal`. Po otrzymaniu dostępnego portalu **TODO: potrzebne szczegóły**, przechodzi do stanu wykonania fuchy `Working`, po wykonaniu wraca do stanu `Ready`

**Zachowanie domyślne:**

Wspólne dla wszystkich stanów jeśli nie określono inaczej
```py
receive_message(skansen, Message("NEW_JOB")):
    jobs += job

receive_message(krasnal, Message("REQ_JOB")):
    send_message(krasnal, Message("ACK_JOB"))

receive_message(krasnal, Message("PORTAL_FREE")):
    portals++

# TODO
```

`#TODO:`
> Nie wiem jak kontrolować liczbę portali przy ich zajmowaniu (podobny problem ze zleceniami), może: uzgodniony proces (w sekcji krytycznej wysyła informację że zajął zasób `TAKEN_PORTAL`) wtedy, na początku `WaitPortal` (i analogicznie `AgreeJob`), gdy od ostatniego komunikatu `ACK_PORTAL`(`ACK_JOB`) nie pojawił się jeszcze `TAKEN_PORTAL`(`TAKEN_JOB`), to czekamy aż się taki pojawi.


**Ready (stan początkowy):**
- Stan bezczynności, i oczekiwania na nowe zlecenia od skansenów
- Po otrzymaniu wiadomości o nowym zleceniu krasnal przechodzi do stanu `AgreeJob`

```py
receive_message(skansen, Message("NEW_JOB")):
    job = message.job
    set_state("AgreeJob")
```

AgreeJob:
- Krasnale uzgadniają który z nich ma wykonać zlecenie. Każdy z krasnali w tym stanie, rozsyła wiadomość `REQ_JOB` do wszystkich pozostałych swoją propozycję przyjęcia zlecenia. Składa się ona z identyfikatora krasnala.
- Gdy krasnal otrzymuje zapytanie `REQ_JOB` od innego krasnala, a jego zegar wektorowy jest młodszy od pytającego, odsyła wiadomość `ACK_JOB` i przechodzi do stanu `Ready`.
- Gdy krasnal otrzyma potwierdzenia `ACK_JOB` od wszystkich krasnali, przechodzi do stanu `WaitPortal`
- w przypadku remisu, wybierany jest krasnal o niższym identyfikatorze.

```py
# TODO
```

WaitPortal:
- Przy starcie systemu, wszystkie krasnale mają informację o liczbie dostępnych portali.
- Gdy dostępny jest co najmniej 1, krasnal w stanie WaitPortal, wysyła do wszystkich krasnali wiadomość `REQ_PORTAL`
- **TODO: Jak tu teraz nie zgubić informacji o ilości dostępnych portali?**

```py
acknowledged = 0
responded = 0
while (free_portals == 0) wait
send_message(KRASNALE, Message("REQ_PORTAL"))
while (acknowledged < KRASNALE.count) wait
set_state("Working")
```

```py
receive_message(krasnal, Message("REQ_PORTAL")):
    if (krasnal.clock, krasnal.id) < (my.clock, my.id):
        send_message(KRASNALE, Message("ACK_PORTAL"))
        while (responded < KRASNALE.count) wait
        set_state("WaitPortal") # reset state
    else:
        send_message(krasnal, Message("REQ_PORTAL"))

receive_message(krasnal, Message("ACK_PORTAL")):
    acknowledged++
    responded++
```


Working:
- Krasnal pracuje, po zakończeniu przechodzi do stanu `Ready`

```py
job.do_work()
send_message(job.skansen, Message("JOB_DONE"))
send_message(KRASNALE, Message("PORTAL_FREE"))
set_state("Ready")
```