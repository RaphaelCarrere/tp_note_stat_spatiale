# TP5 - Intro à la statistique spatiale 2A - PostGIS - corrigé

## Objectifs du TP

- Découvrir l'utilisation de **PostGIS** (extension PostgreSQL) pour la manipulation d'objets spatiaux
- Plateforme : [datalab SSP Cloud](https://datalab.sspcloud.fr)

```r
library(dplyr)
library(sf)
library(leaflet)
library(DBI)
library(dbplyr)
library(RPostgres)
```

---

## Exercice 1 — Création d'une BDD PostgreSQL avec PostGIS

### 0. Initialisation

Créer un projet et y déposer les programmes `connexion_db.R` et `create_db.R`.

### 1. Créer le service PostGIS

Dans le datalab : **Mes Services → Nouveau Service → DataBases → PostgreSQL → Configuration → Image Catalog → `postgis-standard-trixie`**

### 2. Configurer la connexion

Modifier dans `connexion_db.R` les variables : `name_database`, `user_name`, `password`, `url`, `port` (informations disponibles dans le bouton README du service).

### 3. Lancer `create_db.R`

Exécuter le programme puis vider l'environnement :

```r
rm(list = ls())
```

---

## Exercice 2 — Requêtes sur serveur PostgreSQL

Documentation DBI : [lien](https://dbi.r-dbi.org/)

### 1. Connexion et liste des tables

```r
source("connexion_db.R", encoding = "utf-8")
conn <- connecter()

DBI::dbListTables(conn)
# [1] "bpe21_01"          "bpe21_02"          "bpe21_03"
# [4] "bpe21_04"          "bpe21_metro"       "popnaiss_com"
# [7] "regions_metro"     "spatial_ref_sys"   "geography_columns"
# [10] "geometry_columns"
```

### 2. Variables de la table `popnaiss_com`

```r
DBI::dbListFields(conn, "popnaiss_com")
# [1] "codgeo" "nivgeo" "libgeo" "pop" "sexe_h" "sexe_f"
# [7] "age_00" "age_03" "age_06" "age_11" "age_18" "age_25"
# [13] "age_40" "age_55" "age_65" "age_80" "naisd18_aj"
```

### 3. `dbSendQuery()` vs `dbGetQuery()`

#### `dbSendQuery()` — renvoie un objet `PqResult`

```r
popnaiss <- DBI::dbSendQuery(conn, "SELECT * FROM popnaiss_com;")
str(popnaiss)
# Formal class 'PqResult' [package "RPostgres"] with 4 slots
```

#### `dbGetQuery()` — renvoie directement un `data.frame`

```r
popnaiss_get <- DBI::dbGetQuery(conn, "SELECT * FROM popnaiss_com;")
str(popnaiss_get)
# 'data.frame': 34867 obs. of 17 variables
```

> Pour convertir un `PqResult` en data.frame, utiliser `DBI::dbFetch()` :

```r
popnaiss <- DBI::dbSendQuery(conn, "SELECT * FROM popnaiss_com;")
popnaiss_5rows <- DBI::dbFetch(popnaiss, n = 5)  # 5 premières lignes
DBI::dbClearResult(popnaiss)                      # libérer les ressources
```

### 4. Requête filtrée — Rennes

```r
question5 <- dbGetQuery(conn, "SELECT * FROM popnaiss_com WHERE codgeo='35238';")
```

| codgeo | libgeo | pop    | naisd18_aj |
|--------|--------|--------|-----------|
| 35238  | Rennes | 217728 | 2659      |

### 5. Jointure SQL — Bruz

```r
DBI::dbListFields(conn, "bpe21_metro")
# variables disponibles : id, aav2020, an, bv2012, dep, depcom, dom, epci, ...

query <- paste0(
  "SELECT * FROM popnaiss_com INNER JOIN bpe21_metro ON ",
  "popnaiss_com.codgeo = bpe21_metro.depcom WHERE codgeo = '35047';"
)
question6 <- dbGetQuery(conn, query)
# 'data.frame': 495 obs. of 41 variables
```

### 6. Équivalent `dbplyr`

```r
popnaiss <- tbl(conn, "popnaiss_com")

# Question 5 — afficher la requête SQL générée
popnaiss %>%
  filter(codgeo == "35047") %>%
  show_query()

# Récupérer les données
pop_bruz <- popnaiss %>% filter(codgeo == "35047") %>% collect()

# Question 6 — jointure
bpe_pop_bruz <- popnaiss %>%
  filter(codgeo == "35047") %>%
  inner_join(tbl(conn, "bpe21_metro"), by = c("codgeo" = "depcom")) %>%
  show_query()   # affiche la requête SQL équivalente
  collect()      # ramène les données en mémoire
```

> `show_query()` affiche la requête SQL correspondante aux opérations dplyr — très utile pour apprendre SQL.

---

## Exercice 3 — Manipulation de la BPE

### 1. Équipements de la Manche avec `dbGetQuery()`

```r
bpe_dep50 <- dbGetQuery(conn,
  statement = paste0(
    "SELECT ID, DEPCOM, DOM, SDOM, TYPEQU, GEOMETRY ",
    "FROM bpe21_metro WHERE DEP='50';"
  )
)
# 'data.frame': 15625 obs — geometry traitée comme variable CHR
```

### 2. Avec `st_read()` — objet sf

```r
bpe_dep50 <- st_read(conn,
  query = paste0(
    "SELECT ID, DEPCOM, DOM, SDOM, TYPEQU, GEOMETRY ",
    "FROM bpe21_metro WHERE DEP='50';"
  )
)
# Classes 'sf' and 'data.frame': 15625 obs — géométrie sfc_POINT
```

**Comparaison des performances :**

```r
# Filtrage en SQL : ~0.23s
system.time({
  bpe_dep50 <- st_read(conn, query = "SELECT ... FROM bpe21_metro WHERE DEP='50';")
})

# Import total puis filtrage R : ~43s
system.time({
  bpe_dep50 <- st_read(conn, query = "SELECT * FROM bpe21_metro;") %>%
    filter(dep == "50") %>% select(id, depcom, dom, sdom, typequ)
})
```

> **Toujours préférer filtrer en SQL plutôt que d'importer la table complète.**

### 3. Système de projection

```r
st_crs(bpe_dep50)  # EPSG 2154 — Lambert-93 (France métropolitaine)

# CRS de la Réunion (bpe21_04)
st_read(conn, query = "SELECT * FROM bpe21_04") %>% st_crs()
# EPSG 2975 — UTM40S

dbGetQuery(conn, "SELECT DISTINCT(ST_SRID(geometry)) FROM bpe21_04;")
# 2975
```

### 4. Maternités par région

```r
# Avec sf + dplyr
res1 <- st_read(conn, query = "SELECT * FROM bpe21_metro WHERE TYPEQU='D107';") %>%
  group_by(reg) %>%
  summarise(n_mat = n()) %>%
  arrange(n_mat) %>%
  st_drop_geometry()

# Tout en SQL (~0.19s vs ~0.88s)
res2 <- dbGetQuery(conn,
  statement = paste0(
    "SELECT REG, COUNT(id) as n_mat FROM bpe21_metro ",
    "WHERE TYPEQU='D107' GROUP BY REG ORDER BY n_mat;"
  )
)
```

### 5. Cinémas dans un rayon de 1 km autour de la Sorbonne

Coordonnées Sorbonne : lat = 48.84864, lng = 2.34297 (WGS84)

#### a. Table des cinémas

```r
cinemas_bpe <- st_read(conn, query = "SELECT * FROM bpe21_metro WHERE TYPEQU='F303';")
# 1936 cinémas
```

#### b. Buffer autour de la Sorbonne

```r
sorbonne_buffer <- data.frame(x = 2.34297, y = 48.84864) %>%
  st_as_sf(coords = c("x", "y"), crs = 4326) %>%
  st_transform(2154) %>%
  st_buffer(1000)

plot(sorbonne_buffer %>% st_geometry())  # disque de 1 km
```

#### c. Filtrage des cinémas dans le buffer

**Avec sf :**

```r
cinema_1km_list <- st_within(cinemas_bpe, sorbonne_buffer)
cinema_1km_sorbonne <- cinemas_bpe %>% filter(lengths(cinema_1km_list) > 0)
# 21 cinémas

# Ou avec st_intersection
cinema_1km_sorbonne <- st_intersection(sorbonne_buffer, cinemas_bpe)
```

**Avec PostGIS :**

```r
crs_bpe <- pull(dbGetQuery(conn, "SELECT Find_SRID('public','bpe21_metro','geometry');"))
# 2154

sorbonne       <- "ST_GeomFromText('POINT(2.34297 48.84864)', 4326)"
sorbonne       <- paste0("ST_Transform(", sorbonne, ", 2154)")
sorbonne_buf   <- paste0("ST_Buffer(", sorbonne, ", 1000)")

query <- paste0(
  "SELECT bpe.* FROM bpe21_metro as bpe, ", sorbonne_buf, " AS sorbuff ",
  "WHERE ST_Within(bpe.geometry, sorbuff.geometry) AND TYPEQU='F303';"
)
cinema_1km_sorbonne <- sf::st_read(conn, query = query)
# 21 cinémas
```

#### d. Vérification avec leaflet

```r
leaflet() %>%
  setView(lat = 48.84864, lng = 2.34297, zoom = 15) %>%
  addTiles() %>%
  addMarkers(lat = 48.84864, lng = 2.34297) %>%
  addCircles(lat = 48.84864, lng = 2.34297, weight = 1, radius = 1000) %>%
  addMarkers(data = cinema_1km_sorbonne %>% st_transform(4326))
```

> Remarque : 1000 m en Lambert-93 ≠ 1000 m en WGS84 (légère différence due à la projection).

---

## Exercice 4 — Problèmes de géolocalisation : les boulodromes de PACA

### 1. Extraction des boulodromes et de la région PACA

```r
paca <- st_read(conn, query = "SELECT * FROM regions_metro WHERE code = '93';")

boulodromes <- st_read(conn,
  query = "SELECT id, typequ, geometry FROM bpe21_metro WHERE typequ = 'F102';")
# 21 187 boulodromes
```

### 2. Intersection géométrique — boulodromes en PACA

**Avec sf :**

```r
boulodromes_paca_list <- st_contains(paca, boulodromes)
boulodromes_paca <- boulodromes %>% slice(boulodromes_paca_list[[1]])

plot(paca %>% st_geometry())
plot(boulodromes_paca %>% st_geometry(), pch = 3, cex = 0.8, add = TRUE)
```

**Avec PostGIS :**

```r
query <- "
  SELECT bpe.* FROM bpe21_metro AS BPE, regions_metro AS regions
  WHERE ST_Contains(regions.geometry, bpe.geometry)
    AND bpe.typequ = 'F102'
    AND regions.code = '93';
"
boulodromes_paca <- st_read(conn, query = query)
```

> La méthode PostGIS est préférable pour les objets trop lourds à charger en mémoire.

### 3. Comparaison avec filtrage par département

#### a. Filtrage direct par département

```r
boulodromes_paca_bis <- st_read(conn,
  query = paste0(
    "SELECT id, typequ, dep, qualite_xy, geometry FROM bpe21_metro ",
    "WHERE typequ = 'F102' AND dep IN ('04','05','06','13','83','84');"
  )
)
```

#### b. Identification des différences

```r
diff <- boulodromes_paca_bis %>% mutate(version_bis = TRUE) %>%
  st_join(boulodromes_paca %>% mutate(version_orig = TRUE) %>% select(-typequ), by = "id") %>%
  filter(is.na(version_bis) | is.na(version_orig))
# 6 boulodromes supplémentaires dans la version filtrée par département
```

**Causes identifiées :**

- Le polygône PACA simplifie le tracé réel → l'extrémité de Port-de-Bouc est hors du polygône
- Les îles du Frioul ne sont pas incluses dans le polygône PACA
- 3 boulodromes géolocalisés en pleine mer (erreurs de géolocalisation)
- 1 boulodrome en limite de polygône

```r
leaflet() %>%
  setView(lat = 43.8741, lng = 6.0287, zoom = 8) %>%
  addTiles() %>%
  addMarkers(data = diff %>% st_transform(4326)) %>%
  addPolygons(data = paca %>% st_transform(4326), stroke = 1, color = "red")
```

> **Conclusion** : les fonds simplifiés peuvent introduire des erreurs d'appartenance géographique. La géolocalisation des équipements n'est pas toujours parfaite.

---

## Exercice 5 (Optionnel) — Courbes isochrones

### 1. Sélection et récupération des coordonnées

```r
mater <- sf::st_read(conn, query = "SELECT * FROM bpe21_metro WHERE TYPEQU='D107';") %>%
  slice(1)

mater_coords <- st_coordinates(mater %>% st_transform(4326)) %>% as.numeric
```

### 2. Localisation sur leaflet

```r
leaflet() %>%
  setView(lng = mater_coords[1], lat = mater_coords[2], zoom = 14) %>%
  addTiles() %>%
  addMarkers(lng = mater_coords[1], lat = mater_coords[2])
```

### 3. Calcul des isochrones

```r
iso <- osrm::osrmIsochrone(
  loc    = mater_coords,
  breaks = seq(0, 60, 10),  # en minutes
  res    = 100
)
# Classes 'sf' and 'data.frame': 6 obs. — colonnes : id, isomin, isomax, geometry
```

### 4. Représentation

```r
bks  <- sort(unique(c(iso$isomin, iso$isomax)))
pals <- hcl.colors(n = length(bks) - 1, palette = "Red-Blue", rev = TRUE)

plot(iso["isomax"], breaks = bks, pal = pals,
     main = "Isochrones (in minutes)", reset = FALSE)
points(x = mater_coords[1], y = mater_coords[2], pch = 4, lwd = 2, cex = 1.5)

# Carte leaflet interactive
leaflet() %>%
  setView(lng = mater_coords[1], lat = mater_coords[2], zoom = 8) %>%
  addTiles() %>%
  addMarkers(lng = mater_coords[1], lat = mater_coords[2]) %>%
  addProviderTiles(providers$CartoDB.DarkMatter,
                   options = providerTileOptions(opacity = 0.4)) %>%
  addPolygons(
    data         = iso,
    fillColor    = pals,
    smoothFactor = 0.3,
    weight       = 1,
    opacity      = 1,
    color        = "white",
    dashArray    = "3",
    fillOpacity  = 0.65
  ) %>%
  addLegend(
    position = "bottomleft",
    colors   = pals,
    labels   = rev(c("50-60", "40-50", "30-40", "20-30", "10-20", "0-10")),
    opacity  = 0.6,
    title    = "Temps de trajet par la route (en minutes)"
  )
```
