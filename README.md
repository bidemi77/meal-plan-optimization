Diet Optimization Analysis
================

INTRO
=====

This project directory contains code and data for a Meal Plan Optimization project as part of MSDS 460.

The premise of this project: Create a linear program that reccomends amounts of foods to eat for n number of consecutive days (7 days, for example). The program should seek to minimize the intake of carboydrates present across all meals. The meals must meet all nutritional requirements as prescribed by The Institue of Medicine (IOM).

FORMULATION
===========

Minimize the amount of carbohydrates present in 7 consecutive daily meals subject to upper and lower bounds on various nutritional constraints as prescribed by the IOM. Constraints include upper and lower bounds on daily intake of calories, vitamins, minerals, etc.

DATA:
=====

-   NUTRIENT REQUIREMENT DATA form [wikipedia](https://en.wikipedia.org/wiki/Dietary_Reference_Intake)
-   COMMON NUTRIENT COUNTS IN FOODS from [USDA](https://www.ars.usda.gov/northeast-area/beltsville-md-bhnrc/beltsville-human-nutrition-research-center/nutrient-data-laboratory/docs/sr28-download-files/)
    -   *Note* the values of the nutriets (in the columns) are per 100g of the food item listed. For example "BUTTER, WITH SALT" has 15.87 g of water per 100 g of Butter with Salt

PROGRAM
=======

``` r
suppressPackageStartupMessages({
  library(tidyverse)
  library(lpSolve)
})
```

Raw data
--------

``` r
raw_nutriet_data <- suppressMessages(read_csv("data/Nutrient Data.csv"))
message("Number of food items in raw nutriets data: ",scales::comma(nrow(raw_nutriet_data)))
```

    ## Number of food items in raw nutriets data: 8,790

``` r
message("Number of nutriets in raw nutriets data: ",scales::comma(length(raw_nutriet_data)-2))
```

    ## Number of nutriets in raw nutriets data: 51

``` r
raw_constraints <- suppressMessages(read_csv('data/nutrient-constraints.csv') %>% select(`Original Nutrient Name`:Unit))
```

    ## Warning: Missing column names filled in: 'X6' [6], 'X7' [7]

``` r
message("Number of constraints in raw constraints data: ",scales::comma(nrow(raw_constraints)))
```

    ## Number of constraints in raw constraints data: 35

Model Data
----------

``` r
nutriets <- read_csv("data/nutrient-categories.csv")
constraints <- read_csv("data/nutrient-constraints.csv") %>% select(`Original Nutrient Name`:Unit)
```

``` r
message("Number of food items in processed food data: ",scales::comma(nrow(nutriets)))
```

    ## Number of food items in processed food data: 1,101

Processing
----------

``` r
# keep only the cross-section of both lists
keep_names <- inner_join(data_frame(names = names(nutriets))
                         , data_frame(names = constraints$`Nutrient Name`)
                         , by = "names") %>% unlist() %>% as.character()

nutriets <- nutriets %>% select_if(names(.) %in% c("Category",keep_names))
constraints <- constraints %>% filter(`Nutrient Name` %in% keep_names)


# data cleansing -----------------------------------------------------------

# manually adding an upper caloric limit of 2,500 kcals
# 2,000 is RDI for women, 2,500 is for men
constraints <- constraints %>% 
  mutate(`Upper Bound` = ifelse(`Nutrient Name`=="Energ_Kcal", 2500,`Upper Bound`))

# if there is no upper bound, then adding UL as lower bound X2
constraints <- constraints %>% 
  mutate(`Upper Bound` = ifelse(`Upper Bound`<`Lower Bound`, `Lower Bound`*2, `Upper Bound`))


# Remove duplicate nutirents and oddities
nutriets <- nutriets %>% 
  filter(!Category %in% c("EGGS","NUTRITIONAL SUPP FOR PEOPLE W/ DIABETES"
                          , "INF FORMULA. MEAD JOHNSON"
                          ,"SHAKE", "SOY SAU", "GUMS", "SHORTENING CONFECTIONERY"
                          , "TOPPING", "MUSHROOM", "WORTHINGTON STRIPPLES"
                          , "MARGARINE-LIKE SHORTENING", "BREAKFAST BAR", "FST FOODS"
                          , "PAPA JOHN'S 14\" CHS PIZZA", "EGG MIX", "JEW'S EAR"
                          , "INF FOR","BEVERAGE", "SUNFLOWER SD BUTTER","SIDE DISHES"
                          , "CISCO", "INF FORMULA")
  ) %>% 
  filter(!grepl(c("BUTTER|OIL|MARGARINE|FAT|LARD"),Category, ignore.case = TRUE))

# nutrients are all amount per 100 grams of food, so convert to per 1 gram
nutriets <- nutriets %>% mutate_at(vars(`Water_(g)`:`Vit_A_(g)`), funs(./100))
```

View of model data after processing
-----------------------------------

### Constraints:

``` r
knitr::kable(constraints)
```

| Original Nutrient Name   | Nutrient Name      |  Lower Bound|  Upper Bound| Unit |
|:-------------------------|:-------------------|------------:|------------:|:-----|
| Calories                 | Energ\_Kcal        |      2.0e+03|      2.5e+03| kcal |
| Vitamin A                | Vit\_A\_(g)        |      9.0e-04|      3.0e-03| g    |
| Thiamin (B1)             | Thiamin\_(g)       |      1.2e-03|      2.4e-03| g    |
| Riboflavin (B2)          | Riboflavin\_(g)    |      1.3e-03|      2.6e-03| g    |
| Niacin (B3)              | Niacin\_(g)        |      1.6e-02|      3.5e-02| g    |
| Pantothenic acid (B5)    | Panto\_Acid\_g)    |      5.0e-03|      1.0e-02| g    |
| Vitamin B6               | Vit\_B6\_(g)       |      1.3e-03|      1.0e-01| g    |
| Folate (B9)              | Folate\_Tot\_(g)   |      4.0e-04|      1.0e-03| g    |
| Cyanocobalamin (B12)     | Vit\_B12\_(g)      |      2.4e-06|      4.8e-06| g    |
| Vitamin C                | Vit\_C\_(g)        |      9.0e-02|      2.0e+00| g    |
| Vitamin D                | Vit\_D\_g          |      1.5e-05|      1.0e-04| g    |
| α-tocopherol (Vitamin E) | Vit\_E\_(g)        |      1.5e-02|      1.0e+00| g    |
| Vitamin K                | Vit\_K\_(g)        |      1.2e-04|      2.4e-04| g    |
| Choline                  | Choline\_Tot\_ (g) |      5.5e-01|      3.5e+00| g    |
| Calcium                  | Calcium\_(g)       |      1.0e+00|      2.5e+00| g    |
| Copper                   | Copper\_g)         |      9.0e-04|      1.0e-02| g    |
| Iron                     | Iron\_(g)          |      1.8e-02|      4.5e-02| g    |
| Magnesium                | Magnesium\_(g)     |      4.2e-01|      8.4e-01| g    |
| Manganese                | Manganese\_(g)     |      2.3e-03|      1.1e-02| g    |
| Phosphorus               | Phosphorus\_(g)    |      7.0e-01|      4.0e+00| g    |
| Potassium                | Potassium\_(g)     |      4.7e+00|      9.4e+00| g    |
| Selenium                 | Selenium\_(g)      |      5.5e-05|      4.0e-04| g    |
| Sodium                   | Sodium\_(g)        |      1.5e+00|      2.3e+00| g    |
| Zinc                     | Zinc\_(g)          |      1.1e-02|      4.0e-02| g    |
| Carbohydrates            | Carbohydrt\_(g)    |      1.3e+02|      2.6e+02| g    |
| Water                    | Water\_(g)         |      2.7e+03|      5.4e+03| g    |
| Protein                  | Protein\_(g)       |      4.6e+01|      9.2e+01| g    |
| Fiber                    | Fiber\_TD\_(g)     |      2.5e+01|      5.0e+01| g    |
| Cholesterol              | Cholestrl\_(g)     |      3.0e-01|      6.0e-01| g    |

