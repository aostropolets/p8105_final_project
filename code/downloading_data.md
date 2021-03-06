downloading\_data
================

``` r
library(readxl)
library(rvest)
library(stringr)
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.1 ──

    ## ✓ ggplot2 3.3.5     ✓ purrr   0.3.4
    ## ✓ tibble  3.1.4     ✓ dplyr   1.0.7
    ## ✓ tidyr   1.1.3     ✓ forcats 0.5.1
    ## ✓ readr   2.0.1

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::filter()         masks stats::filter()
    ## x readr::guess_encoding() masks rvest::guess_encoding()
    ## x dplyr::lag()            masks stats::lag()

``` r
library(filesstrings)
```

    ## 
    ## Attaching package: 'filesstrings'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     all_equal

``` r
library(lubridate)
```

    ## 
    ## Attaching package: 'lubridate'

    ## The following objects are masked from 'package:base':
    ## 
    ##     date, intersect, setdiff, union

``` r
library(ZipRadius)
library(httr)
```

**if you want to extract more columns from a particular file/tab, you
can add the column name to the list at the bottom of this page**

### Current data groupings:

-   wl\_tx\_counts: overall waitlist information, counts and
    percentages, over a 2y period
-   wl\_demo: demographic and medical information about pts on waitlist
-   tx\_out: variables associated with delayed organ function, whether
    the organ was brought in from an outside institution and whether the
    pt needed dialysis in the first week post tx
-   ctr\_oar: center-level info on offer/acceptance ratios at various
    KDRI scores (*not all years have this info*)
-   ttt: centiles for time to transplant (in months)
-   ddtx\_demo: demographic and medical info about pts who received
    deceased donor transplants
-   ldtx\_demo: demographic and medical info about pts who received
    living donor transplants
-   wl\_out: variables associated with waitlist removal, including
    transplant rates and mortality rates

# Pipeline

Creating list of date codes and URLs for the current data file and all
archived files.

Dates and URLs came from inspecting this website:
<https://www.srtr.org/reports/program-specific-reports/>

``` r
# archive date format is YYMM, two reports per year
# current year is available with kidney only data, but other years are in .zip

# there was a change in how regions were assigned in 2017
# archive_dates_17 = c("1711", "1808", "1811", "1905", "1911", "2006")

archive_dates = c("1207", "1301", "1307", "1401", "1406", "1412", "1506", "1512", "1606", "1701", "1707", "1711", "1808", "1811", "1905", "1911", "2006")

current_file_date = "2105"

all_dates = append(archive_dates, current_file_date)

current_file_url = "https://www.srtr.org/assets/media/PSRdownloads/csrs_tables/csrs_final_tables_2105_KI.xls"

url_base = "https://www.srtr.org/assets/media/PSRdownloads/csrs_tables_all/csrs_final_tables_"

# replace with archive_dates to get all years
vec_urls = str_c(url_base, archive_dates, "all.zip")

vec_dates = list(
  date = archive_dates,
  url = vec_urls
)
```

Download the most recent data and save it to `/data` folder:

``` r
current_file_path = "../data/csrs_final_tables_2105_KI.xls"

if (!file.exists(current_file_path)) {
  download.file(current_file_url, current_file_path, mode = "wb")}

# clean the incoming sheets from each file
# remove the second unnecessary descriptor line from each excel file
# add a column with the report date
# parse dates
# clean names
# add an ID code since CTR_CD does not appear consistently across sheets

clean_sheets = function(fpath, x) {
  
  clean_cols = array(readxl::read_xls(fpath, sheet = x, n_max = 1, col_names = FALSE))
  
  clean_sheet = readxl::read_xls(fpath, sheet = x, skip = 2, col_names = FALSE)
  colnames(clean_sheet) = clean_cols
  
  report_date = reduce(map(.x = fpath, ~str_extract(.x, "\\d{4}")), ~ .) %>%
    parse_date_time(orders = c("%y%m"), truncated = 3, exact = TRUE)
  
  clean_sheet = clean_sheet %>%
    janitor::clean_names("all_caps") %>%
    mutate(REPORT_DATE = report_date) %>%
    rowwise() %>%
    mutate(CTR_ID = ifelse(
      "CTR_CD" %in% colnames(.), 
      paste(CTR_CD, CTR_TY, sep = ""), 
      ifelse("CENTER" %in% colnames(.), CENTER, NA))) %>%
    select(CTR_ID, REPORT_DATE, everything()) %>%
  return(clean_sheet)
}

current_data = suppressMessages(sapply(readxl::excel_sheets(current_file_path), 
                      simplify = F, 
                      USE.NAMES = T, 
                      function (X) clean_sheets(current_file_path, X)))
