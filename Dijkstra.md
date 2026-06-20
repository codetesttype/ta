# Skripta — Dijkstrin algoritam + Min-Heap | zadatak2.c

---

## Sadržaj

1. [Pregled arhitekture](#1-pregled-arhitekture)
2. [Lista susedstva — Graf](#2-lista-susedstva--graf)
3. [Min-Heap (prioritetni red)](#3-min-heap-prioritetni-red)
4. [Heap operacije](#4-heap-operacije)
5. [Dijkstrin algoritam](#5-dijkstrin-algoritam)
6. [Ispis putanja](#6-ispis-putanja)
7. [main()](#7-main)
8. [Kompleksnost](#8-kompleksnost)

---

## 1. Pregled arhitekture

Dijkstra nalazi **najkraće putanje** od jednog izvora do svih ostalih čvorova u **težinskom grafu sa nenegativnim težinama**.

Implementacija koristi:
- **Lista susedstva** za reprezentaciju grafa
- **Min-Heap** (binarni min-gomila) kao prioritetni red za efikasno uzimanje čvora s najmanjom distancom
- **Niz roditelja** za rekonstrukciju putanja

```
Graf (lista susedstva)          Min-Heap
  0 → [2,5] → [1,1]             [src, 0]
  1 → [0,1] → [4,2]           ↙         ↘
  2 → [0,5] → [3,1] → ...    [A, INF]  [B, INF]
  ...                          ...
```

---

## 2. Lista susedstva — Graf

### Čvor liste susedstva

```c
typedef struct CvorListeSusedstva {
    int dest;                       // odredišni čvor
    int tezina;                     // težina grane
    struct CvorListeSusedstva *sled; // sledeći sused
} CvorListeSusedstva;
```

### Lista za jedan čvor

```c
typedef struct ListaSusedstva {
    struct CvorListeSusedstva *glava; // glava ulančane liste suseda
} ListaSusedstva;
```

### Ceo graf

```c
typedef struct Graf {
    int V;                   // broj čvorova
    struct ListaSusedstva *niz; // niz listi susedstva, jedna po čvoru
} Graf;
```

---

### `kreirajGraf`

```c
Graf *kreirajGraf(int V) {
    Graf *graf = malloc(sizeof(Graf));
    graf->V   = V;
    graf->niz = malloc(V * sizeof(ListaSusedstva));
    for (i = 0; i < V; ++i)
        graf->niz[i].glava = NULL;  // sve liste prazne
    return graf;
}
```

---

### `dodajGranu` — dodaje **neusmerenu** granu

```c
void dodajGranu(Graf *graf, int src, int dest, int tezina) {
    // grana src → dest
    CvorListeSusedstva *noviCvor = noviCvorListeSusedstva(dest, tezina);
    noviCvor->sled = graf->niz[src].glava;
    graf->niz[src].glava = noviCvor;   // dodaj na POČETAK liste

    // grana dest → src (neusmereno!)
    noviCvor = noviCvorListeSusedstva(src, tezina);
    noviCvor->sled = graf->niz[dest].glava;
    graf->niz[dest].glava = noviCvor;
}
```

Svaka grana se dodaje **dvaput** (u oba smera) jer je graf neusmerен.  
Novo se uvek dodaje na **početak** liste (O(1)), pa susedi nisu sortirani.

---

## 3. Min-Heap (prioritetni red)

Min-Heap je osnovna struktura koja omogućava Dijkstri da uvek izvuče čvor s **trenutno najmanjom distancom** efikasno.

### Čvor heapa

```c
typedef struct CvorHeapa {
    int v;    // indeks čvora u grafu
    int dist; // trenutna (procenjena) najkraća distanca
} CvorHeapa;
```

### Min-Heap struktura

```c
typedef struct MinHeap {
    int velicina;    // trenutni broj elemenata
    int kapacitet;   // maksimalni broj elemenata
    int *poz;        // poz[v] = indeks čvora v u nizu heapa
    CvorHeapa **niz; // niz pokazivača na čvorove heapa
} MinHeap;
```

**Zašto `poz` niz?**  
Ovo je ključ efikasnosti! `poz[v]` nam kaže gde se čvor `v` trenutno nalazi u heap nizu. Bez toga, `smanjiKljuc` bi morao da pretražuje ceo heap O(V) da nađe čvor — sa njim je O(1) pristup + O(log V) za popravku.

**Vizualizacija:**
```
niz:   [S,0] [A,1] [B,5] [C,INF] ...
indeks:   0     1     2     3
poz:   poz[S]=0, poz[A]=1, poz[B]=2, poz[C]=3 ...
```

---

### `kreirajMinHeap`

```c
MinHeap *kreirajMinHeap(int kapacitet) {
    MinHeap *minHeap   = malloc(sizeof(MinHeap));
    minHeap->poz       = malloc(kapacitet * sizeof(int));
    minHeap->velicina  = 0;
    minHeap->kapacitet = kapacitet;
    minHeap->niz       = malloc(kapacitet * sizeof(CvorHeapa *));
    return minHeap;
}
```

---

## 4. Heap operacije

### `zameniCvoroveHeapa`

```c
void zameniCvoroveHeapa(MinHeap *minHeap, int i, int j) {
    CvorHeapa *tmp   = minHeap->niz[i];
    minHeap->niz[i]  = minHeap->niz[j];
    minHeap->niz[j]  = tmp;

    // OBAVEZNO ažuriraj poz[] posle zamene!
    minHeap->poz[minHeap->niz[i]->v] = i;
    minHeap->poz[minHeap->niz[j]->v] = j;
}
```

Posle svake zamene pozicija, `poz[]` mora biti ažuriran — inače bi `smanjiKljuc` tražio čvor na pogrešnom mestu.

---

### `heapify` — popravka heapa odozgo prema dole

```c
void heapify(MinHeap *minHeap, int idx) {
    int najmanji = idx;
    int levo     = 2 * idx + 1;  // levo dete
    int desno    = 2 * idx + 2;  // desno dete

    if (levo < minHeap->velicina &&
        minHeap->niz[levo]->dist < minHeap->niz[najmanji]->dist)
        najmanji = levo;

    if (desno < minHeap->velicina &&
        minHeap->niz[desno]->dist < minHeap->niz[najmanji]->dist)
        najmanji = desno;

    if (najmanji != idx) {
        zameniCvoroveHeapa(minHeap, najmanji, idx);
        heapify(minHeap, najmanji);  // rekurzivno nadole
    }
}
```

Koristi se nakon `izvuciMin`. Koren je zamenjen poslednjim elementom, a `heapify(0)` ga spušta na pravo mesto.

**Kako radi binarni heap u nizu:**
```
Parent(i) = (i-1)/2
LeftChild(i)  = 2*i + 1
RightChild(i) = 2*i + 2
```

---

### `izvuciMin` — uzimanje minimuma

```c
CvorHeapa *izvuciMin(MinHeap *minHeap) {
    if (prazan(minHeap)) return NULL;

    CvorHeapa *koren = minHeap->niz[0];          // minimum je uvek na indeksu 0

    CvorHeapa *poslednji = minHeap->niz[minHeap->velicina - 1];
    minHeap->niz[0]      = poslednji;            // poslednji element ide na vrh

    minHeap->poz[koren->v]     = minHeap->velicina - 1; // ažuriraj poz (ode van heapa)
    minHeap->poz[poslednji->v] = 0;

    --minHeap->velicina;
    heapify(minHeap, 0);  // popravka od vrha nadole

    return koren;
}
```

**Koraci:**
1. Sačuvaj koren (minimum)
2. Poslednji element stavi na vrh
3. Smanji veličinu
4. `heapify(0)` — spusti novi koren na pravo mesto

---

### `smanjiKljuc` — ažuriranje distance

```c
void smanjiKljuc(MinHeap *minHeap, int v, int dist) {
    int i = minHeap->poz[v];       // nađi poziciju čvora v u O(1)
    minHeap->niz[i]->dist = dist;  // postavi novu (manju) distancu

    // Penjanje gore dok je heap uslov narušen
    while (i > 0 &&
           minHeap->niz[i]->dist < minHeap->niz[(i - 1) / 2]->dist) {
        zameniCvoroveHeapa(minHeap, i, (i - 1) / 2);
        i = (i - 1) / 2;
    }
}
```

Pošto se distanca **smanjuje**, čvor može da se popne gore (prema manjem ključu = vrhu heapa). Penjanje je O(log V).

---

### `uHeapu` — provera da li je čvor još u heapu

```c
int uHeapu(MinHeap *minHeap, int v) {
    return minHeap->poz[v] < minHeap->velicina;
}
```

Kad se čvor izvuče iz heapa, njegova pozicija ostaje kao `velicina - 1` (van aktivnog dela), pa je ovo jednostavna provera.

---

## 5. Dijkstrin algoritam

```c
void dijkstra(Graf *graf, int src, const char *oznakeCvorova[])
```

### Inicijalizacija

```c
int dist[V];
int roditelj[V];
MinHeap *minHeap = kreirajMinHeap(V);

for (int v = 0; v < V; ++v) {
    dist[v]         = INT_MAX;   // sve distance nepoznate = beskonačno
    roditelj[v]     = -1;        // nema roditelja
    minHeap->niz[v] = noviCvorHeapa(v, dist[v]);
    minHeap->poz[v] = v;         // čvor v je na poziciji v u heapu
}

dist[src] = 0;
smanjiKljuc(minHeap, src, 0);   // izvor: distanca 0 → ide na vrh heapa
minHeap->velicina = V;           // svi čvorovi su u heapu
```

---

### Glavna petlja

```c
while (!prazan(minHeap)) {
    CvorHeapa *minCvor = izvuciMin(minHeap);  // uzmi čvor s min distancom
    int u = minCvor->v;
    free(minCvor);

    CvorListeSusedstva *susedCvor = graf->niz[u].glava;
    while (susedCvor != NULL) {
        int v      = susedCvor->dest;
        int tezina = susedCvor->tezina;

        // Relaksacija grane (u, v)
        if (uHeapu(minHeap, v) &&        // v još nije finalizovan
            dist[u] != INT_MAX &&         // u je dostupan
            dist[u] + tezina < dist[v])   // našli smo bolji put
        {
            dist[v]     = dist[u] + tezina;
            roditelj[v] = u;
            smanjiKljuc(minHeap, v, dist[v]);  // ažuriraj heap
        }
        susedCvor = susedCvor->sled;
    }
}
```

**Relaksacija** je srž Dijkstre: za granu (u,v) sa težinom `w`, ako je `dist[u] + w < dist[v]`, pronašli smo kraći put do v kroz u.

**Zašto `dist[u] != INT_MAX`?**  
Sprečava integer overflow pri sabiranje `INT_MAX + tezina`.

---

## 6. Ispis putanja

### `stampajPutanju` — rekurzivno ispisuje celu putanju

```c
void stampajPutanju(int roditelj[], int v, const char *oznakeCvorova[]) {
    if (roditelj[v] == -1) {
        printf("%s", oznakeCvorova[v]);  // bazni slučaj: izvorni čvor
        return;
    }
    stampajPutanju(roditelj, roditelj[v], oznakeCvorova);  // idi do roditelja
    printf(" -> %s", oznakeCvorova[v]);                    // pa ispiši trenutni
}
```

Rekurzija ide do izvora, pa se vraća ispisujući put **od izvora ka cilju**.

Primer: za putanju `S→B→C→D`:
```
roditelj[D]=C, roditelj[C]=B, roditelj[B]=S, roditelj[S]=-1
Rekurzija:  S  →  B  →  C  →  D
```

### `stampajResenje`

```c
void stampajResenje(int dist[], int roditelj[], int V, int src, const char *oznakeCvorova[]) {
    printf("\nNajkrace putanje od cvora %s:\n", oznakeCvorova[src]);
    for (int i = 0; i < V; ++i) {
        if (i == src) continue;
        if (dist[i] == INT_MAX) {
            printf("%s: nedostupan\n", oznakeCvorova[i]);
        } else {
            stampajPutanju(roditelj, i, oznakeCvorova);
            printf(", cost: %d\n", dist[i]);
        }
    }
}
```

---

## 7. main()

```c
const char *oznakeCvorova[] = {"S","A","B","C","D","E","F","G","H","I"};
Graf *graf = kreirajGraf(10);

// Dodavaju se neusmerene grane s težinama
dodajGranu(graf, 0, 2, 5);  // S-B, težina 5
dodajGranu(graf, 0, 1, 1);  // S-A, težina 1
// ... itd.

dijkstra(graf, 0, oznakeCvorova);  // polaz iz S (indeks 0)
oslobodiGraf(graf);
```

---

## 8. Kompleksnost

| Operacija | Sa Min-Heapom |
|---|---|
| `izvuciMin` | O(log V) |
| `smanjiKljuc` | O(log V) |
| Ukupno | **O((V + E) log V)** |

Bez heapa (naivan pristup): O(V²) — sporije za retke grafove.

**Kada koji pristup?**
- Gust graf (E ≈ V²): O(V²) je slično kao O(E log V), naivni može biti bolji
- Redak graf (E ≈ V): heap pristup je znatno brži

---

## Kratki podsednik — šta pamtiti za ispit

1. Dijkstra radi **samo sa nenegativnim težinama**
2. Min-Heap daje sledeći čvor za obradu u O(log V)
3. `poz[]` niz: čuva gde je svaki čvor u heap nizu → O(1) pristup
4. **Relaksacija**: `if (dist[u] + w < dist[v])` → ažuriraj dist[v] i roditelja
5. Čvor jednom izvučen iz heapa = distanca **finalizovana**
6. `smanjiKljuc` = novi dist + penjanje gore u heapu
7. Kompleksnost: **O((V+E) log V)**