### Sample of food items (decision variables). First ten nutrients

``` r
set.seed(1)
knitr::kable(head(sample_frac(nutriets, 1)[,1:10]))
```

| Category        |  Water\_(g)|  Energ\_Kcal|  Protein\_(g)|  Carbohydrt\_(g)|  Fiber\_TD\_(g)|  Calcium\_(g)|  Iron\_(g)|  Magnesium\_(g)|  Phosphorus\_(g)|
|:----------------|-----------:|------------:|-------------:|----------------:|---------------:|-------------:|----------:|---------------:|----------------:|
| EGG SUBSTITUTE  |      0.0386|         4.44|        0.5550|           0.2180|           0.000|       0.00326|  0.0000316|         0.00065|          0.00478|
| HICKORYNUTS     |      0.0265|         6.57|        0.1272|           0.1825|           0.064|       0.00061|  0.0000212|         0.00173|          0.00336|
| ORANGES         |      0.8634|         0.49|        0.0094|           0.1189|           0.024|       0.00043|  0.0000010|         0.00010|          0.00017|
| TURKEY HAM      |      0.7200|         1.24|        0.1960|           0.0293|           0.000|       0.00005|  0.0000135|         0.00020|          0.00304|
| COTTONSEED MEAL |      0.0120|         3.67|        0.4910|           0.3843|           0.000|       0.00504|  0.0001335|         0.00760|          0.01684|
| TREE FERN       |      0.8860|         0.40|        0.0029|           0.1088|           0.037|       0.00008|  0.0000016|         0.00005|          0.00004|