```

Function to download archived files:

``` r
download_archived_data = function(url) {
  
  data_path = "../data"
  ki_file_pattern = "KI\\.[xX][lL][sS]$"
  
  file_date = reduce(map(.x = url, ~str_extract(.x, "\\d{4}")), ~ .)
  
  ki_file_name = reduce(list.files(path = data_path, pattern = file_date, full.names = TRUE), ~ .)
  
  
  # don't redownload data if you already have it
  
  if (!file.exists(ki_file_name)) {
    temp = tempfile()
      
    download.file(url, temp, mode = "wb")
    
    ki_file = tibble(files = as.character(unzip(temp, list = TRUE)$Name)) %>% filter(str_detect(files, ki_file_pattern))[[1]]

    ki_file_name = unzip(temp, ki_file)
    move_files(ki_file_name, "../data/", overwrite = FALSE)
    unlink(temp)
  }
  
  ki_data = suppressMessages(sapply(readxl::excel_sheets(ki_file_name), 
                   simplify = F, USE.NAMES = T,
                   function(X) clean_sheets(ki_file_name, X)))
  
  return(ki_data)
}
```

Compiling all data files (archived and current) into one nested list:

``` r
data_all_yrs = map(.x = vec_dates$url, ~download_archived_data(url = .x))

data_all_yrs = append(data_all_yrs, list(current_data))

name_labels = vec_dates$date %>% 
  append(current_file_date)

names(data_all_yrs) = str_c("data_", name_labels)
```

Cleaning data names and specifying variables to extract from each file
and sheet via functions

``` r
# each xls file has a number of tabs representing different data types
# will use these columns for future merge/joins

id_cols = c("CTR_ID", "REPORT_DATE")

# could probably do regex for these but save that for another day...

# columns relating to numbers on waitlist, number added for 2 distinct years, number of total pts on the waitlist at the end of 2 years, number removed due to transplant(C = decased, L = living), number removed due to death and deterioration

waitlist_cols = c("WLA_ADDCEN_NC2", "WLA_ST_NC2", "WLA_END_NC2", "WLA_REMTXC_NC2", "WLA_REMTXL_NC2", "WLA_REMTXC_PCZ", "WLA_REMTXL_PCZ", "WLA_REMDIED_PCZ", "WLA_REMDET_PCZ")

# columns relating to waitlist demographics - age groups, gender, race/ethnicity, blood type, peak PRA >=80%, previous transplant, primary dx

waitlist_demo_cols = c("WLC_A2_ALLC2", "WLC_A10_ALLC2", "WLC_A17_ALLC2", "WLC_A34_ALLC2", "WLC_A49_ALLC2", "WLC_A64_ALLC2", "WLC_A65P_ALLC2", "WLC_A69_ALLC2", "WLC_A70P_ALLC2", "WLC_GM_ALLC2", "WLC_GF_ALLC2", "WLC_RA_ALLC2", "WLC_RB_ALLC2", "WLC_RH_ALLC2", "WLC_RO_ALLC2", "WLC_RU_ALLC2", "WLC_RW_ALLC2", "WLC_BAB_ALLC2", "WLC_BA_ALLC2", "WLC_BB_ALLC2", "WLC_BO_ALLC2", "WLC_BU_ALLC2","WLC_PRA80_ALLC2", "WLC_PTXY_ALLC2", "WLC_KIDIA_ALLC2", "WLC_KIGLO_ALLC2", "WLC_KIHYP_ALLC2", "WLC_KIMIS_ALLC2", "WLC_KINEO_ALLC2", "WLC_KIOTH_ALLC2", "WLC_KIPOL_ALLC2", "WLC_KIREN_ALLC2", "WLC_KIRTR_ALLC2", "WLC_KITUB_ALLC2", "WLC_KICON_ALLC2")

# columns relating to delayed organ function, % of kidneys that were "imported" and % of pts on dialysis at 1 week
tx_out_cols = c("TOC_SHR_C", "TOC_WKDIALY_C")

# columns relating to offer acceptance ratios
# not all of the years have these data

ctr_oar_cols = c("OA_OVERALL_HR_MN_CENTER", "OA_LOWRISK_HR_MN_CENTER", "OA_MEDIUMRISK_HR_MN_CENTER", "OA_HIGHRISK_HR_MN_CENTER", "OA_HARDTOPLACE100_HR_MN_CENTER", "OA_HARDTOPLACE_HR_MN_CENTER")

# columns with time to transplant by centile in months

ttt_cols = c("TTT_25_C", "TTT_50_C", "TTT_75_C")

# deceased donor transplant recipients demographics

