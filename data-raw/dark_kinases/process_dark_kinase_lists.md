Processing the Dark Kinase Lists
================
Matthew Berginski

    ## ── Attaching packages ────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse 1.2.1 ──

    ## ✔ ggplot2 3.0.0     ✔ purrr   0.2.5
    ## ✔ tibble  1.4.2     ✔ dplyr   0.7.5
    ## ✔ tidyr   0.8.1     ✔ stringr 1.3.1
    ## ✔ readr   1.1.1     ✔ forcats 0.3.0

    ## ── Conflicts ───────────────────────────────────────────────────────────────────────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()

    ## here() starts at /home/mbergins/Documents/Projects/DarkKinaseTools

    ## 
    ## TERMS OF USE NOTICE:
    ##   When using Synapse, remember that the terms and conditions of use require that you:
    ##   1) Attribute data contributors when discussing these data or results from these data.
    ##   2) Not discriminate, identify, or recontact individuals or groups represented by the data.
    ##   3) Use and contribute only data de-identified to HIPAA standards.
    ##   4) Redistribute data only under these same terms of use.

DRGC List
---------

This notebook describes the data cleaning and processing steps used to build the dark kinase lists from several sources. The first of which is the spreadsheet produced by the research groups that lists the kinases included in the Dark set. This spreadsheet was made in excel and mostly uses HGNC identifiers. There is a single special case "SGK494", which is dealt with below.

Save the resulting list as a data set that should be readily available

``` r
dark_kinases_raw = readxl::read_xlsx(here('data-raw/dark_kinases/Modified IDG Kinase List for NIH.xlsx'))

dark_kinases_set = dark_kinases_raw %>%
  filter(`Keep/Add` != 'Remove' | is.na(`Keep/Add`)) %>%
  filter(! is.na(`Approved name`)) %>%
  mutate(hgnc_symbol = `Approved name`) %>%
  rename(DRGC_symbol = `Approved name`) %>%
  #There is one symbol in the "Approved name" that isn't in the HGNC list:
  #SGK494, mark it as NA
  mutate(hgnc_symbol = case_when(
    hgnc_symbol == "SGK494" ~ "NA",
    TRUE ~ as.character(hgnc_symbol)
  )) %>%
  select(hgnc_symbol,DRGC_symbol)

# devtools::use_data(dark_kinases, overwrite = TRUE)
```

Kinase.com List
---------------

There is a list of kinases maintained on kinase.com that stems from the original Manning et al 2002 paper that used the early human genome sequence to identify all (maybe?) kinases. The resulting list is an excel spreadsheet with a wide range of columns. For now, I'm only really interested in using this list to get a full set of kinases collected and organized with a standardized list of names/IDs. Unfortunately, the list has it's own, probably historical, names for each kinase. I want to keep these because the other lists on kinase.com (such as mouse) also use these names. Instead of these, I'll key off the list HGNC IDs.

``` r
kinome_com_file = here('data-raw','dark_kinases','kinase.com_list.xls')
if (! file.exists(kinome_com_file)) {
  download.file('http://kinase.com/human/kinome/tables/Kincat_Hsap.08.02.xls',
                kinome_com_file);
}

kinase_com_list = readxl::read_xls(kinome_com_file);
#The list from Kinase.com has a set of psuedogenes at the end, which we won't work with
kinase_com_list = kinase_com_list %>% filter(`Pseudogene?` == "N")

#Several of the kinases listed have been assigned HGNC IDs now, so I manually made a list of these 
additional_hgncs = read.csv(here('data-raw','dark_kinases','additional_hgnc_IDs.csv'))
for (this_row_num in 1:dim(additional_hgncs)[1]) {
  this_row = additional_hgncs[this_row_num,]
  kinase_row = grep(this_row$kinase_name,kinase_com_list$Name)
  
  new_cross_ref_str = paste0(this_row$HGNC.ID,"|",kinase_com_list$Entrez_dbXrefs[kinase_row])
  
  kinase_com_list$Entrez_dbXrefs[kinase_row] = new_cross_ref_str
}

kinase_com_list$hgnc_id = str_extract(kinase_com_list$Entrez_dbXrefs,"HGNC:[:digit:]+")
kinase_com_simplified = kinase_com_list %>%
  rename(kinase_com_name = Name) %>%
  select(kinase_com_name,hgnc_id)
```