Run the LP in a loop for n number of days
-----------------------------------------

``` r
# how many days should we plan? (each day is a loop iteration)
all_days <- 7


foods_used <- c("WATER")
all_results <- list()

for(day in 1:all_days){
  # day <- 1

  # for development purposes, sample the nutrients list. Set to 1 to use full list
  sample_size <- 1
  set.seed(1)
  sample_nutriets <- sample_frac(nutriets, sample_size)
  
  # remove foods that have already been used, other than water
  foods_used <- foods_used[foods_used!="WATER"]
  sample_nutriets <- sample_nutriets %>% filter(!Category %in% foods_used)
  
  # set objective: minimize carbohydrates
  objective_function <- sample_nutriets$`Carbohydrt_(g)`
  
  # initiate LHS constraint matrix
  Left_Hand_Side <- matrix(numeric(nrow(sample_nutriets)), nrow = 1)
  
  # for each constraint, create a row in the LHS matrix
  for(i in 1:nrow(constraints)){
    
    contraint_row <- constraints[i,]
    constraint_name <- contraint_row$`Nutrient Name`
    constraint_lower <- contraint_row$`Lower Bound`
    constraint_upper <- contraint_row$`Upper Bound`
    
    nutirent_column <- sample_nutriets %>% select_at(vars(constraint_name)) %>% unlist() %>% as.numeric() %>% matrix(nrow = 1)
    rownames(nutirent_column) <- constraint_name
    Left_Hand_Side <- rbind(Left_Hand_Side, nutirent_column)
  }
  
  # remove the initialization row at the top
  Left_Hand_Side <- Left_Hand_Side[2:nrow(Left_Hand_Side),]
  
  # direction of the constraint lower and upper bound
  constraint_directions <- c(rep(">=", nrow(Left_Hand_Side))
                             ,rep("<=", nrow(Left_Hand_Side))
  )
  
  # Lower and Upper bounds for RHS
  Right_Hand_Side <- c(constraints$`Lower Bound`
                       , constraints$`Upper Bound`)
  
  # duplicate the LHS matrix since we have both upper and lower bounds
  Left_Hand_Side_all <- rbind(Left_Hand_Side, Left_Hand_Side)
  
  ## check the rows and columns match up:
  # all_equal(nrow(Left_Hand_Side_all)
  #           , length(Right_Hand_Side)
  #           , length(constraint_directions)
  # )
  # 
  # all_equal(length(objective_function)
  #           , ncol(Left_Hand_Side_all)
  # )

  lp_time_start <- Sys.time()
  (LP_Solved <- lp(direction = "min"
                   , objective.in = objective_function
                   , const.mat = Left_Hand_Side_all
                   , const.dir = constraint_directions
                   , const.rhs = Right_Hand_Side
                   , presolve=0
                   , compute.sens=TRUE
                   , all.bin = FALSE
                   #, binary.vec
                   #, all.int=FALSE
                   #, int.vec
  ))
  (lp_time <- Sys.time()-lp_time_start)
  
  
  # record results
  result_objective <- LP_Solved$objval
  
  results <- data_frame(
    Food = sample_nutriets$Category[LP_Solved$solution>0]
    ,`Amount(g)` = LP_Solved$solution[LP_Solved$solution>0]
  )
  
  results$Day <- as.integer(day)
  
  all_results[[day]] <- results
  
  # record foods used so they will be removed from subsequent loop
  foods_used <- c(foods_used, results$Food)
  
  # display info
  message("DAY ",day, ": ", length(results$Food)," items selected. "
          ,scales::percent(sample_size), " of data used. LP completed in "
          ,round(lp_time,2), units(lp_time))
}
```

    ## DAY 1: 19 items selected. 100% of data used. LP completed in 0.03secs

    ## DAY 2: 15 items selected. 100% of data used. LP completed in 0.08secs

    ## DAY 3: 16 items selected. 100% of data used. LP completed in 0.02secs

    ## DAY 4: 19 items selected. 100% of data used. LP completed in 0.02secs

    ## DAY 5: 16 items selected. 100% of data used. LP completed in 0.02secs

    ## DAY 6: 14 items selected. 100% of data used. LP completed in 0.02secs

    ## DAY 7: 15 items selected. 100% of data used. LP completed in 0.02secs

