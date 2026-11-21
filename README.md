#  Cyberbezpieczeństwo: Symulacja konfliktu Red Team vs. Blue Team

Projekt zaliczeniowy prezentujący obie strony konfliktu w cyberbezpieczeństwie w izolowanym środowisku kontenerowym (**Docker**). Środowisko symuluje realistyczny łańcuch ataku (rekonesans oraz atak SQL Injection) po stronie **Red Team**, a następnie jego wykrycie w czasie rzeczywistym przez system IDS **Suricata** po stronie **Blue Team**.

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat&logo=docker&logoColor=white)
![Red Team](https://img.shields.io/badge/Red_Team-Kali_Linux-557C94?style=flat&logo=kalilinux&logoColor=white)
![Blue Team](https://img.shields.io/badge/Blue_Team-Suricata_IDS-FF6B00?style=flat)
![Target](https://img.shields.io/badge/Target-DVWA-red?style=flat)

---

##  Cel projektu

1. Zbudowanie powtarzalnego, odizolowanego laboratorium sieciowego (Infrastructure as Code).
2. Przeprowadzenie kontrolowanego ataku na podatną aplikację webową (faza rekonesansu i eksploatacji).
3. Wdrożenie autorskiej reguły detekcji w systemie IDS (Intrusion Detection System) i analiza wygenerowanych alertów.

---

##  Architektura środowiska

Wszystkie maszyny działają w dedykowanej, izolowanej sieci mostkowej Dockera (`172.20.0.0/24`). Kontener obrońcy współdzieli interfejs sieciowy z ofiarą (`network_mode: "service:victim"`), co pozwala na pasywne przechwytywanie i analizę całego ruchu kierowanego do serwera WWW.

| Rola | Kontener / Obraz | Adres IP | Zadanie w projekcie |
| :--- | :--- | :--- | :--- |
| **Atakujący (Red Team)** | `kalilinux/kali-rolling` | `172.20.0.10` | Skanowanie sieci (`nmap`) oraz przeprowadzenie ataku SQL Injection (`curl`). |
| **Ofiara (Cel)** | `vulnerables/web-dvwa` | `172.20.0.20` | Podatna aplikacja webowa DVWA (*Damn Vulnerable Web Application*), wystawiona lokalnie na porcie `8080`. |
| **Obrońca (Blue Team)** | `jasonish/suricata:latest` | *(współdzielony z ofiarą)* | Nasłuch ruchu na interfejsie `eth0`, analiza pakietów pod kątem sygnatur z `local.rules`, zapis logów. |

---

##  Struktura repozytorium

```text
.
├── docker-compose.yml   # Definicja kontenerów (Atakujący, Ofiara, Obrońca) i sieci
├── local.rules          # Autorska reguła IDS Suricata wykrywająca atak SQL Injection
└── README.md            # Dokumentacja projektu
```

---

## Instrukcja uruchomienia

### Wymagania wstępne
* Zainstalowany **Docker** oraz **Docker Compose**

### 1. Uruchomienie laboratorium
```bash
git clone (link do repo)
cd <NAZWA_FOLDERU>
docker compose up -d
```

### 2. Inicjalizacja aplikacji ofiary (DVWA)
1. Otwórz w przeglądarce adres `http://localhost:8080`.
2. Zaloguj się domyślnymi danymi: `admin` / `password`.
3. Kliknij **Create / Reset Database**, a następnie zaloguj się ponownie.
4. W zakładce **DVWA Security** ustaw poziom zabezpieczeń na **Low**.

### 3. Przygotowanie narzędzi Red Team (Kali Linux)
```bash
docker exec -it lab_attacker_kali bash
apt update && apt install -y nmap curl
```

---

##  Przebieg symulacji (Atak vs. Obrona)

### Faza 1: Rekonesans (Red Team)
Z poziomu kontenera Kali Linux skanujemy otwarte porty i wersje usług serwera ofiary:
```bash
nmap -sV 172.20.0.20
```

### Faza 2: Atak SQL Injection (Red Team)
Wysłanie złośliwego zapytania wykorzystującego podatność SQLi w module aplikacji DVWA:
```bash
curl "[http://172.20.0.20/vulnerabilities/sqli/?id=1'OR'1'='1&Submit=Submit](http://172.20.0.20/vulnerabilities/sqli/?id=1'OR'1'='1&Submit=Submit)"
```

### Faza 3: Detekcja ataku (Blue Team)
System **Suricata IDS** analizuje ruch w oparciu o autorską regułę z pliku `local.rules`:
```text
alert http any any -> any 80 (msg:"[BLUE TEAM] Wykryto atak SQL Injection na DVWA!"; flow:to_server,established; http.uri; content:"vulnerabilities/sqli"; content:"OR"; nocase; sid:1000001; rev:1;)
```

Podgląd alertów na żywo na hoście:
```bash
# Linux / macOS
tail -f ./suricata-logs/fast.log

# Windows (PowerShell)
Get-Content .\suricata-logs\fast.log -Wait
```

**Przykładowy przechwycony alert:**
```text
[**] [1:1000001:1] [BLUE TEAM] Wykryto atak SQL Injection na DVWA! [**] [Priority: 3] {TCP} 172.20.0.10:49152 -> 172.20.0.20:80
```

---


### 1. Skanowanie portów (Nmap)
![Wynik skanowania Nmap](screenshots/nmap.png)

### 2. Wykrycie ataku SQL Injection przez Suricatę
![Atak SQLi i alert Suricata](screenshots/sqli-alert.png)

---

## 🧹 Zatrzymanie środowiska

```bash
docker compose down
```

---

## Wnioski

Projekt pokazuje, że skuteczne bezpieczeństwo nie polega wyłącznie na usuwaniu podatności, ale wymaga podejścia **Defense in Depth**. Nawet jeśli w aplikacji webowej pojawi się krytyczny błąd (jak SQL Injection), odpowiednio skonfigurowany system detekcji (IDS) pozwala zespołowi Blue Team natychmiast zidentyfikować źródło ataku (`172.20.0.10`), typ zagrożenia oraz podjąć działania naprawcze (np. wdrożenie zapytań parametryzowanych PDO oraz reguł WAF).

