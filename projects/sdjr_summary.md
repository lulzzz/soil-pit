# SDJR and MLRA Summary
Stephen Roecker  
December 12, 2016  





```r
setwd("C:/Users/stephen.roecker/ownCloud/Documents/lims_data")

### Set parameters
ro <- 11
pt <- c("SDJR")
pt2 <- c("SDJR", "MLRA", "INITIAL", "ES")
pt3 <- c("SDJR", "MLRA")

# ## Fetch data from the server
# 
# data(state); st <- tolower(state.abb)
# # ssa <- read.dbf("M:/geodata/soils/soilsa_a_nrcs.dbf")
# # st <- sort(unique(substr(ssa$areasymbol, 1, 2)))
# asymbol <- paste0(st, "%25")
# project_id <- "%25"
# years <- 2012:2016; start_date <- paste0("10/01/", years - 1); finish_date <- paste0("10/01/", years)
# 
# goals <- lapply(years, function(x){
#   cat("fetching", x, "goal\n")
#   goals_report(x, "%25")
#   }); goals <- do.call("rbind", goals)
# 
# # rcors <- mapply(function(years, start_date, finish_date){
#   cat("fetching", years, "reverse correlation\n")
#   sdjr_correlation(asymbol, project_id, start_date, finish_date)
#   }, years, start_date, finish_date, SIMPLIFY = FALSE); rcors <- do.call("rbind", rcors)


## Load data from files

corr_dates <- "2015_09_24"
goal_dates <- "2016_12_13"
corr_files <- paste0("lims_correlation_fy", 2012:2015, "_", corr_dates, ".csv")
goal_files <- paste0("lims_goals_fy", substr(2012:2016, 3, 4), "_", goal_dates, ".csv")

rcors.. <- {lapply(corr_files, read.csv) ->.; do.call("rbind", .)}
rcors16 <- read.csv("lims_correlation_fy2016_AK%25_WY%25_2016_09_19.csv")
goals <- {lapply(goal_files, read.csv) ->.; do.call("rbind", .)}
load(file = "ssa.Rdata"); ssa -> ssa2
# ssa <- read.csv("M:/geodata/soils/SSA_Regional_Ownership_MASTER_MLRA_OFFICE.csv", stringsAsFactors = FALSE)
# ssa_r11 <- ssa[ssa$Region == 11, "AREASYMBOL"]

## House Cleaning

### rcors
old_names <- c("sso", "old_musym", "old_natsym", "new_natsym", "old_mukey", "old_acres", "new_acres", "old_muname", "old_musym2", "spatial")
new_names <- c("office", "musym", "nationalmusym", "new_nationalmusym", "mukey", "muacres", "new_muacres", "muname", "musym_orig", "spatial_change")
names(rcors..) <- tolower(names(rcors..))
names(rcors..)[which(names(rcors..) %in% old_names)] <- new_names
rcors.. <- mutate(rcors..,
                  projecttypename = substr(projectname, 1, 4),
                  state = substr(areasymbol, 1, 2)
                  )
rcors16 <- filter(rcors16, !is.na(new_mukey))
rcors16 <- mutate(rcors16,
                  state = substr(areasymbol, 1, 2)
                  )

match_names <- names(rcors..)[names(rcors..) %in% names(rcors16)]
rcors <- full_join(rcors.., rcors16)
rcors <- rcors[, match_names]


### goals

old_names <- c("ssoffice", "projecttype")
new_names <- c("office", "projecttypename")
names(goals)[which(names(goals) %in% old_names)] <- new_names

### save copy

save(rcors, file = paste0("rcors_fy_summary_", corr_dates, "_", format(Sys.time(), "%Y_%m_%d.Rdata")))
save(goals, file = paste0("goals_fy_summary_", goal_dates, "_", format(Sys.time(), "%Y_%m_%d.Rdata")))
```


## Estimate Workload


