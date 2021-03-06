TITLE
================

<details>

<summary>Codes</summary> <br>

``` r
want = c("dplyr",
         "foreign",
        "ggplot2",
        "forcats",
        "survey",
        "gtsummary")

have = want %in% rownames(installed.packages())

# Install the packages that we miss
if ( any(!have) ) { install.packages( want[!have] ) }

# Load the packages
junk <- lapply(want, library, character.only = T)

# Remove the objects we created
rm(have, want, junk)
```

</details>

# Introduction

``` r
unzip("k_deploc.zip")

Dpl.Rawdata = read.dta("k_deploc.dta") %>%
  select(v2_mdisttot,v2_mmoy1s:v2_mmoy4s, ident_ind, v2_date_dep, poids_jour, v2_mtp, pondki, v2_mori1motif)

unzip("Q_INDIVIDU.dta")

ENTD.INDIVIDU.rawdata = read.dta("Q_INDIVIDU.dta") %>% 
  select(ident_men, ident_ind,sexe,age, noi)

Motifs = read.csv("Motifs_deplacement.csv", sep = ";")

Dpl.data = Dpl.Rawdata  %>% 
  merge(ENTD.INDIVIDU.rawdata, by = "ident_ind") %>% 
  merge(Motifs, by = "v2_mori1motif", all.x = T)
```

# Weighting of the survey

``` r
Dpl.data.Ind <- Dpl.data %>%
  distinct(ident_ind, .keep_all = TRUE) %>%
  mutate(sexe = ifelse(sexe == 1, "male", "female"))

dw <- svydesign(ids = ~ident_ind,
                data = Dpl.data.Ind,
                weights =  ~pondki)

ggplot(dw$variables) +
  aes(weight = weights(dw), x = age, group = sexe,  fill = sexe) + 
  geom_histogram(aes(color = sexe, fill = sexe),
                         alpha = 0.4, position = "identity", stat = "count") +
  scale_fill_manual(values = c("#00AFBB", "#E7B800")) +
  scale_color_manual(values = c("#00AFBB", "#E7B800"))+
  theme_minimal() +
  ylab("") +
  xlab("age") +
  theme(legend.position="top") + 
  labs(title = "Weighted")
```

<img src="EGT_files/figure-gfm/unnamed-chunk-3-1.png" style="display: block; margin: auto;" />

``` r
ggplot(dw$variables) +
  aes( x = age, group = sexe,  fill = sexe) + 
  geom_histogram(aes(color = sexe, fill = sexe),
                         alpha = 0.4, position = "identity", stat = "count") +
  scale_fill_manual(values = c("#00AFBB", "#E7B800")) +
  scale_color_manual(values = c("#00AFBB", "#E7B800"))+
  theme_minimal() +
  ylab("") +
  xlab("age") +
  theme(legend.position="top") + 
  labs(title = "Not Weighted")
```

<img src="EGT_files/figure-gfm/unnamed-chunk-3-2.png" style="display: block; margin: auto;" />

``` r
Dpl.data.active =  Dpl.data %>% 
  filter(v2_mtp ==1.10 | v2_mtp == 2.2) %>% 
  mutate(age_grp.FACTOR = cut( age, breaks = seq(0,150, by = 5), include.lowest = T, right = F),
         age_grp = as.character(age_grp.FACTOR), 
         age_grp = gsub("\\[|\\]|\\(|\\)", "", age_grp),
         age_grp = gsub(",", "-", age_grp),
         post = sub(".*-","",age_grp),
         age_grp = sub("-.*", "", age_grp),
         age_grp = paste0(age_grp,"-", as.numeric(post)-1),
         order = as.numeric(substr(age_grp,1,regexpr("-",age_grp)-1)),
         poids_jour = poids_jour/7,
         Mode = ifelse(v2_mtp == 1.1, "Walking", "Cycling"),
         distance = v2_mdisttot * poids_jour) %>%
  group_by(age_grp, sexe, Mode, v2_mtp, order) %>%
  summarise(distance = sum(distance, na.rm = T)) %>%
  ungroup() %>% 
  group_by(Mode) %>% 
  mutate(freq = distance/sum(distance))

saveRDS(Dpl.data.active, "Dpl.data.active.rds")
```

# Visualization of the distribution

## Cycling

<img src="EGT_files/figure-gfm/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

## Walking

<img src="EGT_files/figure-gfm/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

# reference
