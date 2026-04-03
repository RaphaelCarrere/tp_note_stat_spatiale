# TP3 - Intro à la statistique spatiale 2A - Carto avec R - corrigé

## Objectifs du TP

- Se sensibiliser à la **discrétisation de variables**
- Découvrir les packages de cartographie `mapsf` (cartes statiques) et `leaflet` (cartes dynamiques)

Cheatsheets : [mapsf](https://riatelab.github.io/mapsf/) | [leaflet](https://rstudio.github.io/leaflet/)

```r
library(dplyr)
library(sf)
library(mapsf)
library(classInt)
library(leaflet)
library(openxlsx)
```

---

## Exercice 1 — Discrétisation

### 1. Préparation du jeu de données

```r
# Fond communes France métropolitaine
communes_fm <- st_read("fonds/commune_francemetro_2021.gpkg") %>%
  select(code, libelle, surf)

# Population légale 2019
pop_com_2019 <- openxlsx::read.xlsx("data/Pop_legales_2019.xlsx")

# Correction Paris (arrondissements → commune 75056)
pop_com_2019 <- pop_com_2019 %>%
  mutate(COM = if_else(substr(COM, 1, 3) == "751", "75056", COM)) %>%
  group_by(code = COM) %>%
  summarise(pop = sum(PMUN19))

# Jointure et calcul de densité
communes_fm <- communes_fm %>%
  left_join(pop_com_2019, by = "code") %>%
  mutate(densite = pop / surf)
```

---

### 2. Distribution de la densité

```r
summary(communes_fm$densite)
#    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.
#    0.00   18.30   40.34  163.25   95.72 27306.61

hist(communes_fm$densite)
```

> Très grande majorité de communes avec faible densité. Trois quarts des communes ont moins de 95 hab/km².

---

### 3. Carte choroplèthe sur variable continue

```r
plot(communes_fm["densite"], border = FALSE)
```

> Une représentation directe sur variable continue peut être pertinente si la distribution est équilibrée — ce qui n'est pas le cas ici (France très majoritairement rurale). La carte est donc peu informative.

---

### 4. Comparaison des méthodes de discrétisation

```r
plot(communes_fm["densite"], breaks = "sd",       main = "sd",       border = FALSE)
plot(communes_fm["densite"], breaks = "pretty",   main = "pretty",   border = FALSE)
plot(communes_fm["densite"], breaks = "quantile", main = "quantile", border = FALSE)
plot(communes_fm["densite"], breaks = "jenks",    main = "jenks",    border = FALSE)
```

**Analyse :**

- **sd (écarts-types)** : non adapté aux données très asymétriques. Peu informatif.
- **pretty** : intervalles réguliers et arrondis — clairement inefficace ici.
- **quantile** : chaque classe contient le même nombre d'observations — révèle bien la structuration du territoire mais concentre peut-être trop de communes dans les classes denses.
- **jenks** : optimise la variance inter-classes — adapté, mais produit une classe de très faible densité très volumineuse.

---

### 5. Analyse avec `classInt`

#### a. Discrétisation quantile avec `classIntervals()`

```r
denspop_quant <- classIntervals(
  communes_fm$densite,
  style = "quantile",
  n = 5
)

str(denspop_quant)
denspop_quant$brks
# [1]     0.00    15.35    29.48    55.05   121.07 27306.61
```

> L'objet contient `$var` (la variable de départ) et `$brks` (les bornes des 5 intervalles).

#### b. Visualisation avec palette

```r
pal1 <- RColorBrewer::brewer.pal(n = 5, name = "YlOrRd")
plot(denspop_quant, pal = pal1, main = "quantile")
```

#### c. Comparaison des méthodes

```r
analyser_discret <- function(method, nb_classes) {
  denspop_c <- classIntervals(communes_fm$densite, style = method, n = nb_classes)
  print(denspop_c$brks)
  plot(denspop_c, pal = pal1, main = method)
  return(denspop_c)
}

all_discret <- sapply(c("quantile", "sd", "pretty", "jenks"), analyser_discret, nb_classes = 5)
```

Bornes obtenues :

| Méthode   | Bornes                                        |
|-----------|-----------------------------------------------|
| quantile  | 0, 15.35, 29.48, 55.05, 121.07, 27306.61     |
| sd        | -7249.99, 163.25, 7576.48, 14989.71, 22402.95, 29816.18 |
| pretty    | 0, 5000, 10000, 15000, 20000, 25000, 30000   |
| jenks     | 0, 1050.25, 3984.77, 8499.08, 15944.92, 27306.61 |

Quantiles déciles utiles :

```r
quantile(communes_fm$densite, probs = seq(0, 1, 0.1))
summary(communes_fm$densite)
# Médiane = 40, Moyenne = 162
```

#### d. Découpage manuel — 5 classes

```r
denspop_man_brks5 <- c(0, 40, 162, 1000, 8000, 27200)

popcomfm_sf <- communes_fm %>%
  mutate(
    densite_c = cut(
      densite,
      breaks          = denspop_man_brks5,
      include.lowest  = TRUE,
      right           = FALSE,
      ordered_result  = TRUE
    )
  )
```

#### e. Carte avec palette personnalisée

```r
table(popcomfm_sf$densite_c)
# [0,40)        [40,162)      [162,1e+03)   [1e+03,8e+03)  [8e+03,2.72e+04]
#  17327          12248           4371            827              62

pal2 <- c(
  RColorBrewer::brewer.pal(n = 5, name = "Greens")[4:3],
  RColorBrewer::brewer.pal(n = 5, name = "YlOrRd")[c(2, 4:5)]
)

plot(
  popcomfm_sf["densite_c"],
  pal    = pal2,
  border = FALSE,
  main   = "Densité de population"
)
```

---

## Exercice 2 — Carte choroplèthe avec `mapsf`

### 1. Import et jointure

```r
dep_francemetro_2021 <- st_read("fonds/dep_francemetro_2021.gpkg")
tx_pauvrete         <- read.xlsx("data/Taux_pauvrete_2018.xlsx")
mer                 <- st_read("fonds/merf_2021.gpkg")

dep_francemetro_2021_pauv <- dep_francemetro_2021 %>%
  left_join(tx_pauvrete %>% select(-Dept), by = c("code" = "Code"))
```

### 2. Cartes basiques

```r
# Méthode Jenks
mf_map(x = dep_francemetro_2021_pauv, var = "Tx_pauvrete", type = "choro",
       nbreaks = 4, breaks = "jenks")

# Méthode equal
mf_map(x = dep_francemetro_2021_pauv, var = "Tx_pauvrete", type = "choro",
       nbreaks = 4, breaks = "equal")

# Méthode quantile
mf_map(x = dep_francemetro_2021_pauv, var = "Tx_pauvrete", type = "choro",
       nbreaks = 4, breaks = "quantile")
```

### 3. Carte avec découpage manuel + zoom Paris

```r
couleur <- rev(mf_get_pal(4, "Mint"))

mf_map(
  x       = dep_francemetro_2021_pauv,
  var     = "Tx_pauvrete",
  type    = "choro",
  breaks  = c(0, 13, 17, 25, max(dep_francemetro_2021_pauv$Tx_pauvrete)),
  pal     = couleur,
  leg_pos = NA
)

# Encadré zoom Paris petite couronne
mf_inset_on(x = dep_francemetro_2021_pauv, pos = "topright", cex = .2)
mf_init(dep_francemetro_2021_pauv %>% filter(code %in% c("75", "92", "93", "94")))
mf_map(
  dep_francemetro_2021_pauv %>% filter(code %in% c("75", "92", "93", "94")),
  var     = "Tx_pauvrete",
  type    = "choro",
  breaks  = c(0, 13, 17, 25, max(dep_francemetro_2021_pauv$Tx_pauvrete)),
  leg_pos = NA,
  add     = TRUE
)
mf_label(dep_francemetro_2021_pauv %>% filter(code %in% c("75", "92", "93", "94")),
         var = "code", col = "black")
mf_inset_off()

# Légende personnalisée
mf_legend(
  type  = "choro",
  title = "Taux de pauvreté",
  val   = c("", "Moins de 13", "De 13 à moins de 17", "De 17 à moins de 25", "25 ou plus"),
  pal   = couleur,
  pos   = "left"
)

mf_map(mer, add = TRUE)
mf_layout(title = "Taux de pauvreté par département en 2018", credits = "Source : Insee")
```

### 4. Sauvegarder en GeoPackage

```r
st_write(dep_francemetro_2021_pauv, "dept_tx_pauvrete_2018.gpkg", delete_layer = TRUE)
```

---

## Exercice 3 — Carte `prop_choro` (ronds proportionnels + choroplèthe)

```r
region <- st_read("fonds/reg_francemetro_2021.gpkg")
pop_reg <- openxlsx::read.xlsx("data/pop_region_2019.xlsx")
mer     <- st_read("fonds/merf_2021.gpkg")

region_pop <- region %>%
  left_join(pop_reg, by = c("code" = "reg")) %>%
  mutate(densite = pop / surf)

mf_map(region_pop)
mf_map(mer, add = TRUE, col = "darkslategray3")

mf_map(
  region_pop,
  var        = c("pop", "densite"),
  nbreaks    = 3,
  breaks     = "fisher",
  leg_val_rnd = c(-2, 0),
  leg_title  = c("Population", "Nombre d'habitants au km²"),
  type       = "prop_choro"
)

mf_layout(title = "Densité et population des régions françaises en 2018",
          credits = "Source : Insee")
```

---

## Exercice 4 — Carte interactive avec `leaflet`

### 1. Import des données

```r
comfm_sf <- st_read("fonds/commune_francemetro_2021.gpkg")

bpesl_tb <- read.csv("data/bpe20_sport_loisir_xy.csv", sep = ";", dec = ".")
```

### 2. Fond OpenStreetMap centré sur la France

```r
map <- leaflet() %>%
  setView(lng = 2.6, lat = 47.2, zoom = 5) %>%
  addTiles()
map
```

### 3. Localisation des bowlings

#### a. Filtrage

```r
bowlings <- bpesl_tb %>%
  filter(TYPEQU == "F119" & REG > 10 & !is.na(LAMBERT_X) & !is.na(LAMBERT_Y))
```

#### b. Conversion en objet sf + reprojection WGS84

```r
bowlings_sf <- bowlings %>%
  st_as_sf(coords = c("LAMBERT_X", "LAMBERT_Y"), crs = 2154) %>%
  st_transform(crs = 4326)
```

#### c. Ajout des marqueurs

```r
map %>% addMarkers(data = bowlings_sf)
```

#### d. Avec popup

```r
map %>% addMarkers(data = bowlings_sf, popup = ~DEPCOM)
```

---

### 4. Carte choroplèthe interactive (ex. département 40 — Landes)

#### a. Initialisation

```r
map2 <- leaflet() %>%
  setView(lng = -0.5, lat = 44, zoom = 8) %>%
  addTiles()
```

#### b. Import et calcul

```r
popcom_tb <- read.csv("data/base-cc-evol-struct-pop-2018_echantillon.csv",
                      sep = ",", dec = ".") %>%
  filter(stringr::str_detect(CODGEO, "^40")) %>%
  select(CODGEO, P18_POP, P18_POP0014) %>%
  mutate(PART18_POP0014 = P18_POP0014 / P18_POP * 100)
```

#### c. Jointure + reprojection

```r
popcomd40_sf <- comfm_sf %>%
  right_join(popcom_tb, by = c("code" = "CODGEO")) %>%
  st_transform(crs = 4326)
```

#### d. Polygones sans remplissage

```r
map2 %>%
  addPolygons(data = popcomd40_sf, weight = 0.5, color = "purple", fill = NA)
```

#### e. Palette de couleurs

```r
pal <- colorQuantile("Blues", domain = popcomd40_sf$PART18_POP0014, n = 5)
```

#### f. Carte choroplèthe

```r
map_choro <- map2 %>%
  addPolygons(
    data        = popcomd40_sf,
    weight      = 0.5,
    color       = "purple",
    fillOpacity = 0.5,
    fillColor   = ~pal(PART18_POP0014)
  )
```

#### g. Légende

```r
map_choro_leg <- map_choro %>%
  addLegend(pal = pal, values = popcomd40_sf$PART18_POP0014)
```

#### h. Popup avec informations communes

```r
popcomd40_sf$contenu_popup <- paste0(
  "<b>", popcomd40_sf$libelle, "</b>",
  "<br>Population : ", format(popcomd40_sf$P18_POP, big.mark = " "),
  "<br>Part moins de 15 ans : ", round(popcomd40_sf$PART18_POP0014, 1), "%"
)

map_choro_popup <- map2 %>%
  addPolygons(
    data        = popcomd40_sf,
    weight      = 0.5,
    color       = "purple",
    fillOpacity = 0.5,
    fillColor   = ~pal(PART18_POP0014),
    popup       = ~contenu_popup
  ) %>%
  addLegend(pal = pal, values = popcomd40_sf$PART18_POP0014)
```

#### i. Bassins de natation

```r
natation_d40 <- bpesl_tb %>%
  filter(TYPEQU == "F101" & DEP == "40" & !is.na(LAMBERT_X) & !is.na(LAMBERT_Y))

natation_sf <- natation_d40 %>%
  st_as_sf(coords = c("LAMBERT_X", "LAMBERT_Y"), crs = 2154) %>%
  st_transform(crs = 4326)

map_choro_popup %>%
  addMarkers(data = natation_sf, label = ~as.character(DEPCOM))
```

#### j. Contrôle des couches

```r
leaflet() %>%
  setView(lng = -0.5, lat = 44, zoom = 8) %>%
  addTiles() %>%
  addPolygons(data = popcomd40_sf, weight = 0.5, color = "purple", fill = NA,
              group = "limites communales") %>%
  addPolygons(data = popcomd40_sf, weight = 0.5, color = NA,
              fillOpacity = 0.5, fillColor = ~pal(PART18_POP0014),
              popup = ~contenu_popup, group = "Analyse thématique") %>%
  addLegend(pal = pal, values = popcomd40_sf$PART18_POP0014) %>%
  addMarkers(data = natation_sf, label = ~as.character(DEPCOM),
             group = "Bassins de natation") %>%
  addLayersControl(
    overlayGroups = c("limites communales", "Analyse thématique", "Bassins de natation")
  )
```

---

## Bonus — `mapview`

```r
library(mapview)

qt_part <- quantile(popcomd40_sf$PART18_POP0014, probs = seq(0, 1, .2))
qt_part_arr <- c(floor(qt_part[1]*10)/10, round(qt_part[2:5], 1), ceiling(qt_part[6]*10)/10)

mapview(
  popcomd40_sf,
  z            = "PART18_POP0014",
  at           = qt_part_arr,
  alpha.regions = 0.35,
  layer.name   = "Moins de 14 ans (en %)",
  label        = "libelle",
  popup        = leafpop::popupTable(popcomd40_sf, z = c("code", "libelle", "P18_POP", "PART18_POP0014"))
) +
mapview(
  natation_sf %>%
    mutate(COUVERT = factor(COUVERT, levels = 0:1, labels = c("Non couverte", "Couverte"))),
  z            = "COUVERT",
  col.regions  = c("steelblue", "coral"),
  alpha        = 0.8,
  alpha.regions = 0.7,
  layer.name   = "Piscines",
  popup        = leafpop::popupTable(natation_sf, z = "NB_AIREJEU")
)
```

---

## Commentaires — Rappels sur la discrétisation

**Principales méthodes :**

- **Quantiles** : chaque classe contient le même nombre d'observations
- **Seuils naturels** (k-means, Jenks, Fisher) : optimisent le rapprochement des valeurs similaires — classes les plus différentes possibles entre elles
- **Écarts-types** : adapté aux distributions proches de la loi normale
- **Seuils manuels** : pour affiner une discrétisation automatique