View results
------------

All amounts are in grams per day.

For reference: - 1 cup = 320 grams - 1 liter = 1000 grams (for water)

``` r
# print results:
max_len <- max(map_dbl(all_results, nrow))
blank_rows <- data_frame(rownum = 1:max_len)

all_results_print <- all_results %>% 
  map(~.x %>% mutate(rownum = row_number())) %>% 
  map(~full_join(.x, blank_rows, by = "rownum")) %>% 
  map(~.x %>% select(-rownum)) %>% 
  map(~{
    nms <- names(.x)
    dy <- unique(.x$Day)
    dy <- dy[!is.na(dy)]
    nm1 <- paste0("Day ",dy," food")
    nm2 <- paste0("Day ",dy," amt(g)")
    names(.x) <- c(nm1, nm2, "Day")
    .x %>% select(-Day)
  }) %>% 
  bind_cols()

knitr::kable(all_results_print[,1:6])
```

| Day 1 food                  |  Day 1 amt(g)| Day 2 food                 |  Day 2 amt(g)| Day 3 food             |  Day 3 amt(g)|
|:----------------------------|-------------:|:---------------------------|-------------:|:-----------------------|-------------:|
| ANCHOVY                     |     30.432977| SOYBEAN                    |      5.059394| EEL                    |     20.594619|
| HYACINTH BNS                |     35.999703| SALT                       |      1.584396| KASHI BLACK BEAN MANGO |     27.892346|
| ROSE HIPS                   |      7.884195| MARJORAM                   |      8.214257| GRAVY                  |     82.789816|
| OOPAH (TUNICATE)            |     49.111234| CREAM                      |    138.698264| SUNFLOWER SD KRNLS     |      1.408313|
| TODDL FORM                  |     68.600362| FROG LEGS                  |    226.057724| BEVER                  |     16.937803|
| CARP                        |     29.682237| POMPANO                    |    123.831556| CAULIFLOWER            |    275.549956|
| PUMPKIN LEAVES              |     67.973415| SHORTENING FRYING HVY DUTY |     71.736076| CASSAVA                |    195.658612|
| JELLYFISH                   |      7.392385| WINGED BNS                 |      7.388633| ARUGULA                |     15.086085|
| SHORTENING INDUSTRIAL       |     70.690003| ASPARAGUS                  |    158.945184| SNAIL                  |     46.762577|
| WOCAS                       |     31.021826| MUSHROOMS                  |    144.225965| CHRYSANTHEMUM LEAVES   |    313.338360|
| PATE                        |     12.624938| VERMICELLI                 |     81.489795| SHORTENING CAKE MIX    |    125.020402|
| TOFU                        |    217.072526| SISYMBRIUM SP. SEEDS       |     37.002208| WATER                  |   1626.664695|
| CHERVIL                     |     42.945332| WATER                      |   1811.182315| SHAD                   |    191.697167|
| EGG                         |     70.804278| WHALE                      |     28.323914| AGAVE                  |     41.073746|
| WHEY                        |     74.751619| TARO                       |    236.560918| CARIBOU                |     31.516509|
| WATER                       |   2223.049462| NA                         |            NA| HEADCHEESE             |    111.109978|
| ALMONDS                     |      2.793477| NA                         |            NA| NA                     |            NA|
| EDAMAME                     |     23.486682| NA                         |            NA| NA                     |            NA|
| FLATFISH (FLOUNDER&SOLE SP) |     23.226119| NA                         |            NA| NA                     |            NA|