ddtx_demo_cols = c("RCC_A2_C", "RCC_A10_C", "RCC_A17_C", "RCC_A34_C", "RCC_A49_C", "RCC_A64_C", "RCC_A65P_C",
"RCC_A69_C","RCC_A70P_C","RCC_GM_C", "RCC_GF_C", "RCC_RA_C", "RCC_RB_C", "RCC_RH_C", "RCC_RO_C", "RCC_RU_C", "RCC_RW_C", "RCC_BAB_C", "RCC_BA_C", "RCC_BB_C", "RCC_BO_C", "RCC_BMI20_C", "RCC_BMI25_C", "RCC_BMI30_C", "RCC_BMI_31P_C","RCC_BMI35_C","RCC_BMI40_C", "RCC_BMI41P_C", "RCC_DIA_C", "RCC_GLO_C", "RCC_HYP_C", "RCC_MIS_C", "RCC_NEO_C", "RCC_OTK_C", "RCC_POL_C", "RCC_VAS_C", "RCC_RET_C", "RCC_TUB_C", "RCC_CON_C", "RCC_PRA80_C", "RCC_PTXY_C")

# living donor recipient demographics
# # can probably turn this into a regex expression for demographic categories for waitlist, and DD and LD recipients "RC[CL]_[+/-KIvariables of interest]_C"

ldtx_demo_cols = str_replace(ddtx_demo_cols, "^RCC", "RCL")
ldtx_demo_cols = str_replace(ldtx_demo_cols, "_RET_", "_KIRET_")

# info on why people were removed from waitlist because of receiving a transplant the transplant rate

# transplant rate per year on waitlist, ratio of observed/exepcted, deceased donor transplant rate and ratio, and mortality rate

wl_out_cols = c("TMR_TXR_C2", "TMR_TXRATIO_C2", "TMR_CADTXR_C2", "TMR_CADTXRATIO_C2", "TMR_DTHR_C2", "TMR_DTHRATIO_C2")

wl_out_cols2 = toupper(c("TMR_Tx_R_c", "TMR_Tx_Ratio_c", "TMR_Cad_Tx_R_c", "TMR_Cad_Tx_Ratio_c", "TMR_Dth_R_c", "TMR_Dth_Ratio_c"))
```

Define the functions to pull out the data of interest. Because of
inconsistencies across years and sheets, some of this must be hard
coded.

``` r
select_and_clean = function(df, select_cols) {
  df = df %>%
    select(1, all_of(id_cols), any_of(select_cols)) %>%
    mutate(across(.cols = any_of(select_cols), ~ as.numeric(as.character(.))))
  return(df)
}

t1_function = function(df) {
  tbl1 = as_tibble(df[[1]]) %>% 
    select_and_clean(., waitlist_cols)
  tbl1
}

t2_function = function(df) {
  tbl2 = as_tibble(df[[2]]) %>% 
    select_and_clean(., waitlist_demo_cols)
  tbl2
}

t3_function = function(df) {
  if (length(df) < 32) {
    tbl3 = as_tibble(df[[22]])}
  else if (length(df) == 32) {
    tbl3 = as_tibble(df[[23]])}
  else if (length(df) == 33) {
    tbl3 = as_tibble(df[[25]])}
  else if (length(df) == 34) {
    tbl3 = as_tibble(df[[26]])}
  
  tbl3 = tbl3 %>% 
    select_and_clean(., tx_out_cols)
  
  tbl3
}

t4_function = function(df) {
  
  if (length(df) < 32) {
    tbl4 = as_tibble("")}
  else {
    if (length(df) == 32) {
      tbl4 = as_tibble(df[[16]]) %>%
        rename( OA_HARDTOPLACE100_HR_MN_CENTER = OA_HARDTOPLACE_HR_MN_CENTER)}
    else if (length(df) == 33) {
      tbl4 = as_tibble(df[[18]])}
    else if (length(df) == 34) {
      tbl4 = as_tibble(df[[19]])}
    tbl4 = tbl4 %>% 
      select_and_clean(., ctr_oar_cols)
    
  
  }
  tbl4  
}

t5_function = function(df) {
  if (length(df) < 33) {
    tbl5 = as_tibble(df[[15]])}
  else if (length(df) == 33) {
    tbl5 = as_tibble(df[[17]])}
  else if (length(df) == 34){
    tbl5 = as_tibble(df[[18]])}
  
  tbl5 = tbl5 %>%
    select_and_clean(., ttt_cols)
  
  tbl5
}

t6_function = function(df) {
  if (length(df) < 32) {
    tbl6 = as_tibble(df[[16]])}
  else if (length(df) == 32) {
    tbl6 = as_tibble(df[[17]])}
  else if (length(df) == 33) {
    tbl6 = as_tibble(df[[19]])}
  else if (length(df) == 34) {
    tbl6 = as_tibble(df[[20]])}
  
  tbl6 = tbl6 %>%
    select_and_clean(., ddtx_demo_cols)
  
  tbl6
}


