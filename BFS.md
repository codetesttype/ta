# Skripta — BFS (Pretraga po širini) | zadatak1.c

---

## Sadržaj

1. [Strukture podataka](#1-strukture-podataka)
2. [Red (Queue)](#2-red-queue)
3. [Graf — inicijalizacija i izgradnja](#3-graf--inicijalizacija-i-izgradnja)
4. [BFS algoritam](#4-bfs-algoritam)
5. [Ispis rezultata](#5-ispis-rezultata)
6. [main()](#6-main)
7. [Kompleksnost](#7-kompleksnost)
8. [Tok izvršavanja — primer](#8-tok-izvršavanja--primer)

---

## 1. Strukture podataka

### `Cvor` — čuva stanje svakog čvora u BFS obilasku

```c
typedef struct Cvor {
    int boja;        // BELA=0 (neposećen), SIVA=1 (u redu), CRNA=2 (obrađen)
    int distanca;    // broj grana od izvora, -1 = nedostupan
    int prethodnik;  // indeks roditelja u BFS stablu, -1 = nema
} Cvor;
```

**Zašto boja?**  
BFS prati stanje svakog čvora kroz 3 faze:
- `BELA` — čvor još nije otkriven
- `SIVA` — čvor je otkriven i dodat u red, ali mu susedu još nisu obrađeni
- `CRNA` — čvor je potpuno obrađen (izašao iz reda)

Ovo direktno odgovara pseudokodu iz CLRS knjige.

---

### `CvorListe` — jedan element liste susedstva

```c
typedef struct CvorListe {
    int indeks;            // indeks susednog čvora
    struct CvorListe *sled; // sledeći u listi
} CvorListe;
```

Svaki čvor grafa ima **listu susedstva** — jednostruko ulančanu listu koja čuva sve čvorove do kojih ide grana.

---

### `Graf` — ceo graf

```c
typedef struct {
    int n;                       // broj čvorova
    CvorListe *adj[MAX_CVOROVA]; // niz pokazivača na liste susedstva
    Cvor v[MAX_CVOROVA];         // niz stanja čvorova (boja, dist, preth.)
} Graf;
```

`adj[i]` je pokazivač na glavu liste susedstva čvora `i`.  
`v[i]` čuva BFS stanje čvora `i`.

---

### `CvorReda` — element reda za BFS

```c
typedef struct CvorReda {
    int indeks;             // koji čvor je u redu
    struct CvorReda *sled;  // sledeći u redu
} CvorReda;
```

Red je implementiran kao **jednostruko ulančana lista** sa pokazivačima na glavu i rep.

---

## 2. Red (Queue)

BFS zahteva red (FIFO). Implementiran je ručno pomoću ulančane liste.

### `enqueue` — dodavanje na kraj reda

```c
CvorReda *enqueue(CvorReda *rep, int indeks) {
    CvorReda *novi = (CvorReda *)malloc(sizeof(CvorReda));
    novi->indeks = indeks;
    novi->sled = NULL;
    if (rep != NULL) {
        rep->sled = novi;   // stari rep pokazuje na novi čvor
    }
    return novi;            // novi čvor postaje rep
}
```

**Kako se koristi:**
```c
rep = enqueue(rep, v);
if (glava == NULL) glava = rep;  // ako je red bio prazan, glava = rep
```

Vraća novi rep. Glava se menja samo pri `dequeue`.

---

### `dequeue` — uzimanje s početka reda

```c
int dequeue(CvorReda **glava, CvorReda **rep) {
    CvorReda *tmp = *glava;
    int indeks = tmp->indeks;
    *glava = tmp->sled;        // glava se pomera na sledeći
    if (*glava == NULL) {
        *rep = NULL;           // red je prazan, rep = NULL
    }
    free(tmp);
    return indeks;
}
```

Uzima čvor s **glave**, oslobađa memoriju, vraća indeks čvora.

---

## 3. Graf — inicijalizacija i izgradnja

### `initGraf`

```c
void initGraf(Graf *g, int n) {
    g->n = n;
    for (i = 0; i < n; i++) {
        g->adj[i] = NULL;  // sve liste susedstva prazne
    }
}
```

---

### `dodajGranu` — dodaje **usmerenu** granu uz **abecedni poredak**

```c
void dodajGranu(Graf *g, int src, int dest, char oznake[]) {
    CvorListe *novi = malloc(sizeof(CvorListe));
    novi->indeks = dest;
    novi->sled = NULL;

    // Ubaci na početak ako je lista prazna ILI novi čvor je abecedno manji
    if (g->adj[src] == NULL || oznake[dest] < oznake[g->adj[src]->indeks]) {
        novi->sled = g->adj[src];
        g->adj[src] = novi;
    } else {
        // Nađi pravo mesto (lista je sortirana abecedno)
        CvorListe *tmp = g->adj[src];
        while (tmp->sled != NULL && oznake[tmp->sled->indeks] < oznake[dest]) {
            tmp = tmp->sled;
        }
        novi->sled = tmp->sled;
        tmp->sled = novi;
    }
}
```

**Zašto abecedni poredak?**  
BFS obrađuje susede u redosledu kojim su u listi. Da bismo dobili deterministički (ponovljivi) izlaz koji odgovara zadatku, susedi su sortirani abecedno. To znači da BFS uvek bira abecedno manji čvor prvi.

**Napomena:** Ovo je **usmereni** graf — grana `S→A` ne znači `A→S`. Bidirekcionalnost se postiže pozivanjem `dodajGranu` u oba smera u `main()`.

---

### `indeksOznake` — mapiranje slova → indeks

```c
int indeksOznake(char oznaka, char oznake[]) {
    for (i = 0; i < MAX_CVOROVA; i++) {
        if (oznake[i] == oznaka) return i;
    }
    return -1;
}
```

Čvorovi su označeni slovima (`S`, `A`, `B`...) ali interno su indeksi (0, 1, 2...). Ova funkcija prevodi slovo u indeks.

---

## 4. BFS algoritam

```c
void bfs(Graf *g, int s) {
```

`s` je indeks izvornog čvora.

### Faza 1 — Inicijalizacija

```c
for (i = 0; i < g->n; i++) {
    g->v[i].boja = BELA;
    g->v[i].distanca = -1;
    g->v[i].prethodnik = -1;
}
```

Svi čvorovi su BELI (neposećeni), distanca -1 (nedostupan), prethodnik -1 (nema).

### Faza 2 — Priprema izvora

```c
g->v[s].boja = SIVA;
g->v[s].distanca = 0;
g->v[s].prethodnik = -1;

rep = enqueue(rep, s);
if (glava == NULL) glava = rep;
```

Izvorni čvor je SIVI (otkriven), distanca 0, dodaje se u red.

### Faza 3 — Glavna petlja

```c
while (glava != NULL) {
    u = dequeue(&glava, &rep);   // uzmi čvor iz glave reda

    sused = g->adj[u];
    while (sused != NULL) {
        v = sused->indeks;

        if (g->v[v].boja == BELA) {   // samo neposećeni susedi
            g->v[v].boja = SIVA;
            g->v[v].distanca = g->v[u].distanca + 1;  // dist = dist(u) + 1
            g->v[v].prethodnik = u;

            CvorReda *noviRed = enqueue(rep, v);
            if (glava == NULL) glava = noviRed;
            rep = noviRed;
        }
        sused = sused->sled;
    }

    g->v[u].boja = CRNA;   // u je potpuno obrađen
}
```

**Ključne tačke:**
- Obrađujemo samo BELE susede (SIVE/CRNE smo već otkrili)
- Distanca suseda = distanca tekućeg + 1 (BFS ide nivo po nivo)
- Postavljamo prethodnika pre dodavanja u red
- Čvor postaje CRNI tek kad je izašao iz reda i obrađeni su mu svi susedi

---

## 5. Ispis rezultata

```c
void ispisRezultata(Graf *g, char oznake[]) {
    for (i = 0; i < g->n; i++) {
        if (g->v[i].distanca == -1) {
            printf("%c - d: INF p: NIL\n", oznake[i]);
        } else if (g->v[i].prethodnik == -1) {
            printf("%c - d: %d p: NIL\n", oznake[i], g->v[i].distanca);
        } else {
            printf("%c - d: %d p: %c\n",
                   oznake[i],
                   g->v[i].distanca,
                   oznake[g->v[i].prethodnik]);
        }
    }
}
```

- `d: INF` → čvor nije dostignut (nedostupan)
- `p: NIL` → nema prethodnika (izvorni čvor ili nedostupan)
- Inače ispisuje distancu i slovo prethodnika

---

## 6. main()

```c
char oznake[MAX_CVOROVA] = {'S','A','B','C','D','E','F','G','H','I','J','K'};
Graf g;
initGraf(&g, MAX_CVOROVA);

// Dodaju se samo usmerene grane prema zadatku
dodajGranu(&g, indeksOznake('S', oznake), indeksOznake('A', oznake), oznake);
// ... itd.

src = indeksOznake('S', oznake);
bfs(&g, src);
ispisRezultata(&g, oznake);
```

Graf ima **12 čvorova** (S, A, B, C, D, E, F, G, H, I, J, K).  
Čvorovi G, I, J, K su **nedostupni** iz S (ili u zasebnoj komponenti).

---

## 7. Kompleksnost

| | Vrednost |
|---|---|
| **Vreme** | O(V + E) — svaki čvor i grana se obradi tačno jednom |
| **Prostor** | O(V) — za red, `dist[]`, `prethodnik[]` |

---

## 8. Tok izvršavanja — primer

Graf iz zadatka (parcijalno), polaz iz `S`:

```
Nivo 0: S
Nivo 1: A, B           (susedi S, abecednim redom)
Nivo 2: C              (sused A; B nema neposećenih suseda)
Nivo 3: D, E           (susedi C koji su BELI)
Nivo 4: F              (sused E)
Nivo 5: H              (sused F; D je već SIVI)
```

Čvorovi G, I, J, K ostaju BELI → `d: INF`.

---

## Kratki podsednik — šta pamtiti za ispit

1. BFS koristi **RED** (FIFO), ne stek
2. Tri boje: BELA → SIVA (u redu) → CRNA (obrađen)
3. Distanca se postavlja **pri otkrivanju** čvora, ne pri obradi
4. Prethodnik se postavlja zajedno sa distancom
5. Abecedni poredak liste susedstva → determinizam
6. Kompleksnost: **O(V + E)**