```r
fte <- data.frame(
  fte = c(2, 2.5, 3, 2.5, 3, 5, 3, 5, 4, 1, 3),
  office = c("11-ATL", "11-AUR", "11-CLI", "11-FIN", "11-GAL", "11-IND", "11-JUE", "11-MAN", "11-SPR", "11-UNI", "11-WAV")
  ) # these numbers need to be updated to reflect retirements
wdays <- (52*5-10-3*5)*.75 # 52 weeks per year, 5 days per week, minus 10 holidays, minus 3 weeks vaction, at 75% an individuals time (i.e. minus 15% for TSS and 10% for miscellaneous)
# projects per person = 69 projects/ 3.5 fte
# days per project = workingdays/projects per person 

days_p_person <- function(days, projects, fte){
  days/(projects/fte)
}

goals_test <- mutate(goals,
                     mapunit = ifelse(projectname %in% c("^SDJR"), projectname, NA),
                     mapunit = sapply(projectname, function(x) unlist(strsplit(x, " - "))[3]),
                     components = sapply(mapunit, function(x) unlist(strsplit(x, " "))[1])
                     )

test <- goals_test %>%
  filter(region %in% ro) %>%
  group_by(fy = as.character(fy), office) %>%
  summarize(
    acres = sum(reported),
    n_projects = length(unique(projectname)),
    n_mapunits = length(unique(mapunit)),
    n_majors = length(unique(sapply(components, function(x) unlist(strsplit(x, "-"))[1])))
    ) %>% 
  inner_join(fte, by = "office")

test <- mutate(test,
               fte = round(fte, 1),
               days = round(fte*wdays, 0),
               days_project_fte = round(days / (n_projects / fte), 0), days_major_fte = round(days / (n_majors / fte), 0)
               ) %>%
  as.data.frame()
test2 <- rbind(test, c("2012:2016", "Average",
                       round(apply(test[3:9], 2, mean), 0))
               )

kable(test2)

# ggplot(test, aes(x = days_project_fte, y = acres, group = office)) +
#   geom_point() +
#   facet_wrap(fy ~ ., scales = "free_y")
```


## Examine Reported Acres


```r
temp <- goals %>%
  filter(region == ro & projecttypename %in% pt2) %>%
  group_by(fy, office, projecttypename) %>%
  summarize(
    goaled = sum(goaled),
    reported = sum(reported)
    ) %>%
  as.data.frame()

temp <- mutate(temp,
               office = factor(office, levels = sort(unique(temp$office), decreasing = TRUE)),
               projecttypename = factor(projecttypename, levels = pt2)
               )

ggplot(temp, aes(x = reported, y = office)) + 
  geom_point() + 
  facet_grid(~ projecttypename ~ fy, scales = "free_x") +
  ggtitle("Time Series of Acres Reported")
```

![](sdjr_summary_files/figure-html/goals-1.png)<!-- -->

```r
ggplot(temp, aes(x = fy, y = reported, group = office, linetype = office, shape = office)) + 
  geom_line(size = 0.7) + 
  geom_point(size = 2.5) +
  scale_shape_manual(values=1:nlevels(temp$office)) +
  facet_grid(projecttypename ~ ., scales = "free_y")
```

![](sdjr_summary_files/figure-html/goals-2.png)<!-- -->

```r
temp <- filter(temp, projecttypename %in% pt3)
rank_sum <- {tapply(temp$reported, list(temp$office), sum) ->.; 
  sort(.[2], decreasing = TRUE) ->.; 
  formatC(., format = "E", digits = 1)
  } # rank based on sum of acres
rank_avg <- {tapply(temp$reported, list(temp$office), mean) ->.; 
  sort(., decreasing = TRUE) ->.; 
  formatC(., format = "e", digits = 1)
  } # rank based on mean of acres

print(rank_sum)
```

```
##    11-UNI 
## "9.9E+05"
```

```r
print(rank_avg)
```

```
##    11-CLI    11-SPR    11-IND    11-JUE    11-WAV    11-ATL    11-FIN 
## "4.4e+05" "4.2e+05" "3.7e+05" "3.5e+05" "3.4e+05" "3.2e+05" "3.1e+05" 
##    11-AUR    11-GAL    11-MAN    11-UNI 
## "2.7e+05" "2.4e+05" "2.1e+05" "2.0e+05"
```


## Compare Project Acres to Goaled Acres