``` r
knitr::kable(all_results_print[,7:14])
```

| Day 4 food       |  Day 4 amt(g)| Day 5 food       |  Day 5 amt(g)| Day 6 food                   |  Day 6 amt(g)| Day 7 food            |  Day 7 amt(g)|
|:-----------------|-------------:|:-----------------|-------------:|:-----------------------------|-------------:|:----------------------|-------------:|
| HICKORYNUTS      |     19.592109| CHEESE PRODUCT   |      1.251307| SCRAPPLE                     |    257.816747| ESCAROLE              |      43.24685|
| INF FORMU        |     58.270610| CORN PUDD        |    407.655864| CANADIAN BACON               |      6.589884| GAME MEAT             |      15.93473|
| SEA BASS         |    160.680142| SAUERKRAUT       |     27.818974| HEALTHY REQUEST              |     20.035006| MACADAMIA NUTS        |      58.11086|
| FUNGI            |      9.889753| SOY FLOUR        |     20.034730| NOPALES                      |    346.229694| CREAM PUFF            |      60.95089|
| CREAM PUFF SHELL |     72.776103| PARSNIPS         |    153.986221| SHORTENING FRYING (HVY DUTY) |     81.063049| SESAME SD KRNLS       |      28.12279|
| PURSLANE         |    407.246464| LOBSTER          |    112.026580| SWORDFISH                    |     92.919379| MULLET                |     108.63113|
| STURGEON         |     26.937934| ENDIVE           |     47.652792| MALABAR SPINACH              |     57.640866| SOUR CREAM            |      69.30739|
| MUFFIN           |     31.548977| WHITEFISH        |     91.944595| PUMPKIN&SQUASH SEEDS         |      8.129893| TOMATO PRODUCTS       |     179.19978|
| INCAPARINA       |      7.472264| BAMBOO SHOOTS    |    216.187135| CRAYFISH                     |    113.618647| BREAKFAST ITEMS       |      67.73985|
| PEPPERIDGE FARM  |      1.723878| WATER            |   1631.611091| ALFALFA SEEDS                |    317.564546| SALMON                |      96.27659|
| FROZ NOVLT       |     21.390540| SESAME MEAL      |      4.799658| WATER                        |   1289.683539| INFFORMULA            |      50.76957|
| SHORTENING       |    121.014855| ROE              |      7.630027| JERUSALEM-ARTICHOKES         |    344.906938| WATER                 |    1322.41611|
| SOYBEANS         |    193.441428| BRATWURST        |     51.912580| VEGETABLE JUC                |      1.223677| PUMPKIN FLOWERS       |     365.53506|
| CAVIAR           |     11.261429| SAVORY           |     15.860487| BALSAM-PEAR (BITTER GOURD)   |    137.704637| CATTAIL               |     565.51665|
| OLIVES           |     44.716665| SHORTENING BREAD |    135.894680| NA                           |            NA| HAZELNUTS OR FILBERTS |      39.46945|
| WATER            |   1777.837106| DOCK             |    193.702075| NA                           |            NA| NA                    |            NA|
| ACEROLA JUICE    |      1.049625| NA               |            NA| NA                           |            NA| NA                    |            NA|
| WATERCHESTNUTS   |    158.510808| NA               |            NA| NA                           |            NA| NA                    |            NA|
| OYSTER           |      3.788686| NA               |            NA| NA                           |            NA| NA                    |            NA|
