[Resetowanie hasla roota w domkumencie](#resetowanie-hasla-roota-w-pizdu-wazne-masz-to-umiec-na)  
[Resetowanie hasla roota](https://linuxconfig.org/redhat-8-recover-root-password)  



# Resetowanie hasła roota w RHEL 10

W systemach bazujących na RHEL 9 i 10 (z systemd i zaktualizowanym procesem bootowania) procedura wygląda następująco:

1. Uruchom ponownie system. Gdy pojawi się menu GRUB, zatrzymaj odliczanie (np. strzałkami).
2. Wciśnij `e` na wybranym wpisie kernela, aby go zedytować.
3. Znajdź linię zaczynającą się od `linux` (lub `linuxefi` / `linux16`).
4. Dopisz na samym końcu tej linii parametr: `rd.break`
5. Wciśnij `Ctrl + X`, aby zbootować system z wprowadzonym parametrem. System zatrzyma się w powłoce awaryjnej (initramfs).
6. Zamontuj główny system plików (sysroot) w trybie do odczytu i zapisu:
```bash
mount -o remount,rw /sysroot

```


7. Przejdź do właściwego systemu plików używając chroot:
```bash
chroot /sysroot

```


8. Zmień hasło roota:
```bash
passwd root

```


9. Wymuś na systemie ponowne przypisanie kontekstów SELinux przy kolejnym uruchomieniu (niezbędne, w przeciwnym razie nie zalogujesz się):
```bash
touch /.autorelabel

```


10. Wyjdź z chroota i powłoki awaryjnej:
```bash
exit
exit

```



System zresetuje się (proces przeliczania etykiet SELinux może potrwać kilka minut) i uruchomi normalnie.

---

# Wyłączenie komunikatu Subscription Manager

Menedżer pakietów domyślnie korzysta z wtyczki `subscription-manager`, która przy każdej akcji weryfikuje status rejestracji systemu. Aby używać lokalnych repozytoriów bez komunikatów o braku subskrypcji, należy tę wtyczkę wyłączyć.

Otwórz plik konfiguracyjny (w RHEL 10 DNF korzysta z poniższych ścieżek):

```bash
vi /etc/dnf/plugins/subscription-manager.conf

```

Zmień parametr `enabled` z `1` na `0`:

```ini
[main]
enabled=0

```

Zapisz plik. Od teraz `dnf` będzie pobierał dane wyłącznie z podpiętych, lokalnych repozytoriów bez ostrzeżeń ze strony Red Hata.

---

# Notatki: Narzędzia systemowe i administracja

## Rozdział 2 - Essential tools

### Przekierowania i potoki (Pipes)

* `>` – Przekierowuje standardowe wyjście (STDOUT), nadpisując plik.
* `>>` – Przekierowuje STDOUT, dopisując na koniec pliku (append).
* `COMMAND > output 2>&1` – Przekierowuje zarówno STDOUT, jak i STDERR (błędy) do tego samego pliku (deskryptor błędów nr 2 jest kierowany tam, gdzie standardowe wyjście nr 1).
* Przekierowania mogą być łączone i są ewaluowane od lewej do prawej:
```bash
sort < jakis_plik > drugi_plik_na_output

```


* **Różnica między przekierowaniem a potokiem (`|`)**: Przekierowanie pobiera lub zapisuje dane bezpośrednio do pliku. Pipe bierze wyjście (STDOUT) jednej komendy i pcha je bezpośrednio na wejście (STDIN) drugiej komendy.

### Komendy wbudowane (Builtins) i weryfikacja plików

* **Internal commands** (wbudowane w powłokę): `echo`, `printf`, `read`, `cd`, `pwd`, `pushd`, `popd`, `dirs`.
* `type KOMENDA` – Pozwala sprawdzić, czy komenda jest wbudowana, czy zewnętrzna.
* `which KOMENDA` – Wyświetla ścieżkę pliku binarnego danej komendy.

### Historia (`history`)

* `history` – Pokazuje historię wykonanych poleceń.
* `Ctrl+R` – Wyszukuje po fragmencie tekstu. Ponowne wciśnięcie szuka kolejnego wystąpienia.
* `!numer` – Wykonuje natychmiast komendę pod danym numerem z historii.
* `!tekst` – Wykonuje natychmiast ostatnią komendę zaczynającą się od danego tekstu (UWAGA: brak potwierdzenia).
* `history -c` – Czyści historię poleceń w pamięci podręcznej (aktualna sesja).
* `history -w` – Zapisuje (lub nadpisuje, czyszcząc, jeśli użyto `-c`) plik `.bash_history`.

### Skrypty logowania powłoki

* `~/.bash_profile` – Wykonywany tylko raz, przy uruchamianiu powłoki logowania (login shell).
* `~/.bashrc` – Czytany za każdym razem, gdy uruchamiana jest dowolna powłoka bash. Konfiguracja w nim powinna być możliwie "lekka", aby nie opóźniać startu skryptów i sub-shelli.
* `/etc/issue` – Komunikat wyświetlany **przed** logowaniem użytkownika do systemu.
* `/etc/motd` – (Message of the Day) Komunikat wyświetlany **po** pomyślnym zalogowaniu.

### Pomoc systemowa (Man / Info)

* `apropos KEYWORD` (lub `man -k`) – Przeszukuje ogólne opisy w podręczniku, przydatne, gdy nie pamiętasz dokładnej nazwy komendy.
* `man -f KOMENDA` – Wyświetla skrócony opis komendy.
* **Kluczowe kategorie Man:**
* `1` - Programy i komendy powłoki.
* `5` - Formaty plików i konwencje konfiguracji.
* `8` - Komendy do administracji systemem.


* `mandb` – Aktualizuje bazę manuali (Musi zostać wykonane jako ROOT, inaczej wywala błąd czyszczenia plików).
* `/usr/share/doc` – Obszerna, dodatkowa dokumentacja pakietów (np. bind, syslog).
* `pinfo` – Pozwala na przeglądanie stron manuali i info z użyciem nawigacji przypominającej hiperlinki.

---

## Rozdział 3 - Mounting of directories

### Dyski i Inody

* `mount` – Zwraca pełny wykaz zamontowanych zasobów, zaczytując dane z `/proc/mounts`.
* `findmnt` – Działa podobnie jak `mount`, ale prezentuje dane w bardzo czytelnej formie drzewiastej.
* `df -Th` – Weryfikuje dostępne miejsce. Flaga `-T` dodatkowo wyświetla kolumnę z typem systemu plików.
* **Inody (Inodes)** – Unikalne identyfikatory fragmentów przestrzeni na dysku, gdzie faktycznie składowane są dane metadane. Pliki w systemie to tylko "linki" prowadzące do inoda.
* Jeśli z danym inodem nie jest już powiązany żaden "plik" (licznik odwołań to zero), dane zostają nadpisane/zwolnione.
* Jeśli skasujesz plik, który jest w danym momencie otwarty do edycji, program dalej może go zmieniać – inode zostanie zwolniony dopiero po zamknięciu procesu.
* Edytory takie jak **VIM** podczas zapisu często zapisują zawartość do nowego inoda (pod nową nazwą), a po zamknięciu podmieniają inode oryginalnego pliku.



### Operacje na plikach (Listowanie, kopiowanie, usuwanie)

* Opcje polecenia `ls`:
* `-l` – Długi format z informacjami o uprawnieniach, ownerze, rozmiarze.
* `-a` – Wyświetla pliki ukryte (zaczynające się od kropki).
* `-t` – Sortuje po dacie modyfikacji.
* `-r` – Odwraca kolejność sortowania (reverse).
* `-R` – Przeszukuje rekursywnie wszystkie podkatalogi.
* `-i` – Pokazuje numery przypisanych inodów.


* Kopiowanie:
* `-R` – Flaga do kopiowania rekursywnego.
* `-a` – Kopiuje zasoby zachowując dokładnie ich uprawnienia, czas modyfikacji itp.
* `cp -r /KATALOG/ /CEL/` – Kopiuje katalog wraz z podkatalogami.
* `cp -a /katalog/. /CEL/` – Kopiuje wszystkie pliki (w tym ukryte) – wymusza to dodanie kropki na końcu ścieżki źródłowej.


* Opcje polecenia `rm`:
* `-f` – (force) Usuwa pliki bez rzucania promptu o zgodę.



### Linki

Składnia: `ln [OPCJE] DOCELOWY_PLIK TWORZONY_LINK`

* **Hard Links (Twarde):**
* Fizycznie ten sam inod na tym samym urządzeniu blokowym.
* Nie można tworzyć hard linków na innych partycjach.
* Nie można ich podpiąć pod foldery.
* Plik usunie się fizycznie z dysku dopiero, gdy usuniesz ostatni przypięty alias (ostatni hard link).
* W systemach RHEL 7+ musisz być właścicielem docelowego pliku, aby utworzyć do niego hard link.


* **Symbolic / Soft Links (Miękkie):**
* Flaga `-s` dla polecenia `ln`.
* Mogą linkować pliki pomiędzy partycjami.
* Mają możliwość linkowania katalogów.
* Jak usuniesz plik źródłowy/target, link symboliczny traci rację bytu (jest uszkodzony).



### Kompresja i archiwizacja

Składnia archiwizatora `tar`:
`tar [OPCJE] DOCELOWY_PLIK CO_ARCHIWIZUJEMY`

* Klawisze opcji `tar`:
* `-c` – Create (tworzenie archiwum).
* `-x` – Extract (rozpakowywanie). Podanie po pliku docelowym parametru z dokładną nazwą wypakuje tylko jeden specyficzny plik. Flaga `-C` ustala folder docelowy dla wypakowania.
* `-f` – File, podanie nazwy pliku archiwum (zawsze powinna być podawana jako ostatnia w bloku, np. `-czf`).
* `-v` – Verbose (lista procesowanych plików).
* `-r` – Dodanie elementu do istniejącego (nieskompresowanego) archiwum.
* `-u` – Update (zaktualizowanie modyfikowanych plików) w archiwum.
* `-z` – Automatyczne przepuszczenie archiwum przez algorytm **GZIP** (tylko w momencie tworzenia/wypakowania).
* `-j` – Automatyczne przepuszczenie archiwum przez algorytm **BZIP2** (tylko w momencie tworzenia/wypakowania).


* Rozpakowywanie natywnymi formatami to domyślnie flaga `-d` (od decompress), np. `gzip -d`.
* `zip` i `unzip`:
* Narzędzia stworzone m.in. dla kompatybilności z systemami MS Windows. Domyślnie na RHEL często nie są zainstalowane.
* Kompresja katalogu: `zip -r NAZWA_PLIKU.zip /CO_ZROBIC`
* Dekompresja: `unzip NAZWA_PLIKU.zip`




#######################

# Resetowanie hasła roota w RHEL 10

W systemach bazujących na RHEL 9 i 10 (z systemd i zaktualizowanym procesem bootowania) procedura wygląda następująco:

1. Uruchom ponownie system. Gdy pojawi się menu GRUB, zatrzymaj odliczanie (np. strzałkami).
2. Wciśnij `e` na wybranym wpisie kernela, aby go zedytować.
3. Znajdź linię zaczynającą się od `linux` (lub `linuxefi` / `linux16`).
4. Dopisz na samym końcu tej linii parametr: `rd.break`
5. Wciśnij `Ctrl + X`, aby zbootować system z wprowadzonym parametrem. System zatrzyma się w powłoce awaryjnej (initramfs).
6. Zamontuj główny system plików (sysroot) w trybie do odczytu i zapisu:
```bash
mount -o remount,rw /sysroot

```


7. Przejdź do właściwego systemu plików używając chroot:
```bash
chroot /sysroot

```


8. Zmień hasło roota:
```bash
passwd root

```


9. Wymuś na systemie ponowne przypisanie kontekstów SELinux przy kolejnym uruchomieniu (niezbędne, w przeciwnym razie nie zalogujesz się):
```bash
touch /.autorelabel

```


10. Wyjdź z chroota i powłoki awaryjnej:
```bash
exit
exit

```



System zresetuje się (proces przeliczania etykiet SELinux może potrwać kilka minut) i uruchomi normalnie.

---

# Wyłączenie komunikatu Subscription Manager

Menedżer pakietów domyślnie korzysta z wtyczki `subscription-manager`, która przy każdej akcji weryfikuje status rejestracji systemu. Aby używać lokalnych repozytoriów bez komunikatów o braku subskrypcji, należy tę wtyczkę wyłączyć.

Otwórz plik konfiguracyjny (w RHEL 10 DNF korzysta z poniższych ścieżek):

```bash
vi /etc/dnf/plugins/subscription-manager.conf

```

Zmień parametr `enabled` z `1` na `0`:

```ini
[main]
enabled=0

```

Zapisz plik. Od teraz `dnf` będzie pobierał dane wyłącznie z podpiętych, lokalnych repozytoriów bez ostrzeżeń ze strony Red Hata.

---

# ROZDZIAŁ 2 - Essential tools

### Przekierowania i potoki (Pipes)

* `>` – Przekierowuje standardowe wyjście (STDOUT), nadpisując plik.
* `>>` – Przekierowuje STDOUT, dopisując na koniec pliku (append).
* `COMMAND > output 2>&1` – Przekierowuje zarówno STDOUT, jak i STDERR (błędy) do tego samego pliku (deskryptor błędów nr 2 jest kierowany tam, gdzie standardowe wyjście nr 1).
* Przekierowania mogą być łączone i są ewaluowane od lewej do prawej:
```bash
sort < jakis_plik > drugi_plik_na_output

```


* **Różnica między przekierowaniem a potokiem (`|`)**: Przekierowanie pobiera lub zapisuje dane bezpośrednio do pliku. Pipe bierze wyjście (STDOUT) jednej komendy i pcha je bezpośrednio na wejście (STDIN) drugiej komendy.

### Komendy wbudowane (Builtins) i weryfikacja plików

* **Internal commands** (wbudowane w powłokę): `echo`, `printf`, `read`, `cd`, `pwd`, `pushd`, `popd`, `dirs`.
* `type KOMENDA` – Pozwala sprawdzić, czy komenda jest wbudowana, czy zewnętrzna.
* `which KOMENDA` – Wyświetla ścieżkę pliku binarnego danej komendy.

### Historia (`history`)

* `history` – Pokazuje historię wykonanych poleceń.
* `Ctrl+R` – Wyszukuje po fragmencie tekstu. Ponowne wciśnięcie szuka kolejnego wystąpienia.
* `!numer` – Wykonuje natychmiast komendę pod danym numerem z historii.
* `!tekst` – Wykonuje natychmiast ostatnią komendę zaczynającą się od danego tekstu (UWAGA: brak potwierdzenia).
* `history -c` – Czyści historię poleceń w pamięci podręcznej (aktualna sesja).
* `history -w` – Zapisuje (lub nadpisuje, czyszcząc, jeśli użyto `-c`) plik `.bash_history`.

### Skrypty logowania powłoki

* `~/.bash_profile` – Wykonywany tylko raz, przy uruchamianiu powłoki logowania (login shell).
* `~/.bashrc` – Czytany za każdym razem, gdy uruchamiana jest dowolna powłoka bash. Konfiguracja w nim powinna być możliwie "lekka", aby nie opóźniać startu skryptów i sub-shelli.
* `/etc/issue` – Komunikat wyświetlany **przed** logowaniem użytkownika do systemu.
* `/etc/motd` – (Message of the Day) Komunikat wyświetlany **po** pomyślnym zalogowaniu.

### Pomoc systemowa (Man / Info)

* `apropos KEYWORD` (lub `man -k`) – Przeszukuje ogólne opisy w podręczniku, przydatne, gdy nie pamiętasz dokładnej nazwy komendy.
* `man -f KOMENDA` – Wyświetla skrócony opis komendy.
* **Kluczowe kategorie Man:**
* `1` - Programy i komendy powłoki.
* `5` - Formaty plików i konwencje konfiguracji.
* `8` - Komendy do administracji systemem.


* `mandb` – Aktualizuje bazę manuali (Musi zostać wykonane jako ROOT, inaczej wywala błąd czyszczenia plików).
* `/usr/share/doc` – Obszerna, dodatkowa dokumentacja pakietów (np. bind, syslog).
* `pinfo` – Pozwala na przeglądanie stron manuali i info z użyciem nawigacji przypominającej hiperlinki.

---

# ROZDZIAŁ 3 - Mounting of directories

### Dyski i Inody

* `mount` – Zwraca pełny wykaz zamontowanych zasobów, zaczytując dane z `/proc/mounts`.
* `findmnt` – Działa podobnie jak `mount`, ale prezentuje dane w bardzo czytelnej formie drzewiastej.
* `df -Th` – Weryfikuje dostępne miejsce. Flaga `-T` dodatkowo wyświetla kolumnę z typem systemu plików.
* **Inody (Inodes)** – Unikalne identyfikatory fragmentów przestrzeni na dysku, gdzie faktycznie składowane są dane metadane. Pliki w systemie to tylko "linki" prowadzące do inoda.
* Jeśli z danym inodem nie jest już powiązany żaden "plik" (licznik odwołań to zero), dane zostają nadpisane/zwolnione.
* Jeśli skasujesz plik, który jest w danym momencie otwarty do edycji, program dalej może go zmieniać – inode zostanie zwolniony dopiero po zamknięciu procesu.
* Edytory takie jak **VIM** podczas zapisu często zapisują zawartość do nowego inoda (pod nową nazwą), a po zamknięciu podmieniają inode oryginalnego pliku.



### Operacje na plikach (Listowanie, kopiowanie, usuwanie)

* Opcje polecenia `ls`:
* `-l` – Długi format z informacjami o uprawnieniach, ownerze, rozmiarze.
* `-a` – Wyświetla pliki ukryte (zaczynające się od kropki).
* `-t` – Sortuje po dacie modyfikacji.
* `-r` – Odwraca kolejność sortowania (reverse).
* `-R` – Przeszukuje rekursywnie wszystkie podkatalogi.
* `-i` – Pokazuje numery przypisanych inodów.


* Kopiowanie:
* `-R` – Flaga do kopiowania rekursywnego.
* `-a` – Kopiuje zasoby zachowując dokładnie ich uprawnienia, czas modyfikacji itp.
* `cp -r /KATALOG/ /CEL/` – Kopiuje katalog wraz z podkatalogami.
* `cp -a /katalog/. /CEL/` – Kopiuje wszystkie pliki (w tym ukryte) – wymusza to dodanie kropki na końcu ścieżki źródłowej.


* Opcje polecenia `rm`:
* `-f` – (force) Usuwa pliki bez rzucania promptu o zgodę.



### Linki

Składnia: `ln [OPCJE] DOCELOWY_PLIK TWORZONY_LINK`

* **Hard Links (Twarde):**
* Fizycznie ten sam inod na tym samym urządzeniu blokowym.
* Nie można tworzyć hard linków na innych partycjach.
* Nie można ich podpiąć pod foldery.
* Plik usunie się fizycznie z dysku dopiero, gdy usuniesz ostatni przypięty alias (ostatni hard link).
* W systemach RHEL 7+ musisz być właścicielem docelowego pliku, aby utworzyć do niego hard link.


* **Symbolic / Soft Links (Miękkie):**
* Flaga `-s` dla polecenia `ln`.
* Mogą linkować pliki pomiędzy partycjami.
* Mają możliwość linkowania katalogów.
* Jak usuniesz plik źródłowy/target, link symboliczny traci rację bytu (jest uszkodzony).



### Kompresja i archiwizacja

Składnia archiwizatora `tar`:
`tar [OPCJE] DOCELOWY_PLIK CO_ARCHIWIZUJEMY`

* Klawisze opcji `tar`:
* `-c` – Create (tworzenie archiwum).
* `-x` – Extract (rozpakowywanie). Podanie po pliku docelowym parametru z dokładną nazwą wypakuje tylko jeden specyficzny plik. Flaga `-C` ustala folder docelowy dla wypakowania.
* `-f` – File, podanie nazwy pliku archiwum (zawsze powinna być podawana jako ostatnia w bloku, np. `-czf`).
* `-v` – Verbose (lista procesowanych plików).
* `-r` – Dodanie elementu do istniejącego (nieskompresowanego) archiwum.
* `-u` – Update (zaktualizowanie modyfikowanych plików) w archiwum.
* `-z` – Automatyczne przepuszczenie archiwum przez algorytm **GZIP** (tylko w momencie tworzenia/wypakowania).
* `-j` – Automatyczne przepuszczenie archiwum przez algorytm **BZIP2** (tylko w momencie tworzenia/wypakowania).


* Rozpakowywanie natywnymi formatami to domyślnie flaga `-d` (od decompress), np. `gzip -d`.
* `zip` i `unzip`:
* Narzędzia stworzone m.in. dla kompatybilności z systemami MS Windows. Domyślnie na RHEL często nie są zainstalowane.
* Kompresja katalogu: `zip -r NAZWA_PLIKU.zip /CO_ZROBIC`
* Dekompresja: `unzip NAZWA_PLIKU.zip`



---

# Wyszukiwanie plików

Do zaawansowanego wyszukiwania służy polecenie `find`. Działa rekursywnie i przeszukuje strukturę katalogów w czasie rzeczywistym.

**Składnia:**
`find [ŚCIEŻKA] [OPCJE] [AKCJE]`

**Podstawowe kryteria:**

* `-name PATTERN` – Szuka po nazwie pliku (obsługuje wyrażenia regularne i wildcardy).
* `-iname PATTERN` – Szuka po nazwie, ignorując wielkość liter (case-insensitive).
* `-type f/d/l` – Szuka konkretnego typu zasobu: `f` (plik), `d` (katalog), `l` (symlink).

**Wyszukiwanie po uprawnieniach i właścicielu:**

* `-user NAZWA_LUB_UID` – Pliki należące do konkretnego użytkownika.
* `-group NAZWA_LUB_GID` – Pliki należące do konkretnej grupy.
* `-perm TRYB` – Szuka po uprawnieniach.
* `-perm 644` – Dokładnie takie uprawnienia.
* `-perm -644` – Przynajmniej takie uprawnienia (może mieć więcej).
* `-perm /644` – Dowolny z bitów dopasowany.


* **Bity specjalne (SUID, SGID, Sticky Bit):**
* `-perm /4000` – Pliki z ustawionym bitem SUID (wykonanie z prawami właściciela).
* `-perm /2000` – Pliki z ustawionym bitem SGID (np. dziedziczenie grupy w katalogu).
* `-perm /1000` – Pliki z ustawionym Sticky Bit (np. `/tmp`, tylko właściciel może usunąć swój plik).



**Wyszukiwanie po czasie (modyfikacja i dostęp):**

* `-mtime N` – Czas modyfikacji zawartości pliku w dniach (N dni temu). `-mtime -N` (mniej niż N dni), `-mtime +N` (więcej niż N dni).
* `-mmin N` – Czas modyfikacji w minutach.
* `-atime N` / `-amin N` – Czas ostatniego dostępu (odczytu) do pliku.
* `-ctime N` / `-cmin N` – Czas zmiany metadanych (np. uprawnień, inoda).

**Akcje wykonywane na znalezionych plikach:**

* `-exec KOMENDA {} \;` – Wykonuje komendę na każdym znalezionym pliku (np. `find / -name "*.tmp" -exec rm -f {} \;`).
* `-delete` – Szybkie usuwanie znalezionych plików (zastępuje `-exec rm`).

### Locate

`locate` jest znacznie szybsze niż `find`, ponieważ nie odpytuje dysku, a jedynie przeszukuje bazę danych indeksu.

* Baza odświeżana jest raz dziennie cronem: `/etc/cron.daily/mlocate`.
* Domyślna ścieżka bazy: `/var/lib/mlocate/mlocate.db`.
* Odświeżenie bazy na żądanie: `updatedb`.

---

# ROZDZIAŁ 4 - TEXT FILES

### Przeglądanie zawartości

* `less` – Pager do przeglądania plików bez wczytywania całości do pamięci.
* `G` – Skok na koniec pliku.
* `g` – Skok na początek pliku.
* `/fraza` – Wyszukiwanie w dół (klawisz `n` szuka kolejnego, `N` poprzedniego wystąpienia).
* `?fraza` – Wyszukiwanie w górę.


* `cat` – Wypisuje całą zawartość na STDOUT.
* `tac` – Odwrócony `cat`, wypisuje linie od dołu do góry.
* `head` / `tail` – Wypisują początek / koniec pliku. Domyślnie 10 linii. Ogranicznik: `-n NUMER`. Flaga `-f` w tail śledzi zmiany w pliku na żywo (przydatne do logów).

### VIM – Podstawy

Wielotrybowy edytor tekstowy, używany powszechnie na serwerach z racji swojej lekkości i braku zależności od GUI.

* **Tryb normalny (Command Mode):** Domyślny po otwarciu. Klawisze to komendy (np. `dd` usuwa linię, `yy` kopiuje linię, `p` wkleja, `u` cofa zmianę).
* **Tryb wstawiania (Insert Mode):** Wchodzi się wciskając `i` (insert) lub `a` (append). Służy do pisania. Wyjście do trybu normalnego przez `ESC`.
* **Tryb poleceń (Command-line Mode):** Otwierany dwukropkiem `:` w trybie normalnym.
* `:wq` lub `:x` – Zapisz i wyjdź.
* `:q!` – Wyjdź odrzucając zmiany.



### Strumieniowe przetwarzanie i modyfikacja tekstu

* **tr (Translate)** – Służy do zamiany lub usuwania pojedynczych znaków (czyta ze STDIN).
* Zamiana małych na wielkie: `cat plik | tr 'a-z' 'A-Z'`
* Usuwanie powielonych spacji (squeeze): `tr -s ' '`
* Usuwanie konkretnego znaku: `tr -d '\n'` (usuwa znaki nowej linii).


* **sed (Stream Editor)** – Potężne narzędzie do modyfikacji strumienia danych w oparciu o wyrażenia regularne.
* Zastępowanie pierwszego wystąpienia w linii: `sed 's/stare/nowe/' plik`
* Zastępowanie wszystkich wystąpień w linii (global): `sed 's/stare/nowe/g' plik`
* Modyfikacja pliku w miejscu (inline): `sed -i 's/stare/nowe/g' plik` (nadpisuje plik źródłowy).


* **cut** – Wycina konkretne pola z pliku tekstowego na podstawie delimitera (separatora).
* `-d` – Określa separator.
* `-f` – Określa numer kolumny (liczone od 1).
* Przykład: `cut -d ':' -f 1 /etc/passwd` (Wyciąga same nazwy użytkowników).


* **sort** – Sortuje linie alfabetycznie.
* `-n` – Sortowanie numeryczne.
* `-r` – Odwrócona kolejność (reverse).
* `-t` – Definiuje separator kolumn.
* `-kX` – Wskazuje kolumnę `X` do posortowania (np. `sort -t ':' -k3 -n /etc/passwd`).
* `-u` – (Unique) Od razu sortuje i usuwa duplikaty. Działa jak połączenie `sort` i `uniq`.


* **uniq** – Narzędzie do filtrowania i raportowania powtarzających się linii. **Ważne:** `uniq` analizuje tylko linie występujące bezpośrednio po sobie, dlatego przed jego użyciem plik niemal zawsze trzeba posortować!
* Podstawowe usunięcie duplikatów: `sort plik.txt | uniq` (pozostawi tylko unikalne wpisy).
* `-c` – (Count) Zlicza wystąpienia danej linii. Klasyczny trik administracyjny na sprawdzanie np. logów to: `sort plik.txt | uniq -c | sort -nr` (najpierw sortuje nazwy, potem `uniq` liczy ich wystąpienia, a na koniec drugi `sort` układa wynik malejąco po liczbie wystąpień).
* `-d` – (Duplicate) Wyświetla **tylko** te linie, które się powtarzają.


* **wc (Word Count)** – Zlicza metryki w pliku. Najczęściej używane: `-l` (linie), `-w` (słowa), `-c` (bajty/znaki).

### Wyszukiwanie wewnątrz plików (grep)

Narzędzie do wyłapywania linii pasujących do podanego wzorca. Domyślnie case-sensitive.

* `-i` – Ignoruje wielkość liter.
* `-v` – Odwraca dopasowanie (zwraca linie, które **nie zawierają** wzorca).
* `-r` – Przeszukuje rekursywnie wszystkie pliki w podanym katalogu.
* `-w` – Dopasowuje tylko całe słowa (np. szukając 'test', pominie słowo 'testowy').
* `-e` – Pozwala zdefiniować wiele wzorców jednocześnie (logiczne OR): `grep -e 'root' -e 'admin' plik`.
* `-A N` / `-B N` / `-C N` – Pokazuje `N` linii po (After), przed (Before) lub wokół (Context) znalezionego dopasowania.

---

# ROZDZIAŁ 5 - LOGOWANIE DO SYSTEMU

**Terminale i powłoki:**
Wirtualne terminale (TTY) otwierane skrótem `Alt+F1` do `F6`. Przełączanie ręczne komendą `chvt N`.

* **Różnice między terminalami:** Z technicznego punktu widzenia terminale wirtualne od `tty1` do `tty6` są identyczne – korzystają z tego samego demona (historycznie `getty`/`mingetty`, obecnie `systemd-logind`) i mają dokładnie takie same uprawnienia (w tym dostęp do sieci). Różnice wynikały jedynie z przyjętych konwencji i celowej konfiguracji systemu:
* Historycznie GUI (X11) było sztywno przypinane do `tty7`, co zostawiało konsole 1-6 czysto tekstowymi. (Obecnie Wayland/GDM często startuje na `tty1` lub `tty2`).
* Konsole od `tty8` do `tty12` były często konfigurowane w `syslogu` do wypluwania logów systemowych na żywo (np. logi kernela leciały prosto na ekran `tty12`, aby admin mógł je widzieć bez logowania).
* Każdy otwarty TTY jest całkowicie niezależną sesją logowania, pozwalającą na równoległą pracę na tej samej maszynie.


* Jeśli GUI blokuje F1, używa się `Ctrl+Alt+F2` do `F6`.
* `tty1-tty6` to fizyczne konsole. Wirtualne emulatory okienkowe korzystają z pseudo-terminali (np. `pts/0`).

**SSH i transfer plików:**

* Składnia: `ssh [OPCJE] USER@HOST`
* Opcje `ssh`:
* `-l` – Definiowanie użytkownika, jeśli nie użyto notacji `user@host`.
* `-i` – Wskazuje lokalizację klucza prywatnego (domyślnie `~/.ssh/id_rsa`).
* `-F` – Wskazuje własny plik konfiguracyjny SSH (domyślnie `~/.ssh/config` per user, ogólny `/etc/ssh/ssh_config`).
* `-X` / `-Y` – X11 Forwarding (uruchamianie aplikacji GUI z serwera na stacji klienckiej). Serwer musi mieć w `/etc/ssh/sshd_config` ustawione `X11Forwarding yes`.
* `-p` – Niestandardowy port.


* Narzędzia powiązane:
* `ssh-keygen` – Generuje parę kluczy szyfrujących (prywatny/publiczny).
* `ssh-copy-id USER@HOST` – Wysyła i dopisuje klucz publiczny do pliku `~/.ssh/authorized_keys` na docelowym serwerze.


* **scp (Secure Copy)** – Narzędzie do przesyłania plików po protokole SSH.
* Wysyłanie do serwera: `scp /lokalny/plik.txt user@host:/tmp/`
* Pobieranie z serwera: `scp user@host:/tmp/plik.txt /lokalny/folder/`
* `-r` – Kopiowanie rekursywne całych katalogów.
* `-P` – Wielkie 'P', definiuje port dla SSH.


* Weryfikacja sesji:
* `who` lub `w` – Pokazuje listę zalogowanych użytkowników i ich procesy.



---

# ROZDZIAŁ 6 - USER AND GROUP MANAGEMENT

Root w systemie ma absolutny dostęp, omijający wszystkie restrykcje plików (ACL, standardowe uprawnienia). Nie ma konta root bez pełnych praw (wyjątek: polityki SELinux mogą go ograniczyć).

### Informacje o kontach

* `id [USER]` – Wyświetla numery UID, GID i przynależność do grup.
* `getent database [KLUCZ]` – Wyciąga wpisy z systemowych baz danych, uwzględniając zewnętrzne źródła (np. LDAP/SSSD). `getent passwd testuser`.

### Przełączanie tożsamości

* `su` – Zamienia użytkownika.
* Logowanie interaktywne bez pełnego środowiska (Interactive Shell): `su` (tylko ładuje `.bashrc`).
* Pełne logowanie z inicjalizacją zmiennych środowiskowych (Login Shell): `su -` (ładuje `.bash_profile` i zachowuje się jak pełnoprawne zalogowanie).
* Wykonanie pojedynczej komendy jako inny user: `su -c 'komenda' user`.



### Modyfikacja użytkowników i grup

* `useradd` – Tworzenie usera.
* `-m` – Wymusza stworzenie katalogu domowego z plików źródłowych znajdujących się w `/etc/skel/`.
* `-u` – Ręczne nadanie konkretnego UID.
* `-G` – Dodanie do grup dodatkowych podczas tworzenia (np. `group1,group2`).


* Domyślne wartości dla tworzonych kont pochodzą z:
* `/etc/login.defs` (ustawienia wygasania, UID_MIN, polityki).
* `/etc/default/useradd` (domyślny shell, katalog domowy, parametry tworzenia).


* `usermod` – Modyfikacja istniejącego usera.
* Zmiana powłoki: `-s /bin/bash`.
* Dodanie do grupy: `-aG GRUPA USER` (OSTRZEŻENIE: brak flagi `-a` zresetuje grupy dodatkowe tylko do tej jednej określonej w poleceniu `-G`). Grupa `wheel` to na RHEL domyślna grupa administratorów `sudo`.
* Blokowanie/odblokowywanie konta: `-L` (Lock) / `-U` (Unlock).
* Zmiana komentarza (pola GECOS): `-c "komentarz"`.
* Zmiana katalogu domowego (z migracją danych): `-d /nowy/home -m`.


* `userdel` – Usunięcie usera. Dodanie `-r` wyczyści również jego katalog domowy oraz maile.
* `groupadd` / `groupmod` – Zarządzanie grupami. Dodawanie użytkowników do grup zawsze realizuje się z poziomu komendy `usermod`. Flaga `-g` zmienia numer GID.
* `groupmems -g GRUPA -l` – Wylistowanie wszystkich członków danej grupy.

### Zarządzanie ważnością hasła i konta (chage i passwd)

**1. Zmiana globalnych wartości domyślnych (dla nowych użytkowników):**
Aby wszystkie nowo tworzone konta w systemie miały odgórnie narzuconą politykę haseł, należy wyedytować plik `/etc/login.defs`. Znajdują się tam kluczowe zmienne:

* `PASS_MAX_DAYS 90` – Hasło wygasa po 90 dniach.
* `PASS_MIN_DAYS 7` – Hasło nie może być zmienione częściej niż co 7 dni (zapobiega to omijaniu historii haseł przez natychmiastowe rotowanie).
* `PASS_WARN_AGE 14` – System zacznie ostrzegać o wygasaniu hasła 14 dni przed czasem.
* *Uwaga:* Zmiana w tym pliku nie zadziała wstecz – konta utworzone wcześniej zachowają swoje stare ustawienia zapisane w `/etc/shadow`.

**2. Zarządzanie istniejącymi użytkownikami za pomocą `chage` i `passwd`:**

* `passwd` – Zmiana hasła.
* `-l` / `-u` – Lock/Unlock hasła (dopisuje `!!` do skrótu hasła w `/etc/shadow`).


* `chage` (od Change Age) – Służy do modyfikowania polityki wygasania konkretnego konta.
* `chage -l USER` – Wyświetla czytelne podsumowanie obecnych restrykcji dla konta (kiedy hasło zostało zmienione, kiedy wygasa, kiedy wygasa całe konto).
* `chage -d 0 USER` – (Trik z zerowym dniem) Ustawia datę ostatniej zmiany hasła na początek epoki uniksowej (0), co wymusza na użytkowniku **natychmiastową zmianę hasła** przy najbliższym logowaniu.
* `chage -M DNI USER` – Ustala, co ile dni (Max) hasło musi zostać zmienione (np. `chage -M 30 jkowalski` wymusi zmianę co miesiąc).
* `chage -m DNI USER` – Ustala minimalny czas między zmianami.
* `chage -W DNI USER` – Ustala czas ostrzegania (Warning) przed wygaśnięciem.
* `chage -I DNI USER` – (Inactive) Ustawia "okres karencji". Jeśli hasło wygaśnie, konto nie jest od razu blokowane. Przez X dni użytkownik może jeszcze zalogować się z użyciem starego hasła *wyłącznie* po to, by je zmienić. Po tym czasie konto blokuje się na twardo.
* `chage -E YYYY-MM-DD USER` – (Expire) Ustawia bezwzględną datę wygaśnięcia **całego konta** (nie tylko hasła). Świetne do zarządzania kontami tymczasowymi (np. `chage -E 2026-12-31 stazysta`). Zablokowanie konta w ten sposób wymaga interwencji admina (`chage -E -1` wyłącza to ograniczenie).



### Pliki systemowe

Używanie edytorów bezpośrednio do tych plików nie jest zalecane bez odpowiednich wrapperów ze względu na brak blokad zapisu i weryfikacji składni (można uszkodzić logowanie do systemu). Używamy `visudo` dla sudoers, `vipw` dla passwd/shadow, `vigr` dla group.

* `/etc/passwd`: `username:password_placeholder:UID:GID:GECOS:home_dir:shell` (gdzie shell `/sbin/nologin` blokuje dostęp do interaktywnej konsoli i wyświetla zawartość `/etc/nologin.txt`).
* `/etc/shadow`: `username:hash:ostatnia_zmiana:min_dni:max_dni:ostrzeżenie:okres_blokady:wygaśnięcie_konta:zarezerwowane`.
* `/etc/group`: `grupa:hasło_grupy:GID:lista_użytkowników`.

**Środowisko startowe powłoki:**

* `/etc/profile` – Ustawienia globalne dla systemowych powłok logowania.
* `/etc/bashrc` – Globalne aliasy i funkcje dla wszystkich powłok (w tym subshelli).
* `~/.bash_profile` – Konfiguracja usera dla powłok logowania (np. własne `$PATH`).
* `~/.bashrc` – Konfiguracja usera dla każdej nowo otwartej konsoli interaktywnej.

---

# Zarządzanie tożsamością i logowanie sieciowe (SSSD / LDAP / AD)

Dawniej opierano się na instalowaniu klienta LDAP (np. daemon `nslcd`) i konfiguracji przez zdeprecjonowane polecenie `authconfig`. W nowoczesnych dystrybucjach (RHEL 8/9/10, CentOS) standardem jest **SSSD (System Security Services Daemon)**, a system uwierzytelniania konfiguruje się poprzez `authselect` lub `realmd`.

### SSSD

SSSD to demon działający jako pośrednik między lokalnym systemem operacyjnym a zdalnymi katalogami (Active Directory, FreeIPA, serwery LDAP).

* **Do czego służy?** SSSD autoryzuje użytkowników z domeny sieciowej tak, jakby istnieli lokalnie.
* **Zalety:** Obejmuje cache'owanie poświadczeń (pozwala na logowanie domenowe "offline", np. gdy laptop utraci łączność z serwerem AD), upraszcza zarządzanie w dużych infrastrukturach.
* **Konfiguracja:** Znajduje się w pliku `/etc/sssd/sssd.conf`. Aby SSSD poprawnie działało, plik ten musi mieć restrykcyjne uprawnienia `chmod 600`.

### Realmd

Najpopularniejsze i najprostsze narzędzie (wymaga paczek `realmd` oraz `sssd`) do podpinania maszyny z systemem Linux pod usługę katalogową (AD / FreeIPA). `Realmd` pod spodem automatycznie generuje pliki dla SSSD i konfiguruje mechanizmy PAM.

1. Sprawdzenie/wykrycie domeny:
```bash
realm discover AD_SERVER.DOMENA.LOCAL

```


2. Podpięcie do domeny (wymaga poświadczeń administratora usługi katalogowej):
```bash
realm join AD_SERVER.DOMENA.LOCAL -U Administrator

```



Po pomyślnym zintegrowaniu, można logować się zdalnie na serwer z użyciem tożsamości domenowej:
`ssh user@domena.local@ADRES_IP_SERWERA` (zależnie od ustawienia parametru `use_fully_qualified_names` w `sssd.conf`).

### Authselect (Zastępca authconfig)

Narzędzie to (dostępne z pudełka w nowych wydaniach RHEL) służy do przełączania profili PAM i `nsswitch.conf`. Zamiast ręcznie edytować flagi, jak w starym `authconfig --enableldap`, w `authselect` wybiera się profil.
Przykład wdrożenia uwierzytelniania SSSD po konfiguracji profilu na nowym RHEL:

```bash
authselect select sssd with-mkhomedir --force

```

Dodatek `with-mkhomedir` odpowiada dawnej opcji `--enablemkhomedir`, zapewniając automatyczne generowanie katalogów domowych z `/etc/skel` przy pierwszym zalogowaniu użytkownika domenowego, który w systemie lokalnym nie miał jeszcze przestrzeni.


############################

# ROZDZIAŁ 7 - CONFIGURATION PERMISSIONS

## 1. Weryfikacja typu pliku (ls -l)

Pierwszy znak w listowaniu `ls -l` określa typ zasobu:
* `-` : regular file (zwykły plik)
* `d` : directory (katalog)
* `l` : symbolic link (dowiązanie symboliczne)
* `b` : block device (urządzenie blokowe, np. dysk)
* `c` : character device (urządzenie znakowe, np. tty)

## 2. Właściciel, Grupa i Użytkownicy Systemowi

Podczas tworzenia pliku jego właścicielem (owner) zostaje użytkownik, który go utworzył, a właścicielem grupowym – podstawowa (primary) grupa tego użytkownika.

> **Wskazówka `find`:** Aby szybko znaleźć pliki należące do konkretnego użytkownika lub grupy, użyj:
> `find / -user nazwa_usera`
> `find / -group nazwa_grupy`

**Specjalni użytkownicy: `nobody` i `nogroup` (lub `nobody` w RHEL)**
W systemie spotkasz się z uprawnieniami przypisanymi do użytkownika `nobody`. Jest to specjalne konto o najniższych możliwych uprawnieniach systemowych (często bez dostępu do powłoki i jakichkolwiek plików). Używa się go w celach bezpieczeństwa – demony i usługi sieciowe (np. serwery webowe, NFS) często uruchamiają procesy potomne jako `nobody`, aby w przypadku włamania atakujący zyskał minimalne prawa w systemie.

### Zmiana właściciela i grupy (chown / chgrp)

* **`chown KTO DO_CZEGO`** – Zmienia właściciela pliku/katalogu. Flaga `-R` robi to rekursywnie.
* Zmiana właściciela i grupy jednocześnie: 
  * `chown user:grupa plik` lub `chown user.grupa plik`
* Zmiana samej grupy przez chown (wymaga dwukropka/kropki przed nazwą):
  * `chown :nowagrupa plik`
* **`chgrp GRUPA DO_CZEGO`** – Specjalizowana komenda wyłącznie do zmiany grupy (flagi takie same jak w chown).

### Zarządzanie własnymi grupami

* **`groups [USER]`** – Wyświetla grupy, do których należy użytkownik. Pierwsza na liście to grupa podstawowa (primary).
* **`newgrp nazwa_grupy`** – Zmienia grupę podstawową **tylko dla trwającej sesji**. Zazwyczaj musisz być członkiem tej grupy, by ją przypisać. Jeśli nie jesteś członkiem, komenda zapyta o hasło (jeśli grupa ma nadane hasło przez `gpasswd`).

---

## 3. Uprawnienia (chmod)

| Uprawnienie | Wartość | Działanie na PLIKU | Działanie na KATALOGU |
| :--- | :--- | :--- | :--- |
| **Read (r)** | 4 | Pozwala otworzyć i przeczytać plik. | Pozwala wylistować zawartość (np. `ls`). |
| **Write (w)** | 2 | Pozwala zmieniać zawartość pliku. | Pozwala tworzyć, usuwać pliki i modyfikować ich atrybuty w tym katalogu. |
| **Execute (x)** | 1 | Pozwala uruchomić plik (np. skrypt, binarkę). | Pozwala wejść do katalogu (np. `cd`). Domyślne i niezbędne dla katalogów! |

Komenda **`chmod`** działa w dwóch trybach:

**1. Tryb liczbowy (oktalny):**
Podajemy sumę uprawnień dla Ownera/Grupy/Innych (Others).
* `chmod 755 skrypt.sh` (Owner: rwx, Grupa: r-x, Others: r-x)
* `chmod 644 notatka.txt` (Owner: rw-, Grupa: r--, Others: r--)

**2. Tryb relatywny (KTO +|-|= CO):**
* `u` (user), `g` (group), `o` (others), `a` (all)
* `chmod u+x skrypt.sh` – Dodaje wykonywanie dla właściciela.
* `chmod o-rwx plik` – Zabiera wszystkie prawa "innym".
* `chmod a=r plik` – Ustawia tylko odczyt dla wszystkich.
* `chmod -R o+rX /data` – Wielkie **`X`** jest bardzo przydatne! Nadaje prawo wykonywania (wejścia) tylko dla **katalogów** (nie psując przy tym zwykłych plików).

---

## 4. Uprawnienia Specjalne (SUID, SGID, Sticky Bit)

Uprawnienia te manipulują domyślnym zachowaniem procesów i współdzielenia plików. Zapisuje się je jako czwartą (pierwszą od lewej) cyfrę w `chmod` liczbowym.

* **SUID (Wartość: 4, Tryb relatywny: `u+s`)**
  * **Zastosowanie:** Tylko pliki wykonywalne (binarki). **Nie działa na skryptach w systemie Linux!**
  * **Jak działa:** Plik z tym bitem, po uruchomieniu przez dowolnego usera, wykonuje się z uprawnieniami **właściciela pliku**, a nie usera, który go odpalił. 
  * **Przykład:** Komenda `/usr/bin/passwd` ma właściciela `root` i ustawiony SUID. Zwykły user może jej użyć, aby zmienić swoje hasło, co wymaga zapisu do `/etc/shadow` (do którego normalnie nie ma dostępu).
  * **Widok w ls -l:** Zamiast `x` w sekcji właściciela pojawia się `s` (np. `-rwsr-xr-x`).

* **SGID (Wartość: 2, Tryb relatywny: `g+s`)**
  * **Zastosowanie:** Katalogi współdzielone (najczęstsze) lub pliki wykonywalne.
  * **Jak działa (na katalogu):** Wymusza dziedziczenie grupy. Każdy nowy plik lub podkatalog utworzony wewnątrz, automatycznie przyjmie grupę katalogu nadrzędnego, a nie grupę domyślną twórcy. Idealne do folderów projektowych dla zespołów.
  * **Widok w ls -l:** Zamiast `x` w sekcji grupy pojawia się `s` (np. `drwxrwsr-x`).

* **Sticky Bit (Wartość: 1, Tryb relatywny: `+t`)**
  * **Zastosowanie:** Katalogi z uprawnieniami do zapisu dla wszystkich (np. `/tmp`).
  * **Jak działa:** Zabezpiecza przed usuwaniem cudzych plików. Nawet jeśli katalog ma prawa zapisu dla wszystkich (`chmod 777`), włączenie Sticky Bit sprawia, że plik może zostać usunięty **tylko** przez jego właściciela lub właściciela katalogu docelowego.
  * **Widok w ls -l:** Na samym końcu uprawnień, w miejscu wykonywania dla `others`, pojawia się `t` (np. `drwxrwxrwt`).
  * **Przykład nadania:** `chmod 1777 /wspoldzielony_katalog` lub `chmod +t /wspoldzielony_katalog`.

---

## 5. UMASK (Domyślna maska uprawnień)

**UMASK** określa uprawnienia, które są **odejmowane** od domyślnych uprawnień maksymalnych przy tworzeniu nowych plików i katalogów.
* Maksymalne uprawnienia dla nowego **katalogu to 777** (rwxrwxrwx).
* Maksymalne uprawnienia dla nowego **pliku to 666** (rw-rw-rw-).

**Matematyka umask (np. domyślny 022):**
* Nowy plik: `666 - 022 = 644` (rw-r--r--)
* Nowy katalog: `777 - 022 = 755` (rwxr-xr-x)

> Wynik polecenia `umask` zwraca zazwyczaj 4 cyfry (np. `0022`). Pierwsza od lewej odpowiada za uprawnienia specjalne (SUID/SGID/Sticky bit). `umask 0` całkowicie zdejmuje restrykcje (niezalecane).

**Ustawianie umask na stałe:**
Uruchomienie `umask 027` w konsoli zmienia maskę tylko dla trwającej sesji. Aby zmiana była persystentna:
1. **Dla konkretnego usera:** Dopisujemy linię `umask 027` do pliku `~/.bashrc` lub `~/.profile` w katalogu domowym.
2. **Globalnie dla całego systemu:** Tworzymy dedykowany skrypt (np. `custom_umask.sh`) w katalogu `/etc/profile.d/` z zawartością `umask 022`.

---

## 6. ACL (Access Control Lists)

ACL pozwala nadać granularne uprawnienia do pliku lub katalogu specyficznym użytkownikom i grupom (wykraczając poza klasyczną triadę owner/group/others).
* **Wymagania:** System plików musi być zamontowany z obsługą ACL (opcja `acl` w `/etc/fstab` – w nowoczesnych systemach ext4/xfs jest to włączone domyślnie). Tar wspiera zachowanie uprawnień dzięki fladze `--acls`.

### Jak rozpoznać użycie ACL?
Jeśli na obiekcie ustawiono jakiekolwiek niestandardowe uprawnienia ACL, polecenie `ls -l` pokaże znak plusa **`+`** na samym końcu bloku uprawnień.
* Przykład bez ACL: `-rw-r--r--.`
* Przykład z ACL: `-rw-rwxr--+`

### Zarządzanie ACL
Do analizy uprawnień zawsze używamy **`getfacl NAZWA`**, ponieważ `ls -l` nie pokaże, kto dokładnie dostał te specjalne uprawnienia.

Zarządzanie odbywa się poprzez **`setfacl`**:
* **Ustawianie (Modify):** `setfacl -m u:user:uprawnienia plik` lub `setfacl -m g:grupa:uprawnienia plik`
  * Przykład: `setfacl -m u:jankowalski:rwx tajny_plik.txt` (Jan Kowalski dostaje pełne prawa, choć nie jest właścicielem).
  * Przykład: `setfacl -m g:avengers:rw /home/dev/project`
* **Usuwanie ACL:**
  * Usuń dla konkretnej grupy: `setfacl -x g:avengers /dir`
  * Wyczyść wszystkie ACL całkowicie: `setfacl -b /dir`
* **Dziedziczenie (Default ACL):** Aby uprawnienia działały nie tylko na obecny stan katalogu, ale wymuszały się na wszystkich **nowo** tworzonych plikach wewnątrz, stosujemy prefix **`d:`** (default):
  * `setfacl -m d:g:avengers:rw /katalog_projektu` (teraz każdy nowy plik stworzony w tym folderze dostanie prawa rw dla grupy avengers).
  * Usunięcie domyślnego ACL: `setfacl -k /dir`
* **Kopiowanie ACL miedzy zasobami:**
  * `getfacl plik_wzor | setfacl --set-file=- docelowy_plik`

---

## 7. Extended File Attributes (Rozszerzone Atrybuty)

Oprócz klasycznych uprawnień istnieją atrybuty jądra, zarządzane przez system plików (np. ext4, xfs). Są to ograniczenia wkraczające ponad uprawnienia roota.
* Weryfikacja: **`lsattr nazwa_pliku`**
* Zmiana: **`chattr [OPCJA] nazwa_pliku`**

**Kluczowe flagi atrybutów:**
* **`+i` (Immutable):** Plik staje się całkowicie niezmienny. Nikt, **nawet root**, nie może go usunąć, zmienić jego nazwy, zmodyfikować zawartości ani zrobić do niego dowiązania. (Zdjęcie blokady: `chattr -i`).
* **`+a` (Append only):** Plik można otworzyć tylko w trybie dopisywania (append). Używane często do zabezpieczania kluczowych plików logów – root może dopisywać zdarzenia, ale nie może zmodyfikować czy usunąć starych logów z pliku.

######

# ROZDZIAŁ 8 - SIECI

W nowoczesnych systemach RHEL (8/9/10), konfiguracja sieci opiera się na usłudze **NetworkManager**. 

> **UWAGA:** Komenda `ifconfig` jest przestarzała. Używamy zestawu narzędzi z rodziny `iproute2` oraz `nmcli`.

## 1. Wyświetlanie danych (Narzędzia `ip`)
Komenda `ip` jest najlepszym narzędziem do **diagnostyki** i podglądu stanu sieci. Zmiany wprowadzone za jej pomocą są **tymczasowe** i znikną po restarcie systemu.

* `ip addr show` (lub `ip a`): Wyświetla szczegółową konfigurację adresów IP dla wszystkich interfejsów.
* `ip link show`: Wyświetla stan interfejsów (up/down) oraz adresy MAC.
* `ip -s link`: Wyświetla interfejsy wraz ze statystykami przesłanych/odebranych pakietów.
* `ip route show`: Wyświetla tablicę routingu. 
    * *Złota zasada:* Router (brama domyślna) musi znajdować się w tej samej podsieci co interfejs.

## 2. Różnice: Interface, Connection, Link
W NetworkManagerze (NM) te pojęcia są rozdzielone:
* **Interface (Device):** Fizyczna lub wirtualna karta sieciowa w systemie (np. `eth0`, `enp0s3`). Widoczne w `/sys/class/net`.
* **Connection:** Profil konfiguracyjny (ustawienia IP, DNS, brama). Jeden interfejs może mieć wiele profili (np. "Biuro" i "Dom").
* **Link:** Warstwa fizyczna interfejsu (stan "kabla" lub "połączenia" w sensie warstwy 2 OSI).

## 3. Zarządzanie konfiguracją (NetworkManager)
Aby konfiguracja była **trwała** (przetrwała reboot), używamy `nmcli` (CLI) lub `nmtui` (TUI). Pliki w `/etc/sysconfig/network-scripts/` są historycznie istotne (używane w RHEL 7 i starszych), ale obecnie RHEL preferuje konfigurację przechowywaną wewnątrz NetworkManagera (systemowe pliki konfiguracyjne plików ifcfg są stopniowo wycofywane na rzecz plików `keyfile`).

### Nmcli – zarządzanie z CLI (Wymagane na egzaminie!)
`nmcli` jest Twoim głównym narzędziem. Nie polegaj na GUI na egzaminie, może ono nie być zainstalowane.

* `nmcli device status`: Pokazuje status wszystkich kart sieciowych.
* `nmcli connection show`: Pokazuje wszystkie zdefiniowane profile konfiguracji.
* `nmcli connection add type ethernet con-name "NazwaProfilu" ifname "enp0s3" ip4 192.168.1.10/24 gw4 192.168.1.1`: Dodanie nowego profilu.
* `nmcli connection up "NazwaProfilu"`: Aktywacja połączenia.
* `nmcli connection down "NazwaProfilu"`: Deaktywacja połączenia.

### Nmtui – tekstowy interfejs użytkownika
Wygodne narzędzie tekstowe. Mimo że w środowisku serwerowym GUI jest zazwyczaj niedostępne, `nmtui` jest często preinstalowane i stanowi akceptowalną alternatywę dla `nmcli`. Pozwala łatwo zmieniać DNSy, edytować połączenia i nazwę hosta (hostname).

## 4. Diagnostyka portów (`ss`)
Komenda `ss` (Socket Statistics) zastępuje przestarzały `netstat`.

* `ss -tulpn`:
    * `-t`: TCP
    * `-u`: UDP
    * `-l`: Listening (tylko porty w stanie oczekiwania)
    * `-p`: Pokaż procesy korzystające z portu
    * `-n`: Nie zamieniaj nazw portów na numery (szybsze działanie).
* **Interpretacja:** Jeśli port słucha na `127.0.0.1` lub `::1`, jest dostępny **tylko lokalnie**. Jeśli na `0.0.0.0` lub `:::`, słucha na wszystkich interfejsach sieciowych.

## 5. Konfiguracja DNS
DNSy konfigurujemy dla konkretnego profilu połączenia, aby NetworkManager automatycznie zarządzał plikiem `/etc/resolv.conf`. **Nigdy nie edytuj `/etc/resolv.conf` ręcznie**, ponieważ NetworkManager nadpisze te zmiany.

* **Przez nmcli:**
  ```bash
  nmcli con mod "NazwaProfilu" ipv4.dns "8.8.8.8 8.8.4.4"
  nmcli con up "NazwaProfilu"

 * Przez nmtui: Wejdź w "Edit a connection" -> Wybierz profil -> DNS -> Edit.
 * DHCP vs Statyczne: Jeśli karta pobiera IP z DHCP, może nadpisywać DNSy. Aby to zablokować:
   nmcli con mod "NazwaProfilu" ipv4.ignore-auto-dns yes

6. Hostname
Najlepszą metodą na zmianę nazwy hosta jest:
hostnamectl set-hostname "nazwa"

Zmiana ta jest trwała, aktualizuje /etc/hostname i jest widoczna dla systemu natychmiastowo.



# ROZDZIAŁ 9 - MANAGING PROCESÓW

W systemie Linux wyróżniamy dwa główne typy procesów:
* **Shell jobs (procesy interaktywne)** – procesy powiązane z sesją powłoki, w której zostały uruchomione.
* **Demony** – procesy tła uruchamiane najczęściej podczas startu systemu, zazwyczaj z uprawnieniami roota.

Procesy mogą składać się z jednego lub wielu wątków (**threads**). Zarządzanie pojedynczymi wątkami leży po stronie kodu aplikacji i programisty; administrator zarządza całym procesem.

W kontekście zarządzania wyróżniamy też:
* **Procesy jądra (kernel processes)** – w wyniku polecenia `ps aux` ich nazwy wyświetlają się w nawiasach kwadratowych (`[...]`). Nie da się ich ubić ani zmienić ich priorytetu bez restartu systemu.
* **Procesy użytkownika / systemowe** – standardowe procesy real-time i użytkownika.

---

## 1. Zarządzanie procesami w sesji powłoki (Job Control)

Gdy uruchamiasz zadanie i wiesz, że zajmie ono dłuższą chwilę, możesz kontrolować jego bieg za pomocą klawiszy i poleceń:

* **Uruchomienie w tle:** Dodanie `&` na końcu polecenia (np. `command &`) uruchamia proces bezpośrednio w tle. Kiedyś do utrzymania procesu w tle po zamknięciu powłoki stosowano `nohup`, obecnie procesy w tle są odizolowane przez sesję.
* **Przeniesienie na pierwszy plan:** Komenda `fg` (foreground) przywraca zadanie z tła na pierwszy plan. Do `fg` i `bg` można przekazać numer porządkowy zadania (np. `fg 1`).
* **Wstrzymanie procesu:** Skrót **`Ctrl + Z`** pauzuje bieżący proces i przenosi go do tła w stanie wstrzymanym.
* **Wznowienie w tle:** Komenda `bg` (background) wznawia działanie wstrzymanego procesu w tle.
* **Przerwanie procesu:** **`Ctrl + C`** wysyła sygnał przerwania (SIGINT) i usuwa proces z pamięci.
* **Sygnał EOF:** **`Ctrl + D`** wysyła sygnał końca pliku / strumienia (EOF).
* **Listowanie zadań:** Komenda **`jobs`** wyświetla listę wszystkich zadań powiązanych z bieżącą sesją shella.

---

## 2. Monitorowanie i listowanie procesów (`ps` oraz `pgrep`)

Podstawowym narzędziem do podglądu procesów jest polecenie **`ps`**:
* **Bez parametrów:** Pokazuje procesy powiązane z bieżącą sesją terminala aktualnego użytkownika.
* **`aux`:** Wyświetla skrócone podsumowanie wszystkich aktywnych procesów w systemie (wszystkich użytkowników).
* **`-ef`:** Pokazuje pełne informacje o procesach wraz z pełną ścieżką i argumentami wywołania polecenia.
* **`f` (np. `-ef` lub `fax`):** Wyświetla hierarchię procesów (drzewo procesów).
* **`-o`:** Pozwala precyzyjnie wyspecyfikować kolumny, jakie mają zostać wyświetlone.

### Wyszukiwanie PID-ów (`pgrep`)
Jeśli szukasz wyłącznie identyfikatorów procesów (PID) na podstawie nazwy:
* `PGREP NAZWA` – zwraca czystą listę PID-ów (każdy w nowej linii).
* `-l` – wyświetla nazwę procesu obok PID-u.
* `-u USER` – ogranicza wyniki do procesów należącego do wskazanego użytkownika.
* `-v` – odwraca wynik wyszukiwania (pokazuje wszystko, co **nie** spełnia warunku).

---

## 3. Priorytety procesów (`nice` i `renice`)

Każdy proces startuje domyślnie z priorytetem (niceness) równym **0**. 
* Skala priorytetów wynosi od **-20 do 19** (gdzie **-20** to najwyższy możliwy priorytet – proces najważniejszy dla systemu, a **19** to najniższy).
* Wartość niceness jest widoczna w kolumnie **NI** np. w programie `top`.

**Zasady modyfikacji:**
* **`nice -n WARTOSC komenda`** – uruchamia nowy proces z ustaloną wartością niceness.
* **`renice WARTOSC -p PID`** – zmienia priorytet działającego już procesu w czasie rzeczywistym.
* **Uprawnienia:** Zwykły użytkownik może jedynie **zwiększać** wartość niceness (czyli obniżać priorytet procesów). Tylko `root` może nadawać procesom wyższe priorytety (wartości ujemne od -20).

---

## 4. Sygnały i zarządzanie ich wysyłaniem

Sygnały to powiadomienia wysyłane do procesów w celu wymuszenia określonego zachowania. Można je wysyłać za pomocą komend `kill`, `pkill` lub `killall`.

### Najważniejsze sygnały:
* **`SIGTERM (15)`** – domyślny sygnał wysyłany przez `kill`. Grzecznie prosi proces o bezpieczne zakończenie pracy i zwolnienie zasobów.
* **`SIGKILL (9)`** – natychmiastowo ubija proces na poziomie jądra systemu. Nie pozwala na zapisanie stanu (leczenie "poletka"), dlatego należy stosować go ostatecznie.
* **`SIGHUP (1)`** – zawiesza i restartuje proces, co w przypadku większości demonów skutkuje **ponownym odczytaniem plików konfiguracyjnych**.
* **`SIGSTOP (19)`** – zatrzymuje (pauzuje) proces w sposób uniemożliwiający zignorowanie sygnału.
* **`SIGCONT (18)`** – wznawia działanie procesu zatrzymanego przez `SIGSTOP` lub `Ctrl+Z`.
* **`SIGTSTP (20)`** – sygnał wstrzymania generowany terminalowo (odpowiednik `Ctrl+Z`).

### Narzędzia do wysyłania sygnałów:
* **`kill -l`** – wyświetla pełną listę dostępnych sygnałów w systemie.
* **`kill PID`** – wysyła domyślny sygnał `SIGTERM` do wskazanego procesu.
* **`kill -9 PID`** – wysyła sygnał `SIGKILL` (wymuszenie zamknięcia).
* **`pkill NAZWA`** – wysyła sygnał (domyślnie `SIGTERM`) do procesów dopasowanych po nazwie.
* **`killall NAZWA`** – wysyła sygnał do wszystkich procesów o dokładnie tej samej nazwie.


###########


# ROZDZIAL 10 - Virtual machines

**KVM** ​jest natywne w kernelu (kernel modules musi miec ​ **KVM** ​). Jednakze by z tego skorzystac
trzeba miec kawalki ​ **QEMU** ​or demona o nazwie ​ **LIBVIRTD** ​. Konfig tego demona jest w
**/ETC/LIBVIRT/LIBVIRTD.CONF** ​.
By uzywac ​ **KVM** ​potrzeba ​ **64bit** ​ systemu i akceleracji sprzetowej dla virtualizacji. Daj ​ **ARCH** ​ lub
**UNAME -i** ​ by sie dowiedziec czy spelniasz kryteria. akceleracje sprzetowa sprawdzamy ​ **cat
/proc/cpuinfo** ​ -> powinno byc ​ **VMX** ​gdzies w outpucie dla intela i ​ **SVM** ​dla AMD.
Do tego trzeba miec miejsce na dysku - ​ **/VAR/LIB/LIBVIRT/IMAGES** ​ - tam sie skladuja
domyslnie obrazy VMek
By zainstalowac co trzeba najlepiej - ​ **yum groupinstall “Virtualization Host”**
Byc moze trzeba zrobic ​ **SYSTEMCTL ENABLE LIBVIRTD + START**
Generalnie najlepiej robic wszystko z GUI - ​ **Virtual Machine Manager** ​ - uruchamia sie z palca -
**VIRT-MANAGER &**
Do instalacji VMki z command-line sie robi ​ **VIRT-INSTALL**
Do zarzadzania VMkami ma sie tez polecenie z command line - ​ **VIRSH** ​.
* ​ **list** ​Shows all VMs that are currently active
* ​ **list --all** ​ Shows all VMs, including machines that are not currently active
* ​ **help** ​Gives a list of all parameters that can be used with the virsh command
*​ **shutdown <vmname>** ​ Shuts down the VM properly
* ​ **destroy <vmname>** ​ Halts a VM, similar to pulling the power plug on a real computer
* ​ **edit <vmname>** ​ Opens a vi interface that allows you to edit the XML configuration file
belonging to a specific VM
* ​ **console <vmname>** ​ Connects to a VM directly from the console of a KVM host server
* ​ **start <vmname>** ​ Starts a VM
*​ **reboot <vmname>** ​ Reboots a VM

################################


# ROZDZIAŁ 11 - MANAGING SOFTWARE

Współczesne systemy RHEL używają `dnf` (który jest następcą i logicznym rozwinięciem `yum`). Warto pamiętać, że chociaż komendy `yum` są wciąż dostępne (jako aliasy), to pod spodem operuje silnik `dnf`.

## 1. Repozytoria i konfiguracja

Informacje o repozytoriach znajdują się w plikach z rozszerzeniem `.repo` w katalogu `/etc/yum.repos.d/`.

### Struktura pliku repozytorium:
```ini
[label]
name=Nazwa Repozytorium
baseurl=http://serwer/repo/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-nazwa
sslverify=1

```

* **GPG Check:** Kluczowy element bezpieczeństwa. Jeśli `gpgcheck=1`, system weryfikuje podpis cyfrowy pakietów. Klucze importuje się do `/etc/pki/rpm-gpg/`.
* **SSL:** W przypadku problemów z certyfikatami na egzaminacyjnym serwerze (np. samopodpisane certyfikaty), można dodać `sslverify=0` w sekcji repozytorium (choć w środowisku produkcyjnym jest to niezalecane).

### Uwaga: Subscription Manager i lokalne repozytoria

Podczas pracy z lokalnymi repozytoriami (np. na egzaminie), `subscription-manager` może blokować dostęp do darmowych źródeł pakietów. Aby go wyłączyć:

1. Edytuj `/etc/dnf/plugins/subscription-manager.conf`.
2. Zmień `enabled=1` na `enabled=0`.

## 2. Zarządzanie pakietami (DNF / YUM)

DNF automatycznie rozwiązuje zależności (dependencies) pakietów.

* `dnf repolist`: Pokazuje listę aktywnych repozytoriów.
* `dnf search nazwa`: Szuka pakietu po nazwie.
* `dnf provides "*/nazwa_pliku"`: Bardzo przydatna komenda, gdy nie znasz nazwy pakietu, ale wiesz, jaki plik jest Ci potrzebny.
* `dnf install nazwa` (`-y` automatycznie akceptuje).
* `dnf remove nazwa`: Usuwa pakiet.
* `dnf history`: Pokazuje historię operacji. `dnf history undo ID` cofa wybraną akcję.
* `dnf clean all`: Czyści pamięć podręczną (metadata) – przydatne, gdy repozytorium było edytowane.
* `dnf group list`: Listuje grupy pakietów (np. "Development Tools").

## 3. Praca z pakietami RPM (niski poziom)

Czasami instalujemy pojedyncze pliki `.rpm` pobrane z sieci.

* `dnf install ./pakiet.rpm`: Najlepsza metoda (rozwiązuje zależności).
* `rpm -ivh pakiet.rpm`: Instalacja "na twardo" bez rozwiązywania zależności (niezalecane, jeśli nie musisz).

### Zapytania do bazy RPM (`rpm -q`)

Baza `rpm` jest niezależna od metadanych `dnf`.

* `rpm -qa`: Listuje wszystkie zainstalowane pakiety.
* `rpm -qi nazwa`: Informacje o pakiecie.
* `rpm -ql nazwa`: Lista wszystkich plików w pakiecie.
* `rpm -qc nazwa`: Lista plików konfiguracyjnych pakietu.
* `rpm -qf /ścieżka/do/pliku`: Sprawdza, jaki pakiet dostarczył dany plik.
* `rpm -V nazwa`: **Weryfikacja pakietu**. Sprawdza, czy pliki w systemie nie różnią się od oryginału (np. czy nie zostały zmodyfikowane lub uszkodzone).

## 4. Dodatkowe narzędzia (`yum-utils`)

`yum-utils` to zestaw narzędzi ułatwiający administrację:

* `repoquery -l nazwa`: Pozwala przeglądać zawartość pakietów w repozytorium **bez ich pobierania**.
* `yumdownloader --resolve nazwa`: Pobiera pakiet wraz z jego zależnościami do bieżącego katalogu.

---

**Ważne notatki na egzamin:**

1. Zawsze sprawdzaj `gpgkey` – jeśli repo jest udostępnione lokalnie, często musisz zaimportować klucz komendą `rpm --import /path/to/key`.
2. Na egzaminie GUI prawdopodobnie będzie niedostępne – opanuj do perfekcji komendy `dnf` oraz `rpm`.
3. Podczas rozwiązywania zadań z repozytoriami, zawsze weryfikuj czy repo jest `enabled=1` i czy ścieżka w `baseurl` jest poprawna.


# CHAPTER 12 - Managing recurring tasks

Automatyzacja i planowanie zadań w systemie Linux opiera się głównie na demonach `crond` oraz `atd`, a w nowoczesnych systemach również na komponentach `systemd` (timers).

## 1. CRON – Zadania cykliczne

Demon `crond` uruchamia się razem z systemem i co minutę sprawdza, czy nadszedł czas na wykonanie jakiegoś zadania. Status usługi sprawdzamy poleceniem:
```bash
systemctl status crond -l

```

### Składnia tabeli crona (Crontab)

Każdy wpis w tabeli składa się z 5 pól określających czas oraz pola z komendą do wykonania:

```text
* * * * * /sciezka/do/komendy
│ │ │ │ │
│ │ │ │ └─ Dzień tygodnia (0 - 7, gdzie 0 i 7 to Niedziela)
│ │ │ └─── Miesiąc (1 - 12)
│ │ └───── Dzień miesiąca (1 - 31)
│ └─────── Godzina (0 - 23)
└───────── Minuta (0 - 59)

```

**Przykłady harmonogramów:**

* `0 11 * * 1-5` – Codziennie od poniedziałku do piątku o godzinie 11:00.
* `0 */2 * * *` – Co dwie godziny, dokładnie na początku godziny.
* `30 2 * * 0` – W każdą niedzielę o godzinie 02:30.
* **Ważna zasada:** Każdy plik konfiguracyjny crontaba **musi kończyć się pustą nową linią** (blank line), w przeciwnym razie ostatnie zadanie może nie zostać wykonane przez demona!

### Gdzie przechowywane są konfiguracje crona?

1. **Główny plik systemowy:** `/etc/crontab` – zawiera dodatkowo kolumnę określającą użytkownika, z którego uprawnieniami ma się wykonać zadanie. Nie edytuje się go bezpośrednio do zadań użytkowników.
2. **Katalog systemowy:** `/etc/cron.d/` – pliki wrzucane tutaj muszą mieć odpowiednie uprawnienia i wewnątrz definiować użytkownika (analogicznie do `/etc/crontab`). Pliki te **muszą być wykonywalne** (`executable`).
3. **Katalogi uproszczone (Convenience directories):** `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly`, `/etc/cron.monthly` – wrzucone tam skrypty wykonują się automatycznie w ustalonych przez system odstępach czasu (często zarządzane przez anacron).
4. **Crontaby użytkowników:** Każdy użytkownik może zarządzać własnymi zadaniami za pomocą polecenia:
* `crontab -e` – edycja własnego harmonogramu.
* `crontab -l` – wylistowanie zadań.
* `crontab -r` – usunięcie całego crontaba.
* Pliki te lądują w katalogu `/var/spool/cron/`, ale **nigdy nie edytuje się ich ręcznie z poziomu tego katalogu** – zawsze należy używać polecenia `crontab -e`.
* **Root** może zarządzać crontabami innych użytkowników: `crontab -e -u nazwa_uzytkownika`.



### Kontrola dostępu (Security)

Dostęp do polecenia `crontab` dla zwykłych użytkowników regulują pliki:

* `/etc/cron.allow` – jeśli istnieje, tylko użytkownicy w nim wymienieni mogą używać crona.
* `/etc/cron.deny` – jeśli istnieje, użytkownicy w nim wymienieni mają zablokowany dostęp.

---

## 2. ANACRON – Zadania dla systemów niestale włączonych

Tradycyjny `cron` zakłada, że komputer działa 24/7. Jeśli serwer był wyłączony w godzinach, w których miało się wykonać zadanie z crona (np. o 3:00 w nocy), zadanie przepada. Do akcji wkracza **Anacron**.

* **Dla kogo:** Głównie dla stacji roboczych lub serwerów wyłączanych okresowo.
* **Jak działa:** Anacron nie sprawdza dokładnej godziny. Sprawdza, czy od ostatniego uruchomienia danego zadania minęła określona liczba dni (np. 1 dzień dla zadań dziennych). Jeśli w wyznaczonym czasie system był wyłączony, anacron odpali zaległe zadanie niedługo po ponownym uruchomieniu systemu.
* **Konfiguracja:** Znajduje się w pliku `/etc/anacrontab`.
* **Ograniczenie:** Anacron uruchamia się standardowo z uprawnieniami **root** i nie obsługuje precyzyjnego harmonogramu minutowego/godzinowego (działa na interwałach dniowych).

---

## 3. Systemd Timers (Nowoczesna alternatywa dla Crona)

W nowoczesnych dystrybucjach RHEL obok crona funkcjonują **Systemd Timers**, które ściśle współpracują z menedżerem systemd.

* **Zalety:** Posiadają wbudowane wsparcie dla logowania (journald), mogą powiązać uruchomienie zadania z wystąpieniem konkretnego zdarzenia (np. start sieci, upływ czasu od rozruchu systemu) oraz oferują dokładniejszą kontrolę nad zależnościami.
* **Jak to działa:** Składają się z dwóch jednostek o tej samej nazwie:
1. Plik `.service` – definiuje, *co* ma się wykonać (jak zwykła usługa systemd).
2. Plik `.timer` – definiuje, *kiedy* ma się wykonać (harmonogram).


* **Przydatne polecenia:**
* `systemctl list-timers` – Wyświetla listę wszystkich aktywnych timerów w systemie wraz z informacją, kiedy uruchomятся ponownie (odpowiednik dawnego *cronnext*).



---

## 4. AT – Zadania jednorazowe

Jeśli potrzebujesz wykonać komendę **jednorazowo** w określonym czasie w przyszłości, używasz demona `atd` oraz polecenia `at`.

* **Przykład użycia:**
```bash
at 14:00
at> tar -czf /backup/home.tar.gz /home
at> <Ctrl + D>

```


Wpisanie `at 14:00` (lub np. `at noon`, `at tomorrow`, `at now + 1 hour`) otwiera interaktywną powłokę, w której wpisujesz komendy. Wpisywanie kończysz kombinacją klawiszy **`Ctrl + D`**.
* **Zarządzanie kolejką zadań:**
* `atq` (At Queue) – Wyświetla listę zaplanowanych zadań jednorazowych wraz z ich numerami w kolejności.
* `atrm NUMER` – Usuwa wskazane zadanie z kolejki przed jego wykonaniem.


* **Bezpieczeństwo:** Podobnie jak cron, `at` korzysta z plików `/etc/at.allow` oraz `/etc/at.deny` do kontroli uprawnień użytkowników.





# ROZDZIAŁ 13 - CONFIGURING LOGGING

W RHEL 8/9/10 logowanie oparte jest na współpracy dwóch głównych systemów: **journald** (zbiera dane binarne z całego systemu) oraz **rsyslog** (przetwarza i zapisuje logi do plików tekstowych w `/var/log`).

## 1. System logowania w RHEL

* **Journald:** Serwis systemowy, który gromadzi logi z kernela, procesów startowych (boot) i usług. Domyślnie logi są w pamięci RAM (`/run/log/journal`) i **znikają po restarcie**. Można włączyć trwałość (persystencję), tworząc katalog `/var/log/journal`.
* **Rsyslog:** Klasyczny demon logowania. Pobiera logi z journald lub bezpośrednio od usług i zapisuje je w plikach tekstowych w `/var/log`.
* **Logrotate:** Narzędzie automatyzujące rotację logów, aby nie zapełniły dysku (konfiguracja: `/etc/logrotate.conf`).

## 2. Ważne pliki w `/var/log`

* `/var/log/messages`: Główne, ogólne logi systemowe. Tutaj trafia większość zdarzeń.
* `/var/log/secure`: Logi uwierzytelniania (logowanie SSH, sudo, zmiany haseł). Kluczowe przy dochodzeniach włamaniowych.
* `/var/log/dmesg`: Logi startowe kernela (widoczne również komendą `dmesg`).
* `/var/log/boot.log`: Informacje o starcie usług systemowych.
* `/var/log/maillog`: Logi serwera pocztowego.
* `/var/log/audit/audit.log`: Logi audytu (m.in. SELinux).
* `/var/log/cron`: Logi zadań zaplanowanych (`crond`).
* `/var/log/samba/` oraz `/var/log/httpd/`: Przykłady usług, które często piszą własne logi bezpośrednio, z pominięciem rsysloga.

## 3. Praca z dziennikiem (Journalctl)

`journalctl` to narzędzie do przeglądania binarnych logów z journald.

* `journalctl`: Pokazuje cały dziennik (od najstarszych wpisów).
* `journalctl -f`: Tryb "follow" (na żywo, jak `tail -f`).
* `journalctl -n 20`: Pokazuje 20 ostatnich linii.
* `journalctl -p err`: Filtruje logi według priorytetu (w tym przypadku błędy i krytyczne).
* `journalctl --since "2026-08-15 10:00" --until "now"`: Filtrowanie po czasie.
* `journalctl -u nazwa_uslugi`: Logi dla konkretnej usługi systemd.
* `journalctl -k`: Tylko komunikaty kernela.
* `journalctl --no-pager`: Wypisuje wszystko bez użycia `less` (przydatne do skryptów).

## 4. Konfiguracja Rsyslog (`/etc/rsyslog.conf`)

Plik `/etc/rsyslog.conf` oraz skrypty w `/etc/rsyslog.d/` definiują reguły logowania. Reguła składa się z dwóch części: **FACILITY.PRIORITY** oraz **DESTINATION**.

### Facilities (Kategorie):
`auth/authpriv` (logowanie), `cron` (zadania crona), `kern` (kernel), `mail` (poczta), `user` (przestrzeń użytkownika), `local0-7` (lokalne/własne).

### Priorities (Ważność - od najniższego do najwyższego):
`debug`, `info`, `notice`, `warning`, `err`, `crit`, `alert`, `emerg`.
*Zapis `mail.info` oznacza: loguj wszystko od `info` w górę dla kategorii `mail`.*

### Przykładowa reguła:
`*.info;mail.none;authpriv.none;cron.none  /var/log/messages`
* Oznacza: Loguj wszystko powyżej `info`, z pominięciem maila, autoryzacji i crona, do pliku `/var/log/messages`. 
* **Ważne:** Jeśli przed ścieżką dodasz myślnik (np. `- /var/log/messages`), rsyslog będzie buforował zapisy, co poprawia wydajność, ale zwiększa ryzyko utraty najnowszych linii w razie awarii zasilania.

## 5. Ręczne pisanie do logów (`logger`)

Możesz wysłać własną wiadomość do logów systemowych:
```bash
logger -p user.err "To jest komunikat o błędzie"

```

Wpis trafi bezpośrednio do `/var/log/messages` (lub innego miejsca zdefiniowanego dla kategorii `user.err`).

## 6. Logi zadań (Cron / At)

* Logi zadań `cron` trafiają zazwyczaj do `/var/log/cron`.
* Logi zadań `at` są zazwyczaj powiązane z logami systemowymi (`/var/log/messages`) lub powiadomieniami wysyłanymi przez `mail`.

# ROZDZIAŁ 13 - CONFIGURING LOGGING

W RHEL 8/9/10 logowanie oparte jest na współpracy dwóch głównych systemów: **journald** (zbiera dane binarne z całego systemu) oraz **rsyslog** (przetwarza i zapisuje logi do plików tekstowych w `/var/log`).

## 1. System logowania w RHEL

* **Journald:** Serwis systemowy, który gromadzi logi z kernela, procesów startowych (boot) i usług. Domyślnie logi są w pamięci RAM (`/run/log/journal`) i **znikają po restarcie**. Można włączyć trwałość (persystencję), tworząc katalog `/var/log/journal`.
* **Rsyslog:** Klasyczny demon logowania. Pobiera logi z journald lub bezpośrednio od usług i zapisuje je w plikach tekstowych w `/var/log`.
* **Logrotate:** Narzędzie automatyzujące rotację logów, aby nie zapełniły dysku (konfiguracja: `/etc/logrotate.conf`).

## 2. Ważne pliki w `/var/log`

* `/var/log/messages`: Główne, ogólne logi systemowe. Tutaj trafia większość zdarzeń.
* `/var/log/secure`: Logi uwierzytelniania (logowanie SSH, sudo, zmiany haseł). Kluczowe przy dochodzeniach włamaniowych.
* `/var/log/dmesg`: Logi startowe kernela (widoczne również komendą `dmesg`).
* `/var/log/boot.log`: Informacje o starcie usług systemowych.
* `/var/log/maillog`: Logi serwera pocztowego.
* `/var/log/audit/audit.log`: Logi audytu (m.in. SELinux).
* `/var/log/cron`: Logi zadań zaplanowanych (`crond`).
* `/var/log/samba/` oraz `/var/log/httpd/`: Przykłady usług, które często piszą własne logi bezpośrednio, z pominięciem rsysloga.

## 3. Praca z dziennikiem (Journalctl)

`journalctl` to narzędzie do przeglądania binarnych logów z journald.

* `journalctl`: Pokazuje cały dziennik (od najstarszych wpisów).
* `journalctl -f`: Tryb "follow" (na żywo, jak `tail -f`).
* `journalctl -n 20`: Pokazuje 20 ostatnich linii.
* `journalctl -p err`: Filtruje logi według priorytetu (w tym przypadku błędy i krytyczne).
* `journalctl --since "2026-08-15 10:00" --until "now"`: Filtrowanie po czasie.
* `journalctl -u nazwa_uslugi`: Logi dla konkretnej usługi systemd.
* `journalctl -k`: Tylko komunikaty kernela.
* `journalctl --no-pager`: Wypisuje wszystko bez użycia `less` (przydatne do skryptów).

## 4. Konfiguracja Rsyslog (`/etc/rsyslog.conf`)

Plik `/etc/rsyslog.conf` oraz skrypty w `/etc/rsyslog.d/` definiują reguły logowania. Reguła składa się z dwóch części: **FACILITY.PRIORITY** oraz **DESTINATION**.

### Facilities (Kategorie):
`auth/authpriv` (logowanie), `cron` (zadania crona), `kern` (kernel), `mail` (poczta), `user` (przestrzeń użytkownika), `local0-7` (lokalne/własne).

### Priorities (Ważność - od najniższego do najwyższego):
`debug`, `info`, `notice`, `warning`, `err`, `crit`, `alert`, `emerg`.
*Zapis `mail.info` oznacza: loguj wszystko od `info` w górę dla kategorii `mail`.*

### Przykładowa reguła:
`*.info;mail.none;authpriv.none;cron.none  /var/log/messages`
* Oznacza: Loguj wszystko powyżej `info`, z pominięciem maila, autoryzacji i crona, do pliku `/var/log/messages`. 
* **Ważne:** Jeśli przed ścieżką dodasz myślnik (np. `- /var/log/messages`), rsyslog będzie buforował zapisy, co poprawia wydajność, ale zwiększa ryzyko utraty najnowszych linii w razie awarii zasilania.

## 5. Ręczne pisanie do logów (`logger`)

Możesz wysłać własną wiadomość do logów systemowych:
```bash
logger -p user.err "To jest komunikat o błędzie"

```

Wpis trafi bezpośrednio do `/var/log/messages` (lub innego miejsca zdefiniowanego dla kategorii `user.err`).

## 6. Logi zadań (Cron / At)

* Logi zadań `cron` trafiają zazwyczaj do `/var/log/cron`.
* Logi zadań `at` są zazwyczaj powiązane z logami systemowymi (`/var/log/messages`) lub powiadomieniami wysyłanymi przez `mail`.




# ROZDZIAL 14 - MANAGING PARTITIONS

Przy ​ **MBRze** ​ilosc ​ **PRIMARY PARTITIONS** ​ to​ **MAX 4** ​. Jednakze by to obejsc mozna stworzyc
wiecej ​ **EXTENDED PARTITIONS** ​ w ramach jednej ​ **PRIMARY PARTITION** ​. Na ​ **EXTENDED
PARTITION** ​ mogloby byc ​ **MAX 15 partycji** ​. ​ **MAX rozmiar partycji w MBRze to 2 TB.**
Oczywiscie powyzsze w swietle dzisiejszych dyskow to wybitnie za malo. Wiec powstalo ​ **GUID
PARTITION TABLE (GPT)** ​. Na nowych kompach co uzywaja ​ **UEFI** ​to jest jedyny sposob by sie
dostac do dyskow i je poustawiac. Cechy:
* max rozmiar to​ **5 ZiB (zebibytes)** ​- nie ma limitu na 2 TiB oczywsicie
* do ​ **128** ​ partycji
* nie ma sensu dzielic tego na primary, extended i logical partitions (bo nie ma jak w MBRze
limitu 64 KB na info o partycjach)
* uzywa sie ​ **128-bit GUIDa** ​ jako identyfikatora partyci
* automatycznie na koncu dysku robi sie kopie zapasowa tabeli partycji
Do operowania na partycjach czy cos potrzebujemy jak zawsze nazwy dysku (leci /sda, /sdb, a
jak sie skonczy to /sdaa):
*​ **/dev/sda** ​ - A hard disk that uses the SCSI driver. Used for SCSI and SATA disk devices.
Common on physical servers but also in VMware virtual machines.
* ​ **/dev/hda** ​ - The (legacy) IDE disk device type. You will seldom see this device type on modern
computers.
* ​ **/dev/vda** ​ - A disk in a ​ **KVM** ​virtual machine that uses the ​ **virtio** ​disk driver. This is the
common disk device type for KVM virtual machines.
*​ **/dev/xvda** ​ - A disk in a Xen virtual machine that uses the Xen virtual disk driver. You see this
when installing RHEL as a virtual machine in Xen. RHEL 7 cannot be used as a Xen hypervisor,
but you might see RHEL 7 virtual machines on top of the Xen hypervisor using these disk types
Do tworzenia partycji ​ **MBR** ​uzywamy ​ **FDISK** ​. Jest to w pyte stare narzedzie. Generalnie dobrze
poczytac to: ​ **https://www.binarytides.com/linux-command-check-disk-partitions/**
Samo tworzenie partycji trzeba zaczac od tego, ze sie wie co sie ma obecnie. Wali sie​ **FDISK -L**
i on pokazuje podzialy (oczywiscie na ROOT). Generalnie na minimal install w Centosie w
VBoxie poszedl​ **/dev/sda** ​.
* Wiec by stworzyc cos nowego daje sie​ **FDISK LINK_DO_URZADZENIA** ​. Pojawia sie konsola
**FDISK** ​.
* By stworzyc cos nowego klepiemy​ **N** ​ i wybieramy typ partycji (domyslnie primary).
* Potem lecimy rozmiar (start blokow), a potem koncowke - tutaj najsensowniej nie wpisywac
calosci tylko po prostu dac ​ **+ROZMIAR(M|GB)** ​.
* Poki co zmiany sa tylko w wirtualu. By to fizycznie zapisac do ​ **MBR** ​trzeba dac komende ​ **W** ​.
* Jak poleci blad, ze kernel uzywa starej tablicy partycji (porownaj ​ **fdisk -l LINK_DO_DYSKU** ​ z


**cat /proc/partitions** ​) to trzeba odswiezyc tablice kernela.​ **PARTPROBE LINK_DO_DYSKU**
Utworzylismy partycje ​ **PRIMARY** ​. Jednakze mamy jeszcze jeden slot na partycje to utworzymy
sobie ​ **EXTENDED** ​tylko po to by na niej stworzyc ​ **LOGICAL** ​. Na ​ **EXTENDED** ​normalnie nie
mozna tworzyc filesystemow - ona sluzy tylko jako opakowanie na ​ **LOGICAL** ​.
Jezeli dysk ma wiecej niz ​ **2 TiB** ​ lub juz byl konfigurowany za pomoca ​ **GPT** ​no to generalnie
trzeba uzywac polecenia ​ **GDISK** ​. Generalnie ​ **FDISK** ​ma jakies tam wsparcie dla ​ **GPT** ​, ale to jest
niestabilne wiec lepiej tego nie uzywac. ​ **NIE UZYWAJ GDISK JEZELI NA DYSKU SA JUZ
PARTYCJE FDISK!!!** ​ Generalnie u mnie ​ **GDISK** ​nie byl zainstalowany w minimal distribution.
Do tego w sumie uruchamia sie i uzywa jak ​ **FDISK** ​.
Utworzenie partycji jeszcze niczego z nia nie robi. By cos sie dalo na niej ogarnac trzeba
stworzyc na niej filesystem. To, ze wybieralismy 'Linux Filesystem' przy tworzeniu partycji to
jeszcze nic nie znaczy. Do wyboru mamy:
* ​ **XFS** ​ --- The default file system in RHEL 7.
* ​ **Ext4** ​ --- The default file system in previous versions of RHEL. Still available and supported in
RHEL 7.
* ​ **Ext3** ​ --- The previous version of Ext4. On RHEL 7, there is no real need to use Ext3
anymore.
* ​ **Ext2** ​ --- A very basic file system that was developed in the early 1990s. There is no need to
use this file system on RHEL 7 anymore.
* ​ **BtrFS** ​ --- A relatively new file system that was not yet supported in RHEL 7.0 but will be
included in later updates.
* ​ **NTFS** ​ --- Not supported on RHEL 7.
* ​ **VFAT** ​ --- A file system that offers compatibility with Windows and Mac, it is the functional
equivalent of the FAT32 file system. Useful to use on USB thumb drives that are used to
exchange data with other computers but not on a server’s hard disks.
Komenda co formatuje partycje na konkretny typ to ​ **MKFS -T TYP_PARTYCJI** ​ (​domyslnie jest
**ext2** ​wiec trzeba ustawic typ​). Sa tez dedykowane narzedzia (np. ​ **mkfs.ext4** ​). Po tym jak sie
stworzy filesystem mozna (w przypadku ​ **ext2-4** ​) zmieniac ustawienia tego filesystemu za
pomoca komendy ​ **TUNE2FS** ​.
Na poczatek​ **TUNE2FS -L ADRES_PARTYCJI** ​ pokaze co mozna z tym zrobic. Do tego:
* ​ **TUNE2FS -o ATRYBUTY,ATRYBUT** ​ pozwala ustawic atrybuty, ktore normalnie ustawia sie w
**/ETC/FSTAB** ​. By cos wylaczyc trzeba dac ^ przed atrybutem
* ​ **TUNE2FS -O FEATURE** ​ pozwala ustawic feature filesytemu. Nic wiecej na ten temat koles nie
pisze (wylacza sie tak samo jak powyzsze)
* ​ **TUNE2FS -L NAZWA** ​ pozwala ustawic labelke dla filesystemu (alternatywa to ​ **E2LABEL** ​)
Dla ​ **XFS** ​narzedzie to ​ **XFS_ADMIN** ​.


# ROZDZIAŁ 15 - LOGICAL VOLUMES (LVM)

LVM (Logical Volume Manager) wprowadza warstwę abstrakcji między fizycznymi dyskami a systemem plików.

## Architektura LVM
1. **Physical Volumes (PV):** Fizyczne partycje lub całe dyski (np. `/dev/sdb1`), przygotowane do pracy z LVM.
2. **Volume Group (VG):** Pula pamięci utworzona z jednego lub wielu PV. To "kontener" na przestrzeń dyskową.
3. **Logical Volumes (LV):** "Wirtualne partycje" wycinane z VG. Na nich tworzymy systemy plików (XFS, Ext4).

### Dlaczego LVM?
* **Dynamiczna zmiana rozmiaru:** Możliwość powiększania LV w locie.
* **Elastyczność:** Możesz łatwo dodać nowy dysk do systemu, dodać go do VG i powiększyć dowolny LV bez utraty danych.
* **Snapshots:** Możliwość tworzenia punktów przywracania stanu danych.

---

## 1. Tworzenie LVM – krok po kroku

### Krok 1: Inicjalizacja Physical Volume (PV)
Po utworzeniu partycji (np. `fdisk`), zamień ją w PV:
```bash
pvcreate /dev/sdb2

```

* Weryfikacja: `pvs`, `pvdisplay`, `lsblk`.

### Krok 2: Tworzenie Volume Group (VG)

Tworzysz grupę z przypisanych jej PV:

```bash
vgcreate nazwa_vg /dev/sdb2

```

* Weryfikacja: `vgs`, `vgdisplay`.

### Krok 3: Tworzenie Logical Volume (LV)

Wycinasz "partycję" z dostępnej grupy:

```bash
# Tworzenie LV o rozmiarze 10GB z nazwą "lv_data" w grupie "nazwa_vg"
lvcreate -n lv_data -L 10G nazwa_vg

# Tworzenie LV wykorzystującego 50% dostępnej przestrzeni w grupie
lvcreate -n lv_data -l 50%FREE nazwa_vg

```

* Dostęp: Ścieżka do wolumenu to `/dev/nazwa_vg/nazwa_lv` lub `/dev/mapper/nazwa_vg-nazwa_lv`.
* Po utworzeniu LV: sformatuj go (`mkfs.xfs ...`) i zamontuj.

---

## 2. Zarządzanie rozmiarami

### Rozszerzanie (W górę)

Możesz rozszerzać VG oraz LV bez utraty danych.

**Rozszerzanie VG:**
Jeśli masz nowy dysk (`/dev/sdc1`), dodajesz go do istniejącej grupy:

```bash
vgextend nazwa_vg /dev/sdc1

```

**Rozszerzanie LV:**
Używamy `lvextend` lub `lvresize`. Flaga `-r` (resizefs) jest kluczowa – automatycznie rozszerza ona system plików na nową przestrzeń.

```bash
# Dodanie 5GB do wolumenu
lvextend -r -L +5G /dev/nazwa_vg/lv_data

# Wykorzystanie całego wolnego miejsca w VG
lvextend -r -l +100%FREE /dev/nazwa_vg/lv_data

```

### Zmniejszanie (W dół)

* **XFS:** **Nie wspiera zmniejszania** systemu plików.
* **Ext4:** Wspiera zmniejszanie, ale **tylko po odmontowaniu** systemu plików.

**Procedura zmniejszania dla Ext4:**

1. `umount /mnt/dane`
2. `lvreduce -r -L -2G /dev/nazwa_vg/lv_data` (Flaga `-r` tutaj również zadba o przeskalowanie filesystemu przed zmniejszeniem wolumenu).

---

## 3. Przegląd parametrów `-l` (relatywnych) w `lvresize`

Różnica w składni ma kluczowe znaczenie:

* `lvresize -r -l 75%VG /dev/vgdata/lvdata`: Ustawia rozmiar LV na dokładnie **75% całkowitego rozmiaru VG**.
* `lvresize -r -l +75%VG /dev/vgdata/lvdata`: Dodaje do istniejącego wolumenu przestrzeń równą **75% całkowitego rozmiaru VG**.
* `lvresize -r -l 75%FREE /dev/vgdata/lvdata`: Ustawia rozmiar LV tak, aby zajmował **75% aktualnie wolnego miejsca** w VG.
* `lvresize -r -l +75%FREE /dev/vgdata/lvdata`: Dodaje do wolumenu **75% dostępnego obecnie wolnego miejsca** w VG.

> **Egzamin RHCSA:** Pamiętaj, że wszystkie zmiany w `/etc/fstab` dla nowych wolumenów logicznych muszą być trwałe, aby przetrwały reboot. Zawsze używaj UUID zamiast nazw ścieżek `/dev/...` w fstab!


# CHAPTER 16 - Basic Kernel Management

Jądro systemu (kernel) zarządza komunikacją między oprogramowaniem a sprzętem (CPU, operacje I/O) za pomocą wątków (**threads**).

## 1. Narzędzia do monitorowania kernela
Do diagnozowania stanu jądra i podglądu komunikatów służą:
* **`dmesg`** – Służy do podglądu **Kernel Ring Buffer**, czyli bufora, w którym jądro zapisuje komunikaty o inicjalizacji sprzętu i zdarzeniach systemowych. Alternatywnie można użyć polecenia `journalctl -k` lub `journalctl --dmesg`.
* **Wirtualny system plików `/proc`** – Zawiera pliki informacyjne o stanie systemu (np. `/proc/meminfo` z informacjami o pamięci RAM).
* **`uname`** – Uniwersalne narzędzie do sprawdzania wersji jądra i architektury systemowej (np. `uname -r` lub `uname -a`). Dodatkowe informacje znajdziesz w `hostnamectl status` oraz w pliku `/etc/redhat-release`.

---

## 2. Inicjalizacja sprzętu i systemd-udevd

Podczas rozruchów i pracy systemu proces wykrywania i konfiguracji podzespołów przebiega następująco:
1. Podczas bootowania jądro wykrywa dostępny sprzęt.
2. Po wykryciu komponentu proces **`systemd-udevd`** ładuje odpowiedni sterownik, udostępniając urządzenie systemowi.
3. Reguły inicjalizacji są odczytywane najpierw z katalogu dostarczanego przez system: **`/usr/lib/udev/rules.d/`** (nie należy modyfikować tych plików).
4. Następnie przetwarzane są reguły niestandardowe z katalogu **`/etc/udev/rules.d/`**.
5. W efekcie wymagane moduły jądra są ładowane automatycznie, a informacje o nich trafiają do wirtualnego systemu plików **`sysfs`** zamontowanego w katalogu **/sys**.

* **Monitorowanie zdarzeń sprzętowych na żywo:** Możesz użyć interaktywnego polecenia `udevadm monitor`, aby obserwować wykrywanie urządzeń w czasie rzeczywistym.

---

## 3. Zarządzanie modułami jądra (Kernel Modules)

Moduły to dynamicznie ładowane fragmenty kodu jądra (np. sterowniki urządzeń).

* **`lsmod`** – Listuje obecnie załadowane w pamięci moduły.
* **`modinfo NAZWA`** – Wyświetla szczegółowe informacje o konkretnym module.
* **`modprobe NAZWA`** – Ładuje wskazany moduł wraz z jego zależnościami.
* **`modprobe -r NAZWA`** – Usuwa (wyładowuje) wskazany moduł.
  > **Ostrzeżenie:** Stare polecenia `insmod` oraz `rmmod` są przestarzałe (*deprecated*) – zawsze korzystaj z `modprobe`.

* **Sprawdzanie powiązania sprzętu ze sterownikami:**
  Polecenie **`lspci -k`** (wymaga zainstalowanego pakietu `pciutils`) pokazuje urządzenia podpięte do magistrali PCI wraz z informacją, jaki moduł jądra (kernel driver) obsługuje dane urządzenie.

### Automatyczne ładowanie modułów przy starcie
Jeśli chcesz wymusić ręczne ładowanie określonych modułów podczas startu systemu, utwórz plik z rozszerzeniem `.conf` w katalogu **`/etc/modules-load.d/`**. Wpisz w nim nazwy modułów w osobnych liniach.

---

## 4. Konfiguracja parametrów modułów i jądra (Kernel Parameters)

Moduły jądra mogą przyjmować dodatkowe parametry podczas ładowania. 

* **Jak dopisać parametry startowe modułu:**
  Należy utworzyć (lub edytować) plik konfiguracyjny w katalogu **`/etc/modprobe.d/`** z rozszerzeniem `.conf` (np. `/etc/modprobe.d/moj_sterownik.conf`). W pliku stosuje się następującą składnię:

```
  options NAZWA_MODULU PARAMETR=WARTOSC
```

*Przykład:* `options my_driver debug=1 buffer_size=4096`

* **Parametry startowe samego jądra (w programie rozruchowym GRUB):**
Jeśli chcesz przekazać parametry bezpośrednio do jądra podczas startu systemu (np. tryb pracy, parametry konsoli), edytuje się wpisy w menu GRUB (lub plik konfiguracyjny w `/etc/default/grub`), dopisując je do linii zaczynającej się od `linux` / `linux16`.

---

## 5. Aktualizacja kernela

Podczas instalacji nowej wersji za pomocą menedżera pakietów (`dnf upgrade kernel` lub `yum install kernel`), system **nie usuwa** poprzedniej wersji. Stare wersje są zachowywane w katalogu `/boot` (domyślnie system przechowuje kilka ostatnich wersji, aby umożliwić bezpieczny powrót do starszego jądra w razie awarii nowej wersji).



# CHAPTER 18 - Managing Boot Procedure

## 1. Systemd i Unity (Units)

W nowoczesnych systemach RHEL procesem rozruchu oraz zarządzaniem usługami rządzi **systemd**. Podstawowym elementem zarządzanym przez systemd jest **Unit** (jednostka). 

### Typy unitów i ich lokalizacja
Jednostkami mogą być serwisy, sockety, punkty montowania i inne zasoby (możesz je wylistować poleceniem `systemctl -T help`). 
* **Domyślne pliki uniwersalne:** `/usr/lib/systemd/system/` (dostarczane z pakietami, nie należy ich edytować bezpośrednio).
* **Nadpisania (Overrides):** `/etc/systemd/system/` (tutaj lądują pliki modyfikowane przez administratora oraz symlinki generowane przy włączeniu usługi).
* **Pliki generowane automatycznie:** `/run/systemd/system/`

### Struktura pliku Unita
Większość plików konfiguracyjnych unitów składa się z sekcji:
* **`[Unit]`** – Opisuje jednostkę oraz definiuje zależności (np. instrukcje `After=` oraz `Before=`, które określają kolejność startu względem innych unitów). Atrybut `Conflicts=` uniemożliwia jednoczesne uruchomienie dwóch sprzecznych jednostek.
* **`[Service]`** – Definiuje, jak uruchomić, zatrzymać i sprawdzić status usługi (np. linie `ExecStart=` oraz `ExecStop=`).
* **`[Install]`** – Zawiera informacje o instalacji (wykorzystywane przy włączaniu/enabled usługi).

Szczegółowe parametry działającego unita wyświetlisz poleceniem:
```bash
systemctl show NAZWA_UNITA

```

---

## 2. Zarządzanie jednostkami za pomocą `systemctl`

Podstawowym narzędziem interakcji z systemd jest polecenie `systemctl`.

### Podstawowa składnia i operacje:

* `systemctl start NAZWA` – Uruchamia jednostkę.
* `systemctl stop NAZWA` – Zatrzymuje jednostkę.
* `systemctl status NAZWA` – Sprawdza stan unita.
* `systemctl enable NAZWA` – Włącza automatyczne uruchamianie przy starcie systemu (tworzy symlink w `/etc/systemd/system/`).
* `systemctl disable NAZWA` – Wyłącza autostart.
* `systemctl mask NAZWA` – Blokuje jednostkę całkowicie, przekierowując ją do `/dev/null` (uniemożliwia przypadkowe lub automatyczne włączenie). Odpowiednik odblokowujący to `systemctl unmask`.

### Stany usługi (w `systemctl status`):

* **Loaded:** Plik unita został poprawnie przetworzony.
* **Active (running):** Działa z co najmniej jednym aktywnym procesem.
* **Active (exited):** Pomyślnie zakończył jednorazowe zadanie konfiguracyjne.
* **Active (waiting):** Działa i oczekuje na zdarzenie.
* **Inactive:** Nie działa.
* **Enabled / Disabled / Static:** Informacje o zachowaniu podczas bootowania (Static oznacza, że unit nie może być włączony samodzielnie, ale może zostać uruchomiony automatycznie przez inny unit).

### Przydatne filtry i diagnostyka:

* `systemctl --type=service` – Pokazuje tylko jednostki typu service.
* `systemctl list-units --type=service --all` – Pokazuje wszystkie serwisy (aktywne i nieaktywne).
* `systemctl --failed --type=service` – Wylistowuje usługi, które zakończyły się błędem.
* `systemctl status -l your.service` – Szczegółowy status wybranej usługi.
* `systemctl list-dependencies NAZWA` – Pokazuje drzewo zależności wybranego unita.

---

## 3. Targety (Odpowiedniki Runleveli)

**Targety** służą do grupowania unitów systemowych i odpowiadają dawnym runlevelom w systemach SysVinit. Same w sobie nie przechowują listy usług – wskazują jedynie zbiór zależności.

### Najważniejsze targety systemowe (i ich odpowiedniki):

* `poweroff.target` – Runlevel 0 (Wyłączenie systemu)
* `rescue.target` – Runlevel 1 (Tryb ratunkowy / single-user)
* `emergency.target` – Runlevel 2 (Awaryjna powłoka root)
* `multi-user.target` – Runlevel 3 (Tryb wieloużytkownikowy, tekstowy, standard dla serwerów)
* `graphical.target` – Runlevel 5 (Tryb graficzny)
* `reboot.target` – Runlevel 6 (Restart systemu)

### Zarządzanie domyślnym targetem:

* Sprawdzenie aktualnego domyślnego targetu:
```bash
systemctl get-default

```


* Zmiana domyślnego targetu (np. na tryb tekstowy):
```bash
systemctl set-default multi-user.target

```


* **Izolacja targetów (`Isolate`):** Niektóre targety mają ustawiony atrybut `AllowIsolate=yes`. Pozwala to na przełączenie systemu w tryb, w którym uruchomiony jest *tylko* dany target i jego bezpośrednie zależności (np. przejście w tryb ratunkowy):
```bash
systemctl isolate rescue.target

```



---

## 4. Zarządzanie programem rozruchowym GRUB2

**GRUB2** to domyślny bootloader w systemach RHEL. Wykorzystuje on obraz **initramfs** (zawierający niezbędne moduły jądra i sterowniki), aby system mógł poprawnie zamontować główny system plików podczas rozruchu.

### Zasady modyfikacji GRUB-a:

1. Pliki konfiguracyjne powłoki GRUB znajdują się w katalogu `/etc/grub.d/`, a główny plik parametrów to **`/etc/default/grub`**. **Nigdy nie edytujemy ręcznie pliku `/boot/grub2/grub.cfg**`, ponieważ jest on generowany automatycznie przez instalator kernela lub skrypty konfiguracyjne!
2. Najważniejszą linijką w `/etc/default/grub` jest **`GRUB_CMDLINE_LINUX`**, gdzie definiuje się parametry przekazywane do jądra (warto często usunąć parametry `rhgb` oraz `quiet`, aby widzieć pełne logi diagnostyczne podczas startu systemu).
3. Po każdej zmianie w `/etc/default/grub` należy wygenerować nową konfigurację poleceniem:
```bash
grub2-mkconfig > /boot/grub2/grub.cfg

```



### Zarządzanie wersjami kernela i domyślnym wyborem:

Aby sprawdzić, jakie wersje jądra są zainstalowane w systemie i nimi zarządzać, stosuje się następujące polecenia:

* `yum list kernel` lub `rpm -qa | grep kernel-[0-9]` – Wylistowanie zainstalowanych kerneli.
* `grubby --info=ALL` – Wyświetla szczegółowe informacje o wszystkich dostępnych kernelach wraz z ich indeksami i ścieżkami.
* Ustawienie domyślnego jądra do rozruchu (za pomocą narzędzia `grubby` lub `grub2-set-default`):
```bash
grubby --set-default-index NUMER
# lub podając pełną ścieżkę do vmlinuz:
grubby --set-default /boot/vmlinuz-...

```





# CHAPTER 19 - Troubleshooting boot problems

Proces uruchamiania systemu Linux (bootowania) składa się z kilku kluczowych faz. Znajomość tej kolejności jest niezbędna do skutecznego diagnozowania i naprawiania awarii.

## 1. Fazy uruchamiania systemu (Boot Phases)

1. **POST (Power-On Self-Test):** Włączenie maszyny, inicjalizacja sprzętu przez firmware UEFI lub klasyczny BIOS.
2. **Wybór urządzenia rozruchowego:** Lokalizacja nośnika z bootloaderem (z poziomu UEFI lub MBR).
3. **Ładowanie programu rozruchowego (Boot Loader):** Uruchomienie programu ładującego, którym w systemach Red Hat jest zazwyczaj **GRUB2**.
4. **Ładowanie jądra (Kernel) i initramfs:** GRUB ładuje jądro systemu wraz z obrazem **initramfs**. Obraz ten zawiera moduły niezbędne do obsługi sprzętu oraz wstępne skrypty rozruchowe.
5. **Uruchomienie /sbin/init:** Po załadowaniu jądra do pamięci uruchamiany jest pierwszy proces (w systemach Red Hat dowiązany do **systemd**) oraz demon `udev` do dalszej inicjalizacji sprzętu.
6. **Przetwarzanie initrd.target:** Systemd wykonuje jednostki z `initrd.target`, co tworzy minimalne środowisko robocze i montuje główny system plików z dysku w katalogu `/sysroot`.
7. **Przełączenie na główny system plików (Root File System):** Następuje przełączenie na system plików znajdujący się na twardym dysku i załadowanie docelowego procesud systemd z dysku.
8. **Uruchomienie domyślnego targetu (Default Target):** Systemd uruchamia jednostki przypisane do domyślnego targetu, inicjalizuje ekran logowania. *Uwaga:* Pojawienie się promptu logowania nie oznacza jeszcze, że wszystkie usługi w tle zostały w pełni uruchomione!

---

## 2. Podsumowanie faz i ich naprawy (Tabela diagnostyczna)

| Faza Bootowania | Konfiguracja | Jak naprawić awarię |
| :--- | :--- | :--- |
| **POST** | Konfiguracja sprzętu (klawisze F2, Esc, F10 itp.) | Wymienić uszkodzony sprzęt. |
| **Wybór urządzenia** | Konfiguracja BIOS/UEFI lub menu startowe | Wymienić sprzęt lub uruchomić system ratunkowy. |
| **Ładowanie bootloadera** | `grub2-install`, edycja `/etc/default/grub` | Edycja w promptcie GRUB, modyfikacja `/etc/default/grub` + `grub2-mkconfig`. |
| **Ładowanie jądra** | Edycja konfiguracji GRUB oraz `/etc/dracut.conf` | Edycja w promptcie GRUB lub `/etc/default/grub` + `grub2-mkconfig`. |
| **Uruchomienie /sbin/init** | Wbudowane w initramfs | Użycie argumentów jądra: `init=` lub `rd.break`. |
| **Przełączenie na rootfs** | Plik `/etc/fstab` | Poprawienie błędów w `/etc/fstab`. |
| **Uruchomienie domyślnego targetu** | `/etc/systemd/system/default.target` | Uruchomienie `rescue.target` jako argumentu jądra. |

---

## 3. Modyfikacja parametrów startowych w GRUB2

Gdy podczas startu systemu pojawia się menu GRUB2, naciśnięcie klawisza **`E`** pozwala na edycję opcji zaznaczonego wpisu (kernel i jego parametry). 

* **Jednorazowa edycja:** 
  W linijce zaczynającej się od `linux16 /vmlinuz-...` (lub `linux`) znajdują się parametry startowe. Usunięcie stamtąd słów **`QUIET`** oraz **`RHGB`** sprawi, że podczas startu zamiast ekranu ładowania będą widoczne pełne komunikaty diagnostyczne. Zmodyfikowaną konfigurację uruchamia się kombinacją klawiszy **`Ctrl + X`** (jest to zmiana jednorazowa).
* **Trwała zmiana:** 
  Aby zmiany były stałe, należy wyedytować plik **`/etc/default/grub`**, a następnie zaktualizować plik konfiguracyjny GRUB-a poleceniem:
  ```bash
  grub2-mkconfig > /boot/grub2/grub.cfg
```
```



## 4. Dodatkowe parametry ratunkowe dopisywane na końcu linii jądra w GRUB

W przypadku poważniejszych problemów, na końcu linii startowej jądra można dopisać wybraną flagę:

* **`rd.break`** – Zatrzymuje procedurę rozruchową w fazie `initramfs`. Niezwykle przydatne, gdy nie znamy hasła root (pozwala zamontować `/sysroot` jako RW, zrobić `chroot` i zresetować hasło).
* **`init=/bin/sh`** lub **`init=/bin/bash`** – Wymusza uruchomienie powłoki natychmiast po załadowaniu jądra. Opcja przydatna, ale czasami traci się przez nią dostęp do niektórych konsol czy funkcji systemowych.
* **`systemd.unit=emergency.target`** – Uruchamia system w trybie minimalnym (bardzo ograniczona liczba jednostek systemd). Wymaga podania hasła root. Można zweryfikować stan poleceniem `systemctl list-units`.
* **`systemd.unit=rescue.target`** – Uruchamia nieco więcej jednostek niż tryb awaryjny, doprowadzając system do bardziej kompletnego stanu operacyjnego (również wymaga hasła root).

---

## 5. Użycie dysku ratunkowego (Rescue Disc)

Jeśli system w ogóle nie wstaje, najbezpieczniej jest zbootować instalator/płytę ratunkową i wybrać opcję **Rescue a Red Hat system**.

1. System ratunkowy automatycznie odnajdzie Twoją instalację z twardego dysku i podmontuje ją w katalogu **`/mnt/sysimage`**.
2. Aby bezpiecznie pracować na plikach uszkodzonego systemu, musisz wywołać tzw. zmianę roota (chroot):
```bash
chroot /mnt/sysimage
```



3. Dopiero po tym kroku możesz wykonywać operacje naprawcze.

---

## 6. Naprawa uszkodzonego programu rozruchowego (GRUB2)

Najczęstszą przyczyną problemów z rozruchem są błędy w samym programie GRUB. Po podmontowaniu systemu i wejściu przez `chroot` (jak wyżej), naprawa polega na ponownej instalacji bootloadera na partycji systemowej:

```bash
grub2-install /dev/sdX

```

*(gdzie `/dev/sdX` to wskazanie głównego dysku systemowego)*.

---

## 7. Odbudowa obrazu Initramfs (`dracut`)

Jeżeli uszkodzeniu uległ obraz `initramfs`, można go wygenerować od nowa za pomocą narzędzia **`dracut`**:

```bash
dracut --force

```

* Bez parametrów dracut stworzy nowy obraz dla aktualnie załadowanego jądra.
* Konfiguracja dracuta znajduje się w pliku głównym `/etc/dracut.conf` oraz w katalogach `/etc/dracut.conf.d/` i `/usr/lib/dracut/dracut.conf.d/`.

---

## 8. Błędy montowania systemów plików (`/etc/fstab`)

Błędy w pliku `/etc/fstab` (np. błędny UUID lub zła opcja montowania) uniemożliwiają prawidłowy rozruch.

1. System poprosi o podanie hasła root do celów konserwacji (*Give root password for maintenance*).
2. Uruchomione zostanie narzędzie **`fsck`** do sprawdzenia integralności systemów plików.
3. Gdy uzyskasz dostęp do powłoki, zamontuj system w trybie zapisu:
```bash
mount -o remount,rw /

```


4. Popraw błędne wpisy w pliku `/etc/fstab`.
5. Przeanalizuj logi systemowe w poszukiwaniu przyczyny awarii za pomocą polecenia:
```bash
journalctl -xb

```


# CHAPTER 21 - SELinux (Security-Enhanced Linux)

SELinux to mechanizm kontroli dostępu w jądrze systemu Linux. Na egzaminie RHCSA pamiętaj o kluczowej zasadzie: **na sam koniec systemu SELinux musi być włączony i działać w trybie Enforcing!**

## 1. Podstawowe pojęcia SELinux

* **Policy (Polis):** Kolekcja reguł definiujących, które źródło ma dostęp do jakiego celu.
* **Source domain:** Obiekt próbujący uzyskać dostęp (zazwyczaj proces lub użytkownik).
* **Target domain:** Zasób, do którego źródło próbuje uzyskać dostęp (zazwyczaj plik lub port sieciowy).
* **Context (Etykieta bezpieczeństwa):** Bezpieczna etykieta kategoryzująca obiekty w SELinux. Składa się z trzech części (z których najważniejszy dla administratora jest **Type** z końcówką `_t`):
  1. *User* (np. `system_u`)
  2. *Role* (np. `object_r`)
  3. *Type* (np. `httpd_sys_content_t` – kluczowy na egzaminach).

---

## 2. Tryby pracy SELinux

SELinux może działać w trzech stanach:
1. **Enforcing Mode:** SELinux aktywnie wymusza reguły i blokuje niedozwolony dostęp.
2. **Permissive Mode:** Naruszenia polityki są dozwolone i logowane, ale nic nie jest faktycznie blokowane (przydatne do diagnozowania).
3. **Disabled:** SELinux całkowicie wyłączony.

### Zarządzanie trybem:
* Plik konfiguracyjny (zmiana wymaga **rebootu** systemu): `/etc/sysconfig/selinux` (parametr `SELINUX=enforcing|permissive|disabled`).
* Sprawdzenie aktualnego trybu: **`getenforce`**
* Sprawdzanie szczegółów (w tym statusu polityki): **`sestatus -v`**
* Szybka zmiana w czasie rzeczywistym (runtime): **`setenforce 0`** (Permissive) lub **`setenforce 1`** (Enforcing). *Uwaga: Nie da się w ten sposób przejść bezpośrednio z trybu `disabled` na `enforcing` bez restartu systemu.*

---

## 3. Wyświetlanie kontekstów SELinux (`-Z`)

Większość standardowych narzędzi systemowych posiada przełącznik **`-Z`** (wielka litera), który pozwala podejrzeć etykiety SELinux:
* `ls -Z /`
* `ps Zaux`
* `netstat -Ztulpen`

---

## 4. Zarządzanie kontekstami plików (`semanage` i `restorecon`)

Do zmiany kontekstów plików służy zaawansowane narzędzie **`semanage`** (wchodzi w skład pakietu `policycoreutils-python`). Stosowanie polecenia `chcon` jest odradzane na egzaminie, ponieważ zmiany dokonane przez `chcon` zostaną nadpisane przy pierwszym przeładowaniu kontekstów!

### Procedura trwałego dodania kontekstu dla katalogu:
1. Sprawdź obecny kontekst (np. `ls -Zd /mydir`), interesuje Cię typ z końcówką `_t`.
2. Dodaj wpis do polityki SELinux za pomocą `semanage` z flagą `-a` (append) oraz `-t` (typ), podając na końcu ścieżkę jako wyrażenie regularne:
   ```bash
   semanage fcontext -a -t httpd_sys_content_t "/mydir(/.*)?"

```

3. Ponieważ powyższy krok zmienia jedynie reguły w bazie polityki, musisz fizycznie zaktualizować etykiety na dysku za pomocą polecenia **`restorecon`**:
```bash
restorecon -R -v /mydir

```



### Dziedziczenie kontekstów przy operacjach na plikach:

* **Nowy plik:** Dziedziczy kontekst po katalogu nadrzędnym.
* **Kopiowanie (`cp`):** Traktowane jak stworzenie nowego pliku – dziedziczy kontekst katalogu docelowego.
* **Przenoszenie (`mv`) lub kopiowanie z zachowaniem atrybutów (`cp -a`):** Zachowuje oryginalny kontekst pliku.

### Globalne odzyskiwanie kontekstów:

* Jeśli popsujesz konteksty w całym systemie, możesz wymusić ich przeliczenie:
* Na żywo: `restorecon -RV /`
* Przy następnym uruchomieniu (tworząc plik pusty): `touch /.autorelabel` i restart systemu (*Uwaga: ta operacja może trwać bardzo długo, niezalecane na egzaminie, jeśli nie jest konieczne*).



---

## 5. Dokumentacja SELinux dla usług (`sepolicy`)

Aby szybko odnaleźć odpowiednie ustawienia SELinux dla danej usługi, najlepiej skorzystać z dedykowanych stron podręcznika (man pages):

1. Zainstaluj narzędzia dokumentacji:
```bash
yum -y install policycoreutils-devel

```


2. Wygeneruj dedykowane strony man dla polityk:
```bash
sepolicy manpage -a -p /usr/share/man/man8

```


3. Zaktualizuj bazę manpages:
```bash
mandb

```


4. Wyszukaj interesujące Cię opcje, np. dla serwera WWW Apache:
```bash
man -k _selinux | grep http

```



---

## 6. Używanie Booleanów (Przełączników polityk)

Zamiast modyfikować całe reguły, SELinux udostępnia tzw. *booleans* (zmienne typu prawda/fałsz), które pozwalają włączać lub wyłączać konkretne uprawnienia usług.

* Wyświetlenie wszystkich booleanów i ich stanu: **`getsebool -a`** (najlepiej filtrować przez `grep`).
* Sprawdzenie stanu (obecnego i domyślnego): **`semanage boolean -l`**
* Zmiana booleana w czasie rzeczywistym (runtime):
```bash
setsebool nazwa_rule ON|OFF

```


* **Trwała zmiana (persistent):** Dodaj wielką flagę **`-P`**:
```bash
setsebool -P nazwa_rule ON|OFF

```



---

## 7. Diagnostyka problemów z SELinux (Audyt i Sealert)

Gdy SELinux blokuje dostęp, zdarzenie ląduje w pliku **`/var/log/audit/audit.log`** pod oznaczeniem **AVC** (Access Vector Cache). Surowe logi z audytu są jednak trudne do odczytania.

### Narzędzie `setroubleshoot` (sealert):

1. Zainstaluj serwer analizy błędów:
```bash
yum -y install setroubleshoot-server

```


2. Od tej pory czytelne podsumowania i sugestie rozwiązań problemów z SELinux trafiają również do pliku `/var/log/messages`.
3. Możesz ręcznie przeanalizować plik logu audytu poleceniem:
```bash
sealert -a /var/log/audit/audit.log

```


# CHAPTER 22 - Configuring a firewall

W przeszłości do konfiguracji zapory sieciowej stosowano narzędzie **IPTABLES**, które opiera się na mechanizmie jądra o nazwie **NETFILTER**. Choć nadal jest to technicznie możliwe, obecnie nie jest zalecanym standardem. W systemach RHEL domyślnym rozwiązaniem jest **FIREWALLD**. 

> **Ostrzeżenie:** Nigdy nie należy używać `iptables` i `firewalld` jednocześnie na tym samym systemie, ponieważ wchodzą one ze sobą w bezpośredni konflikt!

---

## 1. Koncepcja Stref (Zones) w Firewalld

**Firewalld** opiera się na koncepcji stref (`zones`), które definiują poziom zaufania dla ruchu sieciowego. Domyślnie reguły stref aplikowane są **tylko do ruchu przychodzącego** (ruch wychodzący nie jest restrykcyjnie filtrowany).

### Kolejność dopasowania stref dla przychodzących pakietów:
1. Skąd przyszedł pakiet (źródłowy adres IP).
2. Jaki interfejs sieciowy jest używany.
3. Domyślna strefa systemu.

### Standardowe strefy w Firewalld:
* **block:** Wszystkie przychodzące połączenia są odrzucane z komunikatem `icmp-host-prohibited`. Dozwolone są wyłącznie połączenia zainicjowane z wnętrza tego systemu.
* **dmz:** Dla komputerów w strefie zdemilitaryzowanej (DMZ). Dozwolone są tylko wybrane połączenia przychodzące oraz ograniczony dostęp do sieci wewnętrznej.
* **drop:** Dowolne przychodzące pakiety są bezwzględnie porzucane (dropped) bez żadnej odpowiedzi.
* **external:** Do użytku w sieciach zewnętrznych z włączonym maskowaniem (NAT). Używane głównie na routerach. Dozwolone są tylko wybrane połączenia przychodzące.
* **home:** Dla sieci domowych. Większość komputerów w tej samej sieci jest uważana za zaufane, akceptowane są tylko wybrane połączenia przychodzące.
* **internal:** Dla sieci wewnętrznych. Podobnie jak *home*, większość urządzeń jest zaufana.
* **public:** Dla obszarów publicznych. Urządzenia w sieci nie są zaufane, akceptowane są tylko wybrane, ograniczone połączenia. **Jest to domyślna strefa dla wszystkich nowo tworzonych interfejsów sieciowych.**
* **trusted:** Wszystkie połączenia sieciowe są bezwzględnie akceptowane.
* **work:** Dla obszarów roboczych. Większość komputerów w sieci jest zaufana, akceptowane są tylko wybrane połączenia.

---

## 2. Usługi (Services) w Firewalld

Firewalld operuje pojęciem *usług* (co jest niezależne od jednostek service w systemd). Usługa to gotowy zestaw reguł (np. numery portów i protokoły) dla danej aplikacji (np. http, ssh).
* Wylistowanie dostępnych usług: `firewall-cmd --get-services`
* Pliki konfiguracyjne usług znajdują się w katalogu `/usr/lib/firewalld/services/` lub `/etc/firewalld/services/`.

---

## 3. Narzędzia do zarządzania: `firewall-cmd` oraz `firewall-config`

* **`firewall-cmd`** – Interfejs wiersza poleceń (CLI).
* **`firewall-config`** – Graficzny interfejs użytkownika (GUI).

> **Złota zasada Firewalld:** Wszystkie zmiany dokonane za pomocą `firewall-cmd` bez flagi `--permanent` trafiają **wyłącznie do pamięci operacyjnej (run-time)** i zostaną utracone po reboocie lub przeładowaniu. Aby zapisać zmiany na dysk, należy użyć flagi `--permanent`, a następnie odświeżyć konfigurację poleceniem `--reload`.

---

## 4. Kluczowe opcje i składnia polecenia `firewall-cmd`

* `--get-zones` – Wyświetla listę wszystkich dostępnych stref.
* `--get-default-zone` – Pokazuje aktualnie ustawioną domyślną strefę.
* `--set-default-zone=<ZONE>` – Zmienia domyślną strefę systemową.
* `--get-services` – Pokazuje wszystkie dostępne predefiniowane usługi.
* `--list-services` – Pokazuje usługi aktualnie przypisane i aktywne w danej strefie.
* `--add-service=<service-name> [--zone=<ZONE>]` – Dodaje usługę do domyślnej lub wskazanej strefy.
* `--remove-service=<service-name>` – Usuwa usługę z konfiguracji strefy.
* `--list-all [--zone=<ZONE>]` – Wyświetla kompletną konfigurację wybranej strefy (przypisane interfejsy, źródła, usługi, porty).
* `--add-port=<port/protocol> [--zone=<ZONE>]` – Otwiera konkretny port i protokół (np. `80/tcp`).
* `--remove-port=<port/protocol> [--zone=<ZONE>]` – Usuwa regułę portu z konfiguracji.
* `--add-interface=<INTERFACE> [--zone=<ZONE>]` – Przypisuje wskazany interfejs sieciowy do strefy.
* `--remove-interface=<INTERFACE> [--zone=<ZONE>]` – Usuwa interfejs ze strefy.
* `--add-source=<ipaddress/netmask> [--zone=<ZONE>]` – Wiąże ruch pochodzący z określonego adresu IP lub podsieci z daną strefą.
* `--remove-source=<ipaddress/netmask> [--zone=<ZONE>]` – Usuwa powiązanie źródłowego adresu IP.
* `--permanent` – Zapisuje konfigurację na dysku (musi być łączona z regułą, np. `firewall-cmd --permanent --add-service=http`).
* `--reload` – Odświeża konfigurację zapory z dysku, aplikując wcześniej zapisane reguły permanentne bez przerywania aktywnych połączeń.


# CHAPTER 23 - Mounting external filesystems

W systemach Linux udostępnianie i montowanie zasobów sieciowych odbywa się najczęściej za pomocą protokołów **NFS** (Network File System – dla systemów Linux/Unix) oraz **SMB/CIFS** (dla środowisk mieszanych z systemami Windows).

---

## 1. Konfiguracja serwera NFS (Skrót z praktyki)

Aby udostępnić zasób przez NFS z lokalnego serwera:
1. Utwórz folder do udostępnienia (np. `/nfs`).
2. Nadaj mu odpowiednie uprawnienia (np. `chmod 777 /nfs`).
3. Dopisz wpis do pliku **`/etc/exports`**:
   ```text
   /nfs  *(rw)
```
```

4. Uruchom i włącz wymagane usługi:
```bash
systemctl start {nfs-server,rpcbind,rpc-statd,nfs-idmapd}
systemctl enable {nfs-server,rpcbind,rpc-statd,nfs-idmapd}

```


5. Zatwierdź eksport zasobów (odpowiednik `mount -a` dla NFS):
```bash
exportfs -a

```


6. Zweryfikuj poprawność eksportu lokalnie:
```bash
showmount -e localhost

```


*(Jeśli polecenie zwraca błąd, najprawdopodobniej problem tkwi w konfiguracji zapory sieciowej – firewalla).*

---

## 2. Klient NFS i bezpieczeństwo

Do obsługi klientów NFS niezbędny jest pakiet `nfs-utils` oraz działająca usługa `rpcbind`.

* **Wersje NFS:** RHEL domyślnie korzysta z **NFS w wersji 4**. Wspiera ona tzw. **pseudo-mount**, co pozwala montować zasoby w formie `SERVER:/`. W razie potrzeby wersję można wymusić flagą `nfsvers=X` podczas montowania.
* **Wyszukiwanie zasobów na serwerze:**
1. Wykorzystanie mechanizmu Root Mount (dla NFSv4).
2. Sprawdzenie połączeń: `netstat -an | grep NFS_SERVER_IP:PORT`
3. Użycie polecenia: `showmount -e NFS_SERVER`



### Model bezpieczeństwa (Parametr `sec=`)

Domyślnie klient łączy się z serwerem, weryfikując tożsamość na podstawie lokalnych numerów **UID i GID**. Jeśli UID użytkownika na kliencie pokrywa się z UID na serwerze, uzyskuje on dostęp do plików. Opcje zabezpieczeń (`sec=`) podczas montowania to:

* **`none`** – Dostęp anonimowy (mapowany na użytkownika `nfsnobody`).
* **`sys`** – Domyślny tryb oparty na dopasowaniu wartości UID i GID.
* **`krb5`** – Użytkownicy muszą uwierzytelnić się za pomocą protokołu Kerberos.
* **`krb5i`** – Kerberos + gwarancja niezmienności danych w locie (integrity).
* **`krb5p`** – Kerberos + pełne szyfrowanie żądań (najwyższy poziom bezpieczeństwa, ale wpływa na wydajność).

*Uwaga dotycząca Kerberosa:* Wymaga poprawnej konfiguracji pliku `/etc/krb5.keytab` oraz uruchomienia usługi `nfs-secure` na kliencie.

### Ręczne montowanie NFS:

```bash
mount -t nfs 192.168.10.1:/nfs /mnt/nfs/

```

*(Pamiętaj, że lokalny folder docelowy `/mnt/nfs/` musi wcześniej istnieć!)*

---

## 3. Protokoły SMB / CIFS (Udostępnianie z Windows)

Do obsługi udziałów sieciowych Windows (SMB/CIFS) wymagane jest zainstalowanie na kliencie pakietów **`cifs-utils`** oraz **`samba-client`**.

* **Konfiguracja zapory (Firewalld):**
```bash
firewall-cmd --add-service=samba-client --permanent
firewall-cmd --reload

```


* **Wylistowanie zasobów serwera:**
```bash
smbclient -L IP_ZASOBU -U nazwa_usera

```


* **Ręczne montowanie CIFS:**
```bash
mount -t cifs -o user=GUEST //192.168.122.200/data /mnt

```


*(Montowanie jako guest tworzy udziały w trybie tylko do odczytu. Połączenie jako realny użytkownik Samby wywoła monit o podanie hasła).*

---

## 4. Trwałe montowanie w `/etc/fstab`

Ręczne montowanie znika po reboocie. Aby zautomatyzować ten proces, wpisy umieszcza się w pliku `/etc/fstab`.

### Przykład wpisu NFS dla `/etc/fstab`:

```text
server1:/share  /nfs/mount/point  nfs  _netdev,x-systemd.automount,sync  0 0

```

* **Kluczowe opcje:**
* `_netdev` – Informuje system, że zasób wymaga sieci (zapobiega zawieszaniu się systemu podczas startu, jeśli sieć jeszcze nie wstala).
* `x-systemd.automount` – Opóźnia faktyczne montowanie do momentu, w którym użytkownik spróbuje wejść do katalogu (automount).
* `sync` – Natychmiastowy zapis modyfikacji na serwerze (zamiast buforowania w pamięci RAM).



### Bezpieczne przechowywanie danych logowania (Credentials File) dla CIFS:

Aby nie ujawniać hasła w plaintekście bezpośrednio w pliku `/etc/fstab`, stosuje się plik konfiguracyjny (np. `/root/creds`), należący do roota i zabezpieczony uprawnieniami `600`:

```text
username=linda
password=secret
domain=mydomain

```

Wpis w `/etc/fstab` wygląda wówczas następująco:

```text
//server1/data  /mnt/data  cifs  _netdev,x-systemd.automount,credentials=/root/creds  0 0

```

> **Ważna komenda:** Użycie polecenia **`mount -i`** wymusza odczyt pliku `/etc/fstab` i zamontowanie zdefiniowanych tam zasobów zgodnie z wpisanymi wartościami.

---

## 5. AUTOFS (Automatyczne montowanie na żądanie)

Autofs to serwis działający w przestrzeni użytkownika (`user space`), który automatycznie montuje zdalne zasoby w momencie próby dostępu do nich i odmontowuje je po okresie bezczynności.

### Konfiguracja AutoFS – Krok po kroku:

1. Zainstaluj pakiet:
```bash
yum install -y autofs

```


2. Utwórz plik konfiguracyjny master w katalogu **`/etc/auto.master.d/`** (nazwa dowolna, musi kończyć się na `.autofs`), np. `/etc/auto.master.d/demo.autofs`:
```text
/shares /etc/auto.demo
/- /etc/auto.direct

```


* `/shares` – Punkt wyjściowy dla montowań **pośrednich (indirect)**.
* `/-` – Oznacza montowania **bezpośrednie (direct)**.


3. Zdefiniuj mapowanie dla montowania pośredniego w pliku **`/etc/auto.demo`**:
```text
data -rw,sync labipa:/data

```


*(Katalog `/shares/data` zostanie utworzony automatycznie w locie po wejściu do `/shares/data`).*
4. Zdefiniuj mapowanie dla montowania bezpośredniego w pliku **`/etc/auto.direct`**:
```text
/mnt -rw,sync labipa:/home

```


*(W montowaniu bezpośrednim punkt docelowy `/mnt` musi fizycznie istnieć przed uruchomieniem automountu).*
5. Uruchom i włącz usługę:
```bash
systemctl enable autofs
systemctl start autofs

```



* **Testowanie:** Wejście do katalogu `cd /shares/data` lub `cd /mnt` automatycznie wywoła zamontowanie odpowiedniego zasobu przez demona autofs.
* *Uwaga przy montowaniu SMB przez Autofs:* Ścieżka do pliku `credentials` musi być absolutna, a nazwa montowanego zasobu musi rozpoczynać się odpowiednio.


# CHAPTER 24 - Configuring Time Services

W systemach Linux zarządzanie czasem opiera się na synchronizacji zegara sprzętowego oraz programowego (systemowego).

## 1. Kluczowe pojęcia
* **Hardware clock / Real-time clock (RTC):** Zegar fizyczny znajdujący się na płycie głównej komputera.
* **System time / Software clock:** Czas utrzymywany przez jądro systemu operacyjnego.
* **Universal Time Coordinated (UTC):** Globalny standard czasu (Linux wymienia się czasem między systemami właśnie w formacie UTC).
* **Local time:** Czas odpowiadający bieżącej strefie czasowej użytkownika.
* **Daylight Savings Time (DST):** Automatyczna zmiana czasu na letni/zimowy.

---

## 2. Narzędzia do zarządzania czasem

### Polecenie `date` (Zarządzanie czasem systemowym)
* `date` – Wyświetla aktualny czas systemowy.
* `date +%d-%m-%y` – Wyświetla bieżący dzień, miesiąc i rok w określonym formacie.
* `date -s 16:03` – Ręcznie ustawia godzinę systemową (np. na 16:03).
* `date --date '@1420987251'` – Konwertuje czas w formacie **Epoch** (liczba sekund od 1 stycznia 1970 r.) na czytelną dla człowieka datę.

### Polecenie `hwclock` (Zarządzanie czasem sprzętowym)
* `hwclock -c` – Pokazuje różnicę między czasem sprzętowym a systemowym (wynik odświeża się co 10 sekund).
* `hwclock --systohc` – Synchronizuje bieżący czas systemowy z zegarem sprzętowym.
* `hwclock --hctosys` – Synchronizuje czas sprzętowy z zegarem systemowym.

---

## 3. Synchronizacja przez sieć (Chrony / NTP)

Zamiast ręcznej korekty, synchronizację czasu realizuje się za pomocą protokołu NTP. W nowoczesnych dystrybucjach RHEL standardem jest demon **`chronyd`** (zastępujący starsze rozwiązania typu `ntpd`).

* Konfiguracja serwerów czasu znajduje się w pliku **`/etc/chrony.conf`**.
* Włączenie obsługi NTP za pomocą `timedatectl`:
  ```bash
  timedatectl set-ntp 1
```
```
---

## 4. Superkomenda `timedatectl`

Narzędzie `timedatectl` zostało stworzone w celu kompleksowego zarządzania wszystkimi aspektami czasu i stref czasowych na systemach RHEL (pod spodem kontroluje działanie demona `chronyd`).

* `timedatectl list-timezones` – Wyświetla pełną listę dostępnych stref czasowych.
* `timedatectl set-ntp [0|1]` – Włącza lub wyłącza synchronizację NTP.
* `timedatectl set-local-rtc [0|1]` – Określa, czy zegar sprzętowy (RTC) działa w czasie lokalnym, czy UTC.
* `timedatectl set-timezone <Strefa>` – Ustawia strefę czasową systemu.

---

## 5. Konfiguracja strefy czasowej (Timezone)

Istnieje kilka metod ustawienia strefy czasowej w systemie Linux:

1. Użycie polecenia **`timedatectl`** (najbardziej zalecana metoda).
2. Użycie narzędzia interaktywnego **`tzselect`**.
3. Ręczne utworzenie dowiązania symbolicznego (symlinka) `/etc/localtime` wskazującego odpowiedni plik strefy z katalogu `/usr/share/zoneinfo/`:

```bash
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```


4. Wykorzystanie graficznego narzędzia `system-config-date`.