```r
rcors_acres <- rcors %>% 
  filter(region == ro & projecttypename %in% pt) %>%
  group_by(fy, region, office, projectname, projectiid, areasymbol) %>%
  summarize(
    n = length(unique(musym_orig)), 
    old_acres = sum(unique(muacres)), 
    new_acres = sum(unique(new_muacres))
    ) %>%
  group_by(fy, region, office, projectname) %>% 
  summarize(
    n = sum(n),
    old_acres = sum(old_acres),
    new_acres = sum(new_acres)
    )

temp <- left_join(goals, rcors_acres, by = "projectname")

progress <- temp %>%
  filter(region.x == ro & projecttypename %in% pt) %>%
  group_by(region.x, fy.x, projecttypename) %>% 
  summarize(reported = sum(reported) / 0.2,
            new_acres = sum(new_acres, na.rm = TRUE),
            dif_acres = sum(reported - new_acres, na.rm = TRUE),
            dif_pct = 100 - round(reported / new_acres * 100)
            ) 

kable(progress)
```



 region.x   fy.x  projecttypename    reported   new_acres   dif_acres   dif_pct
---------  -----  ----------------  ---------  ----------  ----------  --------
       11   2013  SDJR               14683105    14518199      164906        -1
       11   2014  SDJR               24150585    24332646     -182061         1
       11   2015  SDJR               21141050    21182385      -41335         0
       11   2016  SDJR               13828070    13786315       41755         0


## Summarize Spatial Changes


```r
r11_spatial <- rcors %>%
  filter(region %in% ro & spatial_change == TRUE & projecttypename %in% pt) %>%
  group_by(fy, region, office) %>%
  summarize(
    acres = sum(muacres, na.rm = T),
    n_areasymbol = length(unique(areasymbol)),
    n_musym = length(unique(musym_orig))
    ) %>%
  arrange(desc(acres))

ggplot(r11_spatial, aes(x = n_musym, y = n_areasymbol, group = fy, color = office)) + 
  geom_point(cex = 5) + 
  scale_colour_brewer(palette="RdYlBu") +
  facet_grid(~ fy)
```

![](sdjr_summary_files/figure-html/spatial-1.png)<!-- -->


## Map Progress by AREASYMBOL


```r
rcors_as <- rcors %>%
  filter(region == ro & projecttypename %in% pt) %>%
  group_by(fy, region, projectiid, projecttypename, areasymbol) %>%
  summarize(
    n = length(unique(new_mukey)), 
    muacres = sum(unique(muacres)), 
    new_muacres = sum(unique(new_muacres))
  ) %>%
  group_by(fy, region, areasymbol) %>%
  summarize(
    muacres = sum(muacres),
    new_muacres = sum(new_muacres)
  ) %>%
  as.data.frame()

rcors_as$acres <- cut(rcors_as$new_muacres, breaks = c(0, 10000, 50000, 100000, 150000, 200000), labels = c("0 - 10,000", "10,000 - 50,000", "50,000 - 100,000", "100,000 - 150,000", "150,000 - 200,000"))

rcors_as_w <- select(rcors_as, fy, region, areasymbol, acres) %>% 
  spread(fy, acres) # spread recycles non-grouping variables, so it needs to be combined with select, reshape() silently drops them but gives a warning if they vary
rcors_as_w1 <- reshape(rcors_as, idvar = "areasymbol", v.names = "acres", timevar = "fy", direction = "wide")
# rcors_as_w2 <- dcast(rcors_as, region + areasymbol ~ fy, value.var = "acres") # converts acres to characters instead of factors

fy <- paste0(sort(unique(rcors_as$fy)))
FY <- paste0("FY", sort(unique(rcors_as$fy)))
names(rcors_as_w)[which(names(rcors_as_w) %in% fy)] <- FY

ssa3 <- merge(ssa2, rcors_as_w, by = "areasymbol", all.x = TRUE)
temp_sub1 <- subset(ssa3, region == ro, select = FY)

# # Use the following code to plot all the county boundaries for states that intersect Region 11
# bb <- bbox(temp_sub1)
# ssa2$state <- substr(ssa2$areasymbol, 1, 2)
# st <- toupper(c("ia", "il", "in", "ks", "ky", "mi", "mn", "mo", "ne", "oh", "ok", "sd", "wi"))
# temp_sub2 <- subset(ssa3, state %in% st, select = FY)

spplot(temp_sub1,
       main = paste0("Time Series of Updated ", paste0(pt, collapse = " & "), " Acres by County"),
       #xlim = bb[1, ], ylim = bb[2, ],
       as.table = TRUE,
       col.regions = brewer.pal(5, "YlOrRd"),
       colorkey = list(space = "bottom", height = 1)
       )
```

![](sdjr_summary_files/figure-html/map_as-1.png)<!-- -->


## Summarize and plot acres by musym