t7_function = function(df) {
  if (length(df) < 32) {
    tbl7 = as_tibble(df[[18]])}
  else if (length(df) == 32) {
    tbl7 = as_tibble(df[[19]])}
  else if (length(df) == 33) {
    tbl7 = as_tibble(df[[21]])}
  else if (length(df) == 34) {
    tbl7 = as_tibble(df[[22]])}
  
  tbl7 = tbl7 %>%
    select_and_clean(., ldtx_demo_cols)
  tbl7
}

t8_function = function(df) {
  if (length(df) < 33) {
    tbl8 = as_tibble(df[[5]]) %>%
      select_and_clean(., wl_out_cols)}
  else if (length(df) >= 33) {
    tbl8 = as_tibble(df[[5]]) %>% 
      janitor::clean_names("all_caps") %>%
      select_and_clean(., wl_out_cols2) %>%
      rename_with(~ wl_out_cols[which(wl_out_cols2 == .x)], .cols = wl_out_cols2)
    }
  tbl8
}
```

Now pull the variables of interest out of the `data_all_yrs`
list-columns and group them together as a list of tibbles.

``` r
wl_tx_counts = map(data_all_yrs, t1_function)
wl_demo = map(data_all_yrs, t2_function)
tx_out = map(data_all_yrs, t3_function)
ctr_oar = map(data_all_yrs, t4_function)
ttt = map(data_all_yrs, t5_function)
ddtx_demo = map(data_all_yrs, t6_function)
ldtx_demo = map(data_all_yrs, t7_function)
wl_out = map(data_all_yrs, t8_function)

var_groups = list(wl_tx_counts, wl_demo, tx_out, ctr_oar, ttt, ddtx_demo, ldtx_demo, wl_out)
names(var_groups) = c("wl_tx_counts", "wl_demo", "tx_out", "ctr_oar", "ttt", "ddtx_demo", "ldtx_demo", "wl_out")
```

Get a list of the transplant centers within a specific radius. Per lit
review, max generally accepted radius is 250 miles.

Shiny dashboard or interactivity component can set the `radius_mi`
argument to a positive numeric value (int or dbl) to specify how far
away they want to look for a given center.

``` r
# get a list of the transplant centers within a set radius of a zip code
# the OPTN regions divide some states like VA and VT into two parts, which is not handled here

optn_regions = read_html("https://optn.transplant.hrsa.gov/about/search-membership/?memberType=Transplant%20Centers&organType=%27AL%27&state=0&region=0") %>%
  html_elements(".listTable") %>%
  html_table() %>% 
  reduce(.f = ~ .) %>%
  as_tibble() %>%
  janitor::clean_names("all_caps") %>%
  filter(str_detect(PROGRAMS, "Kidney") == TRUE) %>%
  select(NAME, REGION) %>%
  mutate(NAME = paste(substr(NAME, 1, 4), "TX1", sep = ""),
         REGION = as.factor(REGION),
         REGION = addNA(REGION)) %>%
  rename(CTR_ID = NAME) %>%
  as_tibble()
  
ctr_location = current_data[["Tiers"]] %>%
  select(CTR_ID, ENTIRE_NAME, PRIMARY_CITY, PRIMARY_STATE, PRIMARY_ZIP) %>%
  left_join(., optn_regions, by = "CTR_ID", keep = FALSE)

get_radii = function (zip_code = "10108", radius_mi = 250) {
  max_radius = 250
  zip_code = str_sub(zip_code, 1, 5)
  
  within_radius = suppressMessages(zipRadius(zip_code, max_radius) %>%
    janitor::clean_names() %>%
    select(zip, distance) %>%
    filter(distance <= radius_mi,
           zip %in% ctr_location$PRIMARY_ZIP) %>%
    rename(PRIMARY_ZIP = zip,
           DISTANCE = distance) %>%
    left_join(ctr_location) %>%
    as_tibble() %>%
    rename_with(~ paste(.,"_WR", sep = "")))
  return(within_radius)
}

ctr_location = ctr_location %>%
  mutate(WITHIN_RADIUS = map(PRIMARY_ZIP, get_radii))
```

Build the final dataframe to store as a CSV/XLS file

``` r
# group by the center name/center code
# 
all_data_df = ctr_location %>% ungroup()

for (var_group in names(var_groups)) {
 
  new_df = unchop(var_groups[[var_group]]) %>% bind_rows()

  if (match(var_group,c(names(var_groups))) == 1) {
    all_data_df = full_join(all_data_df, new_df, by = c("CTR_ID"))}
  else {
    all_data_df = full_join(all_data_df,new_df)}
}
```

    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")
    ## Joining, by = c("CTR_ID", "REPORT_DATE")

``` r
all_data_df %>%
  unnest(WITHIN_RADIUS) %>%
  write_excel_csv(file = "../data/all_KI_data.csv")
```
