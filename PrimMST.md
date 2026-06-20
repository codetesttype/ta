# Skripta — Primov algoritam (MST) + Min-Gomila | zadatak3.c

---

## Sadržaj

1. [Šta je MST?](#1-šta-je-mst)
2. [Strukture podataka](#2-strukture-podataka)
3. [Graf — inicijalizacija i izgradnja](#3-graf--inicijalizacija-i-izgradnja)
4. [Min-Gomila (Min-Heap)](#4-min-gomila-min-heap)
5. [Heap operacije](#5-heap-operacije)
6. [Primov algoritam](#6-primov-algoritam)
7. [Ispis MST-a](#7-ispis-mst-a)
8. [main()](#8-main)
9. [Razlike u odnosu na zadatak2 (Dijkstra)](#9-razlike-u-odnosu-na-zadatak2-dijkstra)
10. [Kompleksnost](#10-kompleksnost)

---

## 1. Šta je MST?

**Minimalno razapinjuće stablo (MST)** je podgraf koji:
- Povezuje **sve čvorove** grafa
- Ne sadrži **cikluse** (stablo)
- Ima **minimalnu ukupnu težinu** grana

Primov algoritam gradi MST **pohlepno**: krenemo od jednog čvora i u svakom koraku dodajemo **najlakšu granu** koja spaja već odabrane čvorove s neodabranim.

**Ključna razlika od Dijkstre:**  
- Dijkstra: `ključ[v] = dist[u] + tezina(u,v)` — ukupna distanca od izvora  
- Prim: `ključ[v] = tezina(u,v)` — samo težina **te jedne grane**

---

## 2. Strukture podataka

### Čvor liste susedstva

```c
typedef struct CvorListe {
    int              dest;     // odredišni čvor
    int              tezina;   // težina grane
    struct CvorListe *sledeci; // sledeći sused
} CvorListe;
```

### Graf

```c
typedef struct {
    int       brojCvorova;               // broj čvorova
    char      oznaka[MAX_CVOROVA][4];    // string oznake čvorova (do 3 slova + '\0')
    CvorListe *listaSledecih[MAX_CVOROVA]; // liste susedstva
} Graf;
```

**Razlika od zadatka 2:** Čvorovi su označeni **stringovima** (npr. `"C"`, `"B"`, `"F"`) a ne samo jednim slovom ili indeksom.

---

### Čvor gomile

```c
typedef struct {
    int cvor;   // indeks čvora u grafu
    int kljuc;  // trenutni minimalni ključ (= najmanja grana do MST-a)
} CvorGomile;
```

### Min-Gomila

```c
typedef struct {
    CvorGomile podaci[MAX_CVOROVA];  // niz čvorova gomile (statički, bez malloc)
    int        pozicija[MAX_CVOROVA]; // pozicija[v] = indeks čvora v u podaci[]
    int        velicina;              // trenutni broj elemenata
} MinGomila;
```

**Razlika od zadatka 2:** Ovde je heap implementiran **statički** (niz unutar strukture, bez dinamičke alokacije), a u zadatku 2 je bio dinamički (pokazivači + malloc). Obe varijante su validne.

---

## 3. Graf — inicijalizacija i izgradnja

### `inicijalizujGraf`

```c
void inicijalizujGraf(Graf *g) {
    g->brojCvorova = 0;
    for (int i = 0; i < MAX_CVOROVA; i++)
        g->listaSledecih[i] = NULL;
}
```

---

### `dodajCvor` — dodaje čvor sa string oznakom

```c
int dodajCvor(Graf *g, const char *ozn) {
    int idx = g->brojCvorova++;
    strncpy(g->oznaka[idx], ozn, 3);
    g->oznaka[idx][3] = '\0';     // null-terminator na sigurno
    return idx;                    // vraća indeks novog čvora
}
```

Čvorovi se dodaju redom, svaki dobija sledeći slobodni indeks.

---

### `pronadjiCvor` — string oznaka → indeks

```c
int pronadjiCvor(Graf *g, const char *ozn) {
    for (int i = 0; i < g->brojCvorova; i++)
        if (strcmp(g->oznaka[i], ozn) == 0)
            return i;
    return -1;
}
```

Ekvivalent `indeksOznake` iz zadatka 1, ali za stringove.

---

### `dodajUsmerenGranu` — interna pomoćna funkcija

```c
static void dodajUsmerenGranu(Graf *g, int izvor, int dest, int t) {
    CvorListe *cvor = malloc(sizeof(CvorListe));
    cvor->dest      = dest;
    cvor->tezina    = t;
    cvor->sledeci   = g->listaSledecih[izvor];  // dodaj na početak liste
    g->listaSledecih[izvor] = cvor;
}
```

`static` znači da je funkcija vidljiva samo unutar ovog fajla.

---

### `dodajGranu` — javna funkcija, dodaje neusmerenu granu

```c
void dodajGranu(Graf *g, const char *oznIzvor, const char *oznDest, int t) {
    int iz  = pronadjiCvor(g, oznIzvor);
    int do_ = pronadjiCvor(g, oznDest);
    // provera greške ako čvor nije pronađen...
    dodajUsmerenGranu(g, iz, do_, t);   // grana izvor → dest
    dodajUsmerenGranu(g, do_, iz, t);   // grana dest → izvor
}
```

Svaka grana se dodaje dvaput (neusmereno).  
**Napomena:** Grane mogu imati **negativne težine** (npr. `F-D` = -1 u zadatku). Primov algoritam to podržava! Dijkstra NE bi.

---

## 4. Min-Gomila (Min-Heap)

### `inicijalizujGomilu`

```c
void inicijalizujGomilu(MinGomila *h) {
    h->velicina = 0;
    for (int i = 0; i < MAX_CVOROVA; i++)
        h->pozicija[i] = -1;  // -1 = čvor nije u gomili
}
```

`pozicija[v] = -1` označava da čvor `v` **nije** u gomili.

---

### `sadrzuGomila` — provera članstva

```c
int sadrzuGomila(MinGomila *h, int cvor) {
    return h->pozicija[cvor] != -1;
}
```

Čvor je u gomili ako mu je pozicija različita od -1.  
(Za razliku od zadatka 2 gde se proveravalo `poz[v] < velicina`)

---

## 5. Heap operacije

### `zameni` — zamena dva čvora u gomili

```c
static void zameni(MinGomila *h, int i, int j) {
    h->pozicija[h->podaci[i].cvor] = j;   // ažuriraj pozicije PRE zamene
    h->pozicija[h->podaci[j].cvor] = i;
    CvorGomile tmp = h->podaci[i];
    h->podaci[i]   = h->podaci[j];
    h->podaci[j]   = tmp;
}
```

---

### `popraviGore` — penjanje čvora gore (posle smanjenja ključa)

```c
static void popraviGore(MinGomila *h, int i) {
    while (i > 0) {
        int roditelj = (i - 1) / 2;
        if (h->podaci[roditelj].kljuc > h->podaci[i].kljuc) {
            zameni(h, roditelj, i);
            i = roditelj;
        } else {
            break;  // heap uslov zadovoljen
        }
    }
}
```

Koristi se u `umetniUGomilu` i `smanjiKljuc`. Penje čvor sve dok je manji od roditelja.

---

### `popraviDole` — spuštanje čvora dole (posle izvlačenja minimuma)

```c
static void popraviDole(MinGomila *h, int i) {
    int najmanji = i;
    int levo     = 2 * i + 1;
    int desno    = 2 * i + 2;

    if (levo  < h->velicina && h->podaci[levo].kljuc  < h->podaci[najmanji].kljuc)
        najmanji = levo;
    if (desno < h->velicina && h->podaci[desno].kljuc < h->podaci[najmanji].kljuc)
        najmanji = desno;

    if (najmanji != i) {
        zameni(h, i, najmanji);
        popraviDole(h, najmanji);  // rekurzivno nadole
    }
}
```

---

### `umetniUGomilu` — ubacivanje čvora

```c
void umetniUGomilu(MinGomila *h, int cvor, int kljuc) {
    int i = h->velicina++;        // dodaj na kraj
    h->podaci[i].cvor  = cvor;
    h->podaci[i].kljuc = kljuc;
    h->pozicija[cvor]  = i;
    popraviGore(h, i);            // popravka heapa
}
```

Dodaje na kraj niza, pa `popraviGore` penje čvor na pravo mesto.

---

### `izvuciMinimum` — uzimanje čvora s minimalnim ključem

```c
CvorGomile izvuciMinimum(MinGomila *h) {
    CvorGomile minCvor = h->podaci[0];  // minimum je uvek na indeksu 0

    h->pozicija[minCvor.cvor] = -1;     // označava da je čvor izašao iz gomile

    h->velicina--;
    if (h->velicina > 0) {
        h->podaci[0]                   = h->podaci[h->velicina]; // poslednji na vrh
        h->pozicija[h->podaci[0].cvor] = 0;
        popraviDole(h, 0);             // popravka heapa
    }

    return minCvor;
}
```

**Koraci:**
1. Uzmi koren (minimum)
2. Postavi mu `pozicija = -1` (više nije u gomili)
3. Poslednji element na vrh
4. `popraviDole(0)` da bi heap bio ispravan

---

### `smanjiKljuc` — ažuriranje ključa čvora

```c
void smanjiKljuc(MinGomila *h, int cvor, int noviKljuc) {
    int i = h->pozicija[cvor];    // nađi poziciju čvora
    h->podaci[i].kljuc = noviKljuc;
    popraviGore(h, i);            // penjanje jer je ključ manji
}
```

---

## 6. Primov algoritam

```c
void primMST(Graf *g, int koren, int roditelj[], int kljuc[]) {
    int n = g->brojCvorova;
    MinGomila gomila;

    // Inicijalizacija: sve distance = BESKONAČNO, roditelji = -1
    for (int u = 0; u < n; u++) {
        kljuc[u]    = BESKONACNO;
        roditelj[u] = -1;
    }

    kljuc[koren] = 0;  // koren ima ključ 0 (ulazi prvi u MST)

    inicijalizujGomilu(&gomila);
    for (int u = 0; u < n; u++)
        umetniUGomilu(&gomila, u, kljuc[u]);  // svi čvorovi u gomilu
```

### Glavna petlja

```c
    while (gomila.velicina > 0) {
        CvorGomile minElem = izvuciMinimum(&gomila);  // čvor s min ključem
        int u = minElem.cvor;

        for (CvorListe *sused = g->listaSledecih[u]; sused != NULL; sused = sused->sledeci) {
            int v = sused->dest;
            int t = sused->tezina;

            // Ažuriraj ako je v još u gomili i grana (u,v) je lakša od dosad poznate
            if (sadrzuGomila(&gomila, v) && t < kljuc[v]) {
                roditelj[v] = u;
                kljuc[v]    = t;
                smanjiKljuc(&gomila, v, t);
            }
        }
    }
}
```

**Ključna linija — pohlepni izbor:**
```c
if (sadrzuGomila(&gomila, v) && t < kljuc[v])
```

- `sadrzuGomila` → v još nije u MST-u
- `t < kljuc[v]` → grana (u,v) je lakša od dosad poznate grane do v

**Šta je `kljuc[v]`?**  
Minimalna težina grane između v i bilo kog čvora koji je već u MST-u. Kada se v izvuče iz gomile, ta grana je **optimalna** za MST.

---

### Vizualizacija toka (primer iz zadatka)

```
Polazni čvor: C, kljuc[C]=0

Iteracija 1: Uzmi C (kljuc=0)
  Susedi: B(2), F(3), D(1)
  kljuc[B]=2, rod[B]=C; kljuc[F]=3, rod[F]=C; kljuc[D]=1, rod[D]=C
  Gomila (min): D(1), B(2), F(3), E(INF), H(INF), G(INF)

Iteracija 2: Uzmi D (kljuc=1)  ← grana C-D, težina 1
  Susedi: C(1 — već van gomile), F(-1), G(4)
  kljuc[F] = min(3, -1) = -1, rod[F]=D  ← grana D-F, težina -1 (bolja!)
  kljuc[G]=4, rod[G]=D
  Gomila: F(-1), B(2), G(4), E(INF), H(INF)

Iteracija 3: Uzmi F (kljuc=-1)  ← grana D-F, težina -1
  ...itd.
```

---

## 7. Ispis MST-a

```c
void ispisiMST(Graf *g, int roditelj[], int kljuc[], int koren) {
    printf("\nGrane minimalnog razapinjuceg stabla (MST):\n");
    int ukupnaTezina = 0;

    for (int v = 0; v < g->brojCvorova; v++) {
        if (v == koren) continue;  // koren nema roditelja

        int u = roditelj[v];
        printf("%s -|%d|-> %s\n", g->oznaka[u], kljuc[v], g->oznaka[v]);
        ukupnaTezina += kljuc[v];
    }

    printf("Ukupna tezina MST-a: %d\n", ukupnaTezina);
}
```

Format: `RODITELJ -|TEŽINA|-> ČVOR`

Obilazi sve čvorove osim korena i ispisuje granu koja ga je dodala u MST (`roditelj[v] → v` sa težinom `kljuc[v]`).

---

## 8. main()

```c
Graf g;
inicijalizujGraf(&g);

// Čvorovi se dodaju po imenu
dodajCvor(&g, "C"); dodajCvor(&g, "B"); dodajCvor(&g, "F");
dodajCvor(&g, "D"); dodajCvor(&g, "E"); dodajCvor(&g, "H"); dodajCvor(&g, "G");

// Grane s težinama (može biti negativna: F-D = -1)
dodajGranu(&g, "C", "B",  2);
dodajGranu(&g, "C", "F",  3);
dodajGranu(&g, "C", "D",  1);
dodajGranu(&g, "B", "F",  1);
dodajGranu(&g, "B", "E",  4);
dodajGranu(&g, "F", "D", -1);  // negativna težina!
dodajGranu(&g, "F", "H",  3);
dodajGranu(&g, "E", "H",  3);
dodajGranu(&g, "D", "G",  4);
dodajGranu(&g, "G", "H",  1);

int koren = pronadjiCvor(&g, "C");
int roditelj[MAX_CVOROVA];
int kljuc[MAX_CVOROVA];

primMST(&g, koren, roditelj, kljuc);
ispisiMST(&g, roditelj, kljuc, koren);
oslobodiGraf(&g);
```

---

## 9. Razlike u odnosu na zadatak2 (Dijkstra)

| Karakteristika | Dijkstra (zadatak2) | Prim (zadatak3) |
|---|---|---|
| **Cilj** | Najkraće putanje od izvora | Minimalno razapinjuće stablo |
| **Ključ čvora** | `dist[u] + tezina(u,v)` | `tezina(u,v)` (samo ta grana) |
| **Negativne grane** | NE — ne radi ispravno | DA — podržano |
| **Izlaz** | Distance + putanje | Roditelji (stablo) + ukupna težina |
| **Heap alokacija** | Dinamička (malloc) | Statička (niz u strukturi) |
| **Oznake čvorova** | Char niz | String niz (do 3 slova) |
| **Inicijalizacija heapa** | Sve in-place, velicina=V na kraju | Umetanje jednog po jednog |
| **Provera u heapu** | `poz[v] < velicina` | `pozicija[v] != -1` |

---

## 10. Kompleksnost

| Operacija | Složenost |
|---|---|
| Inicijalizacija | O(V log V) |
| Svako `izvuciMinimum` | O(log V), V puta → O(V log V) |
| Svako `smanjiKljuc` | O(log V), E puta → O(E log V) |
| **Ukupno** | **O((V + E) log V)** |

---

## Kratki podsednik — šta pamtiti za ispit

1. Prim = pohlepni algoritam za MST: uvek bira **najlakšu granu** do neposećenog čvora
2. `kljuc[v]` = težina najlakše grane između v i trenutnog MST-a (NE ukupna distanca!)
3. `roditelj[v]` = čvor u MST-u koji je "primio" v
4. Radi i sa negativnim težinama (za razliku od Dijkstre)
5. `pozicija[v] = -1` → čvor je izvučen iz gomile (u MST-u)
6. Ispis: za svaki čvor `v != koren` → grana je `roditelj[v] —[kljuc[v]]— v`
7. Kompleksnost: **O((V+E) log V)**
