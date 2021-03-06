Population Projection
================

<details>

<summary>Packages</summary>

<p>

``` r
#Rtools is necessary to run this chunk of codes
#You can download it here: https://cran.r-project.org/bin/windows/Rtools/

want = c("dplyr","readxl", "ggpol","tidyr","forcats")

have = want %in% rownames(installed.packages())

# Install the packages that we miss
if ( any(!have) ) { install.packages( want[!have]) }

# Load the packages
junk <- lapply(want, library, character.only = T)

# Remove the objects we created
rm(have, want, junk)


devtools::install_github('thomasp85/gganimate')
library(gganimate)

devtools::install_github("r-rust/gifski")
```

    ##          checking for file 'C:\Users\Pierre\AppData\Local\Temp\Rtmpe02w5E\remotes6a9422343933\r-rust-gifski-6b86cc6/DESCRIPTION' ...     checking for file 'C:\Users\Pierre\AppData\Local\Temp\Rtmpe02w5E\remotes6a9422343933\r-rust-gifski-6b86cc6/DESCRIPTION' ...   v  checking for file 'C:\Users\Pierre\AppData\Local\Temp\Rtmpe02w5E\remotes6a9422343933\r-rust-gifski-6b86cc6/DESCRIPTION' (1s)
    ##       -  preparing 'gifski': (425ms)
    ##      checking DESCRIPTION meta-information ...     checking DESCRIPTION meta-information ...   v  checking DESCRIPTION meta-information
    ##   -  cleaning src
    ##       -  checking for LF line-endings in source and make files and shell scripts (646ms)
    ##       -  checking for empty or unneeded directories
    ##       -  building 'gifski_1.4.3-1.tar.gz'
    ##   Avis :     Avis : file 'gifski/configure' did not have execute permissions: corrected
    ##      
    ## 

``` r
library(gifski)
```

</details>

# Introduction

To quantify the impact on the long term of walking and cycling on
individuals; we have to understand how the population will evolve by the
horizon of 2050. To do so, we use the scenario of low fecundity 2007
produced by the *Institut national de la statistique et des études
économiques* (INSEE)(**???**). This is the same demographic scenario
adopted by the negaWatt team. The scenario only covers the mainland of
France (*France métropolitaine*).

<details>

<summary>Codes</summary>

<p>

``` r
temp <-  tempfile()
dataURL <- "https://www.insee.fr/fr/statistiques/pyramide/2418122/excel/projpop0760_FECbasESPcentMIGcent.xls"
download.file(dataURL, destfile=temp, mode='wb')

unz(temp, "projpop0760_FECbasESPcentMIGcent.xls")
```

    ## A connection with                                                                                                                        
    ## description "C:\\Users\\Pierre\\AppData\\Local\\Temp\\Rtmpe02w5E\\file6a945a66561d:projpop0760_FECbasESPcentMIGcent.xls"
    ## class       "unz"                                                                                                       
    ## mode        "r"                                                                                                         
    ## text        "text"                                                                                                      
    ## opened      "closed"                                                                                                    
    ## can read    "yes"                                                                                                       
    ## can write   "yes"

``` r
Pop.1 <- readxl::read_excel(temp, sheet = "populationH", skip = 4, col_names = TRUE)
Pop.2 <- readxl::read_excel(temp, sheet = "populationF", skip = 4, col_names = TRUE)

unlink(temp)
```

``` r
Pop.H = Pop.1 %>% 
  rename( age = "Âge au 1er janvier") %>%
  mutate(age = as.numeric(age),
         sexe = "Male") %>%
  pivot_longer(!c(age, sexe),names_to = "year",values_to = "Pop" )

Pop.F = Pop.2 %>% 
  rename( age = "Âge au 1er janvier") %>%
  mutate(age = as.numeric(age),
         sexe = "Female") %>%
  pivot_longer(!c(age, sexe),names_to = "year",values_to = "Pop" )

Pop.proj = rbind(Pop.H, Pop.F) %>%
  mutate(year = as.numeric(year))

Pop.proj.cat = Pop.proj %>% 
  mutate(age_grp.FACTOR = cut( age, breaks = seq(0,150, by = 5), include.lowest = T),
         age_grp = as.character(age_grp.FACTOR), 
         age_grp = gsub("\\[|\\]|\\(|\\)", "", age_grp),
         age_grp = gsub(",", "-", age_grp),
         order = as.numeric(substr(age_grp,1,regexpr("-",age_grp)-1))) %>% 
  na.omit()

saveRDS(Pop.proj.cat, "Data/Pop.proj.rds")

PopPyramid <- Pop.proj.cat %>% 
  group_by(sexe, year, age_grp, order) %>% 
  summarise(Pop = sum(Pop))%>% 
  mutate(Pop = ifelse(sexe == 'Female', as.integer(Pop * -1), as.integer(Pop)),
         sexe = as.factor(sexe)) %>% 
 ggplot(aes(x = fct_reorder(age_grp, order),
            y = Pop/1000,
            fill = sexe)) +
 geom_bar(stat = "identity") +
 scale_fill_manual(values = c("#ee7989" ,"#4682b4")) + 
 coord_flip() + 
 facet_share(~ sexe,
             dir = "h",
             scales = "free",
             reverse_num = TRUE)

PopPyramid <-  PopPyramid +
  theme(panel.background = element_blank(),
       panel.border = element_blank(),
          panel.grid.major = element_blank(),
          panel.grid.minor = element_blank(),
       strip.background = element_blank(),
       strip.text.x = element_blank(),
        axis.title.y = element_blank(),
              legend.title = element_blank(),
       axis.text = element_text(size = 14),
        axis.text.x=element_blank(),
        axis.ticks.x=element_blank(),
       legend.key.size = unit(0.75, "cm"),
       legend.text = element_text(size = 15,face = "bold"),
       plot.title = element_text(size = 22,hjust = 0.5,face = "bold"),
       plot.subtitle = element_text(size = 14, hjust = 0.5,face = "bold"),
       axis.title.x = element_text(size = 12,face = "bold"),
       plot.caption = element_text(size = 12, hjust = 0.5,face = "italic",color = "gray"))


PopPyramid <- PopPyramid + 
 labs(
      title = "France Population Projection\nEstimate from 2007 to 2050\n\n{closest_state}",
      subtitle = "\n\nAge Group",
      y = "\n\nPopulation (in thousands)",
      caption = "\n\nData Source: www.insee.fr/fr/statistiques"
     )


PopPyramid <- PopPyramid + 
  transition_states(year,
                    transition_length = 1,
                    state_length = 1) + 
  enter_fade() +
  exit_fade() + 
  ease_aes('cubic-in-out')
```

</details>

<img src="Population_Projection_files/figure-gfm/unnamed-chunk-4-1.gif" style="display: block; margin: auto;" />

As shown by the graph above, we understand that the number of older
individuals will keep increasing over the years.

# reference
