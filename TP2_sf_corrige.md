# TP2 - Intro à la statistique spatiale 2A - sf - corrigé

## Objectifs du TP

- Se familiariser à la manipulation d'objets spatiaux à l'aide du package `sf`. ([Cheatsheet](https://r-spatial.github.io/sf/))
- Travailler sur le datalab (plateforme SSP Cloud, service de l'INSEE) : https://datalab.sspcloud.fr
- Les fonds de carte sont disponibles sous `U:/Eleves/Cartographie/Fonds_carte` (ne jamais en faire votre répertoire de travail).

---

## Exercice 1

### 0. Initialisation

Créer un repo GitHub `tp_stats_spatiales`, un service RStudio sur le datalab en saisissant le lien HTTPS du repo Git, puis créer un projet pointant vers le dossier `tp_stats_spatiales`.

Charger les packages :

```r
library(sf)
library(dplyr)
```

---

### 1. Importer le fond communal

```r
commune_francemetro_2021 <- st_read("commune_francemetro_2021.gpkg")
```

```
Reading layer `commune_francemetro_2021' from data source ...
Simple feature collection with 34836 features and 16 fields
Geometry type: MULTIPOLYGON
Dimension: XY
Bounding box: xmin: 99225.97 ymin: 6049647 xmax: 1242375 ymax: 7110480
Projected CRS: RGF93 v1 / Lambert-93
```

> - **Geometry** : MULTIPOLYGON
> - **Dimension XY** : espace euclidien 2D
> - **Bounding box** : coordonnées du cadre contenant les objets spatiaux
> - **Projected CRS** : système de projection (code EPSG)

---

### 2. Résumé de l'objet

```r
str(commune_francemetro_2021)
```

```
Classes 'sf' and 'data.frame': 34836 obs. of 17 variables:
$ code     : chr "07038" "07055" ...
$ libelle  : chr "Borne" "Charmes-sur-Rhône" ...
...
$ geom     :sfc_MULTIPOLYGON of length 34836
```

> L'objet est à la fois un objet `sf` ET un `data.frame`.

---

### 3. Afficher les 10 premières lignes

```r
head(commune_francemetro_2021, 10)
```

> La dernière colonne `geom` contient la géométrie (coordonnées géographiques sous forme MULTIPOLYGON).

---

### 4. Système de projection

```r
st_crs(commune_francemetro_2021)
```

> Renvoie les informations détaillées sur le CRS : libellé (`RGF93 v1 / Lambert-93`), unité (mètre), code EPSG (2154), etc.

---

### 5. Créer la table des communes bretonnes

```r
communes_Bretagne <- commune_francemetro_2021 %>%
  filter(reg == "53") %>%
  select(code, libelle, epc, dep, surf)
```

> La variable `geom` (géométrie) est conservée automatiquement même si elle n'est pas sélectionnée explicitement — c'est le comportement standard de `sf`.

---

### 6. Vérifier que l'objet est toujours un sf

```r
str(communes_Bretagne)
```

```
Classes 'sf' and 'data.frame': 1208 obs. of 6 variables:
```

---

### 7. Plot de la table

```r
plot(communes_Bretagne, lwd = 0.1)
```

---

### 8. Plot avec `st_geometry()`

```r
plot(st_geometry(communes_Bretagne), lwd = 0.5)
```

---

### 9. Créer une variable de surface avec `st_area()`

```r
communes_Bretagne <- communes_Bretagne %>%
  mutate(surf2 = st_area(geom))

str(communes_Bretagne$surf2)
# Units: [m^2] num [1:1208] 24537548 12018097 ...
```

> La variable est en **m²**.

---

### 10. Convertir en km²

```r
communes_Bretagne <- communes_Bretagne %>%
  mutate(surf2 = units::set_units(surf2, km * km))

str(communes_Bretagne$surf2)
# Units: [km^2] num [1:1208] 24.54 12.02 6.25 ...
```

---

### 11. `surf` vs `surf2` — sont-elles égales ?

Non. Plusieurs raisons possibles :
- Calcul sur des systèmes de projection différents (pas le cas ici)
- Fonds différents : les fonds simplifiés ont une précision de contour moindre que les fonds détaillés (ex : BDTOPO)

---

### 12. Table départementale avec `summarise()`

```r
dept_bretagne2 <- communes_Bretagne %>%
  group_by(dep) %>%
  summarise(surf = sum(surf))

plot(st_geometry(dept_bretagne2))
```

> Les géométries sont regroupées pour produire une géométrie par département.

---

### 13. Fond départemental avec `st_union()`

```r
dept_bretagne <- communes_Bretagne %>%
  group_by(dep) %>%
  summarise(geometry = st_union(geom))

plot(st_geometry(dept_bretagne), axes = TRUE)
```

> C'est la méthode à préconiser pour regrouper des géométries, surtout sans variable numérique disponible.

---

### 14. Centroïdes des départements bretons

#### a. Type de géométrie

```r
centroid_dept_bret <- st_centroid(dept_bretagne)
# Warning: st_centroid assumes attributes are constant over geometries

class(centroid_dept_bret$geometry)
# [1] "sfc_POINT" "sfc"
```

> Type : **POINT**

#### b. Représentation sur carte

```r
plot(st_geometry(dept_bretagne))
plot(st_geometry(centroid_dept_bret), add = TRUE)
```

#### c. Ajouter le nom du département

```r
dept_lib <- tibble(
  dep    = c("22", "29", "35", "56"),
  dep_lib = c("Côtes d'Armor", "Finistère", "Ille-et-Vilaine", "Morbihan")
)

centroid_dept_bret <- centroid_dept_bret %>%
  left_join(dept_lib, by = "dep")
```

#### d. Récupérer les coordonnées des centroïdes

```r
centroid_coords <- st_coordinates(centroid_dept_bret)
# Résultat : matrice avec colonnes X et Y uniquement

centroid_coords <- centroid_coords %>%
  bind_cols(
    centroid_dept_bret %>%
      select(dep, dep_lib) %>%
      st_drop_geometry()
  )
```

#### e. Carte avec noms des départements

```r
plot(st_geometry(dept_bretagne))
plot(st_geometry(centroid_dept_bret), pch = 16, col = "orangered", add = TRUE)
text(
  x      = centroid_coords$X,
  y      = centroid_coords$Y,
  labels = centroid_coords$dep_lib,
  pos    = 3,
  cex    = 0.8,
  col    = "orangered"
)
```

---

### 15. Commune contenant chaque centroïde — `st_intersects()`

```r
commune_centroid_bret <- st_intersects(communes_Bretagne, centroid_dept_bret)
# typeof : "list"

which(lengths(commune_centroid_bret) > 0)
# [1] 148 476 647 1092

# Méthode dplyr
commune_centre_dep <- communes_Bretagne %>%
  filter(lengths(commune_centroid_bret) > 0)

# Méthode inverse (centroïdes en 1er paramètre — plus simple)
commune_centroid_bret2 <- st_intersects(centroid_dept_bret, communes_Bretagne)
commune_centre_dep <- communes_Bretagne[unlist(commune_centroid_bret2), ]

# Ou avec les crochets et op=
communes_Bretagne[centroid_dept_bret, , op = st_intersects]
```

Résultat :

| code  | libelle      | dep | surf  |
|-------|--------------|-----|-------|
| 22170 | Plaine-Haute | 22  | 15.91 |
| 29139 | Lopérec      | 29  | 40.03 |
| 35024 | Betton       | 35  | 26.80 |
| 56141 | Moustoir-Ac  | 56  | 33.93 |

---

### 16. `st_intersection()` et `st_within()`

```r
Q16_intersection <- st_intersection(communes_Bretagne, centroid_dept_bret)
# Renvoie directement un data.frame — plus pratique, mais plus lent sur gros volumes

Q16_within <- st_within(centroid_dept_bret, communes_Bretagne)
# Renvoie une liste. L'ordre des paramètres compte.
```

---

### 17. Distance centroïdes — chefs-lieux

```r
Q17 <- st_distance(
  centroid_dept_bret,
  communes_Bretagne %>%
    filter(libelle %in% c("Saint-Brieuc", "Quimper", "Rennes", "Vannes"))
)

Q17 <- data.frame(Q17)
rownames(Q17) <- centroid_dept_bret$dep
colnames(Q17) <- c("Saint-Brieuc", "Quimper", "Rennes", "Vannes")
```

---

### 18. Communes à moins de 20 km de chaque centroïde

#### a. Buffer de 20 km

```r
buffer <- st_buffer(commune_centre_dep, dist = 20000)
```

#### b. Représentation

```r
plot(st_geometry(dept_bretagne))
plot(st_geometry(centroid_dept_bret), add = TRUE)
plot(st_geometry(buffer), add = TRUE)
```

#### c. Intersection

```r
Q18 <- buffer %>%
  rename(code_centroid = code, dep_centroid = dep) %>%
  st_intersection(
    communes_Bretagne %>% select(code_intersect = code, dep_intersect = dep)
  )
```

#### d. Nombre de communes par département

```r
Q18 %>%
  st_drop_geometry() %>%
  filter(dep_centroid == dep_intersect) %>%
  group_by(dep_centroid) %>%
  count()
```

| dep_centroid | n   |
|--------------|-----|
| 22           | 93  |
| 29           | 88  |
| 35           | 113 |
| 56           | 71  |

---

### 19. Changement de projection — WGS84

#### a. Reprojection

```r
communes_Bretagne_wgs84 <- st_transform(communes_Bretagne, 4326)
```

#### b. Représentation comparative

```r
par(mfrow = c(1, 2))
plot(st_geometry(communes_Bretagne),      main = "epsg:2154")
plot(st_geometry(communes_Bretagne_wgs84), main = "epsg:4326")
```

---

### 20. Recalcul de la surface en WGS84

```r
communes_Bretagne_wgs84 <- communes_Bretagne_wgs84 %>%
  mutate(surf3 = st_area(geom))
```

> En WGS84, l'unité est le **degré**. Sur les anciennes versions de `sf`, le calcul de distances/aires ne peut pas se faire directement. Depuis `sf >= 1.0.6`, le calcul fonctionne mais les valeurs diffèrent légèrement de `surf2`.