Moret List
----------

A list of kinases has been compiled by Nienke Moret in Peter Sorger's lab. We'll also pull this list in from synapse and integrate it in the final section.

``` r
synLogin()
```

    ## Welcome, Matthew Berginski!

    ## NULL

``` r
fileEntity <- synGet("syn12617467")

moret_kinase_list = read_csv(fileEntity$path)
```

    ## Parsed with column specification:
    ## cols(
    ##   gene_id = col_integer(),
    ##   gene_symbol = col_character(),
    ##   name = col_character(),
    ##   in_manning = col_logical(),
    ##   in_kinmap = col_logical(),
    ##   in_uniprot_kinasedomain = col_logical(),
    ##   in_IDG_darkkinases = col_logical(),
    ##   n_pubmed_citations_2013to2018 = col_integer(),
    ##   pharos_designation = col_character()
    ## )

HGNC List
---------

The HGNC (Human Gene Naming Consortium) maintains a list of accepted identifiers that have been approved by the consortium and seem to be relatively well used. I'll use the list from kinase.com and the dark kinase list made by the research group to get the approved names of most of the kinases (some don't have approved names).

``` r
hgnc_protein_file = here('data-raw','dark_kinases','hgnc_complete_set.txt')
if (! file.exists(hgnc_protein_file)) {
  download.file('ftp://ftp.ebi.ac.uk/pub/databases/genenames/new/tsv/hgnc_complete_set.txt',
                hgnc_protein_file);
}

#Toss out entries which have been withdrawn from the database
HGNC_list = read.delim(hgnc_protein_file) %>%
  filter(status != "Entry Withdrawn");

#Filtering out the HGNC IDs from the kinase.com list, thankfully the format of the ID is identical to that used on the HGNC
HGNC_Kinase_IDs = str_match(kinase_com_list$Entrez_dbXrefs,"HGNC:[:digit:]+")

#Two kinases lack HGNC ids: SgK494/SgK424, so they won't make it through the
#HGNC ID filter. In addition we added several pseudokinases to the list, so they
#should also make it to the master list, add them in with a filter check.
HGNC_Kinases_Full = HGNC_list %>%
  filter(hgnc_id %in% HGNC_Kinase_IDs |
         symbol %in% dark_kinases_set$hgnc_symbol |
         symbol %in% moret_kinase_list$gene_symbol)

#Add the Light/Dark Classification to HGNC_kinases and select only a few columns
all_kinases = HGNC_Kinases_Full %>% mutate(
  class = case_when(
    symbol %in% dark_kinases_set$hgnc_symbol ~ "Dark",
    TRUE ~ "Light"
  )
) %>% select(c("hgnc_id","symbol","ensembl_gene_id","class","name","uniprot_ids","entrez_id"))

#Join in a the Manning names for the kinases
all_kinases = left_join(all_kinases,kinase_com_simplified)
```

    ## Joining, by = "hgnc_id"

    ## Warning: Column `hgnc_id` joining factor and character vector, coercing
    ## into character vector

``` r
write_csv(all_kinases,here('data/all_kinases.csv'))

dark_kinases = all_kinases %>% filter(class == "Dark")
write_csv(dark_kinases,here('data/dark_kinases.csv'))

devtools::use_data(dark_kinases, overwrite = TRUE)
```

    ## Saving dark_kinases as dark_kinases.rda to /home/mbergins/Documents/Projects/DarkKinaseTools/data

``` r
devtools::use_data(all_kinases, overwrite = TRUE)
```

    ## Saving all_kinases as all_kinases.rda to /home/mbergins/Documents/Projects/DarkKinaseTools/data
