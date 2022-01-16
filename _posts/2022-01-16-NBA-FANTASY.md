---
layout: post
title: "NBA Fantasy Basketball Projections"
always_allow_html: yes
output: 
  md_document: 
    variant: gfm
    preserve_yaml: true
---

------------------------------------------------------------------------

## Introduction

This year I joined an NBA fantasy basketball league with some
co-workers. I’ve played NHL fantasy hockey for years, but this is my
first time with basketball. And I’ve been loving it. It also prompted a
few fun data project ideas (well, I think they’re fun).

To do well in these fantasy leagues you must adjust your team as players
under- or over-perform expectations. My first mini data project is
pretty straightforward. I’ve heavily relied on rankings and projections
from [Hashtag Basketball](https://hashtagbasketball.com/), which my
friend recommended. Using the site’s data, I want to compare each
player’s current and projected value. This should help me target players
that are expected to improve, or sell off players expected to decline.

Most of the work for today is cleaning the data from the website, with a
couple summary tables at the end. I’m planning some further analysis and
visualizations for next time.

------------------------------------------------------------------------

## Background on Fantasy Basketball

First, if you’re not familiar with how these make-believe sports leagues
work, here’s the short version: a league is typically made of 10-12
friends/co-workers/strangers who select fantasy “teams” of real-life
players. So in my NBA league, I selected 13 players in October for my
fantasy team. If a player is on my fantasy team, none of my co-workers
can select that same player. Each week, my team plays “against” one of
my co-workers’ teams. Over the course of the week, our players compete
in nine statistical categories (often referred to as “9-cat”). The
categories are:

Field goal percentage (FG%) Free throw percentage (FT%) 3-pointers made
(3PTM) Points (PTS) Rebounds (REB) Assists (AST) Steals (STL) Blocks
(BLK) Turnovers (TO)

Whichever team finishes higher in each category (or lower in turnovers)
gets a “win” for that category. For example, in a recent week I played
against my manager. I scored higher in FG%, FT%, 3PTM, PTS, and STL. He
scored higher in REB, AST, BLK and TO. So I won the week 6 to 3, since
the player with lower TO wins that category.

Over the course of the year each team’s weekly results get added up. The
top six teams at the end of the fantasy season make it to playoffs, and
so on.

As I mentioned earlier, you have to make a lot of adjustments to your
team in order to do well. Players get hurt, play poorly, or surpass
expectations. So by comparing current player rankings to Hashtag
Basketball’s projections, I can hopefully gain an edge on my co-workers
who don’t have the same information.

------------------------------------------------------------------------

## Load Packages in R

``` r
library(tidyverse)
library(knitr)
```

    ## Rows: 430 Columns: 16

    ## -- Column specification --------------------------------------------------------
    ## Delimiter: ","
    ## chr (16): R#, PLAYER, POS, TEAM, GP, MPG, FG%, FT%, 3:00 PM, PTS, TREB, AST,...

    ## 
    ## i Use `spec()` to retrieve the full column specification for this data.
    ## i Specify the column types or set `show_col_types = FALSE` to quiet this message.

    ## Rows: 601 Columns: 16

    ## -- Column specification --------------------------------------------------------
    ## Delimiter: ","
    ## chr (16): R#, PLAYER, POS, TEAM, GP, MPG, FG%, FT%, 3:00 PM, PTS, TREB, AST,...

    ## 
    ## i Use `spec()` to retrieve the full column specification for this data.
    ## i Specify the column types or set `show_col_types = FALSE` to quiet this message.

------------------------------------------------------------------------

## Data Load and Cleaning

To get the data, I simply copy/pasted the full
[ranking](https://hashtagbasketball.com/fantasy-basketball-rankings) and
[projection](https://hashtagbasketball.com/fantasy-basketball-projections)
tables for all players from the Hashtag Basketball website. I made sure
to include z-scores for all categories because I will use these later. I
then loaded those into R from CSV.

I had to tidy up a few things based on display features of the website,
which was good practice for filtering and transforming in R. I even got
into some for loops for splitting multiple columns. If you’re not too
concerned with data cleaning in R, definitely jump to the next section.

The data tables on the website list the column headers every twenty or
so rows for easy scrolling. So I quickly removed those by taking a
subset of the initial tables, where the player’s name was not “PLAYER”
(i.e. a header row).

``` r
Hashtag_proj <- subset(Hashtag_bball_proj, PLAYER != "PLAYER")
Hashtag_rank <- subset(Hashtag_bball_rank, PLAYER != "PLAYER")
```

Next, a minor correction that any spreadsheet user is likely familiar
with: correcting a field assumed to be a date. In this case the “3PTM”
column name came in as “3:00 PM”.

``` r
Hashtag_proj <- Hashtag_proj %>%
  rename("3PTM" = "3:00 PM", "RANK" = "R#")

Hashtag_rank <- Hashtag_rank %>%
  rename("3PTM" = "3:00 PM", "RANK" = "R#")
```

As mentioned, I included z-scores in the Hashtag Basketball tables.
However, the z-score values are included in the same field as their
corresponding value for that category. For the projections, they’re
separated by a line break. Splitting columns using a line breaker as a
delimiter is a bit unwieldy, so I replaced the line breaks (denoted “”
with “\#”).

``` r
Hashtag_proj <- Hashtag_proj %>% 
  mutate_all(funs(str_replace(., "\n", "#")))
```

    ## Warning: `funs()` was deprecated in dplyr 0.8.0.
    ## Please use a list of either functions or lambdas: 
    ## 
    ##   # Simple named list: 
    ##   list(mean = mean, median = median)
    ## 
    ##   # Auto named with `tibble::lst()`: 
    ##   tibble::lst(mean, median)
    ## 
    ##   # Using lambdas
    ##   list(~ mean(., trim = .2), ~ median(., na.rm = TRUE))
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was generated.

The rankings table separated the values and z-scores with a space, so no
replacement was needed there.

Now, what I want to do is split each of the nine statistical categories
into a value column and a z-score column (the FG% and FT% columns also
include additional data that I’ll split out after handling the
z-scores). For the z-score columns I’ll just add a “z” prefix
(e.g. “3PTM” for values, “z3PTM” for z-scores).

Since the naming convention and column split delimiter will be the same
for each column in the table (“\#” for projections, " " for rankings), I
wanted to be more efficient than splitting each column individually. I
ended up using a for loop for fun. But I should note that for loops
aren’t very computationally efficient in R (from what I read). So there
is probably an even better way to do this, but I settled for generalized
naming and column splits compared to repetitive code.

For the statistical categories I need to split (column indexes 7 to 15),
I separate the column i into one column with the same name, and one with
the “z” prefix (using the paste function). And then I specify the
delimiter (I included “)” as a delimiter for the rankings field as well,
because of the FG% and FT% values I mentioned earlier). Lastly, the
“convert = TRUE” condition will convert all of the values to numbers
(they were initially loaded as characters).

``` r
for (i in names(Hashtag_proj)[7:15]){
  Hashtag_proj <- Hashtag_proj %>%  
    separate(i, c(i, paste("z", i, sep = "")), sep = "#", convert = TRUE)
}

for (i in names(Hashtag_rank)[7:15]){
  Hashtag_rank <- Hashtag_rank %>%  
    separate(i, c(i, paste("z", i, sep = "")), sep = "([) ])", convert = TRUE)
  }
```

Lastly, I need to do some additional splitting for the FG% and FT%
categories. The z-scores were split out above, but the cells for these
percentage stats include a fraction as well. So one cell might be
0.576(10.9/18.9). I want to split this into three columns: FG% (the
first decimal number), field goals made (abbreviated FGM, the numerator
of the fraction), and field goals attempted (FGA, the denominator of the
fraction). I can do this using the separate function again, with “(”,
“/” and “)” as separators. In truth, I already used the “)” for the
rankings separation above. This seemed to work better on the rankings
side.

``` r
Hashtag_proj <- Hashtag_proj %>%
  separate("FG%", 
           c("FG%", "FGM", "FGA"), 
           sep = "([(/)])", 
           convert = TRUE,
           extra = "drop") %>%
  separate("FT%", 
           c("FT%", "FTM", "FTA"), 
           sep = "([(/)])",
           convert = TRUE,
           extra = "drop")

Hashtag_rank <- Hashtag_rank %>%
  separate("FG%", 
           c("FG%", "FGM", "FGA"), 
           sep = "([(/])", 
           convert = TRUE,
           extra = "drop") %>%
  separate("FT%", 
           c("FT%", "FTM", "FTA"), 
           sep = "([(/])",
           convert = TRUE,
           extra = "drop")
```

Now, finally, all statistical categories are properly split into values
and z-scores, with the additional FGM, FGA, FTM, and FTA columns
discussed. We’re just about ready to merge the tables together, and be
able to compare players’ ranking values to their projections.

First, a quick check to see if we have any NA values that could trip up
future calculations. I used summarise and across to see the count of NA
values in each

``` r
Hashtag_rank %>%
  summarise(across(everything(), ~ sum(is.na(.))))
```

    ## # A tibble: 1 x 29
    ##    RANK PLAYER   POS  TEAM    GP   MPG `FG%`   FGM   FGA `zFG%` `FT%`   FTM
    ##   <int>  <int> <int> <int> <int> <int> <int> <int> <int>  <int> <int> <int>
    ## 1     0      0     0     1     0     0     0     0     0      0     0     0
    ## # ... with 17 more variables: FTA <int>, zFT% <int>, 3PTM <int>, z3PTM <int>,
    ## #   PTS <int>, zPTS <int>, TREB <int>, zTREB <int>, AST <int>, zAST <int>,
    ## #   STL <int>, zSTL <int>, BLK <int>, zBLK <int>, TO <int>, zTO <int>,
    ## #   TOTAL <int>

``` r
Hashtag_proj %>%
  summarise(across(everything(), ~ sum(is.na(.))))
```

    ## # A tibble: 1 x 29
    ##    RANK PLAYER   POS  TEAM    GP   MPG `FG%`   FGM   FGA `zFG%` `FT%`   FTM
    ##   <int>  <int> <int> <int> <int> <int> <int> <int> <int>  <int> <int> <int>
    ## 1     0      0     0     0     0     0     0     0     0      0     0     0
    ## # ... with 17 more variables: FTA <int>, zFT% <int>, 3PTM <int>, z3PTM <int>,
    ## #   PTS <int>, zPTS <int>, TREB <int>, zTREB <int>, AST <int>, zAST <int>,
    ## #   STL <int>, zSTL <int>, BLK <int>, zBLK <int>, TO <int>, zTO <int>,
    ## #   TOTAL <int>

The only NA is in the TEAM column of the rank table, which shouldn’t be
an issue for analysis.

For completeness, though, we can look at the NA directly. Note that I
have to use if\_any rather than across here, because I want to see the
rows where any value is NA, rather than the rows where every value is
NA.

``` r
Hashtag_rank_NA <- Hashtag_rank %>%
  filter(if_any(everything(), ~ is.na(.x)))
Hashtag_rank_NA
```

    ## # A tibble: 1 x 29
    ##   RANK  PLAYER      POS   TEAM  GP    MPG   `FG%`   FGM   FGA `zFG%` `FT%`   FTM
    ##   <chr> <chr>       <chr> <chr> <chr> <chr> <dbl> <dbl> <dbl>  <dbl> <dbl> <dbl>
    ## 1 456   Tim Frazier PG,SG <NA>  10    20    0.302   1.3   4.3  -1.44 0.556   0.5
    ## # ... with 17 more variables: FTA <dbl>, zFT% <dbl>, 3PTM <dbl>, z3PTM <dbl>,
    ## #   PTS <dbl>, zPTS <dbl>, TREB <dbl>, zTREB <dbl>, AST <dbl>, zAST <dbl>,
    ## #   STL <dbl>, zSTL <dbl>, BLK <dbl>, zBLK <dbl>, TO <dbl>, zTO <dbl>,
    ## #   TOTAL <chr>

Tim Frazier was recently released by the Philadelphia 76ers, hence the
NA.

One last step is to convert the following columns to numeric: RANK, GP
(games played), MPG (minutes per game), and TOTAL (sum of z-scores).

``` r
Hashtag_proj <- Hashtag_proj %>%
  mutate(across(c("RANK", "GP", "MPG", "TOTAL"), as.numeric))

Hashtag_rank <- Hashtag_rank %>%
  mutate(across(c("RANK", "GP", "MPG", "TOTAL"), as.numeric))
```

------------------------------------------------------------------------

## Comparing Ranks to Projections

It’s finally time to merge the rank and projection tables by player
name. I did an outer join so I can keep track of any players that are
only in one table.

``` r
Hashtag_merge <- merge(Hashtag_rank, Hashtag_proj, by = "PLAYER", all = TRUE, suffixes = c(".r", ".p"))
```

From there, the first thing I want to look at is the difference in
players’ total value between their current rank and their projection.
This is measured in the TOTAL column, which sums up the respective
z-scores (with a 0.25 weighting on TO, since players that have the ball
more will accrue more turnovers, so their TO rating is less of a
priority).

``` r
Hashtag_merge <- Hashtag_merge %>%
  mutate(TOTAL.diff = TOTAL.p - TOTAL.r)
```

Note that the TOTAL.diff value will be NA for any players that are not
included in the rank or projections column. I’m fine with this, since
anyone not in the projections table likely won’t be fantasy-relevant the
rest of the way. Conversely, anyone not in the rankings table (maybe
they haven’t played yet due to injury) should be looked at separately,
anyway. Comparing their projected TOTAL to zero wouldn’t be too helpful.

With that, let’s have a look at the 25 players whose projected value
most exceeds their current value. The slice\_max function is great for
this. I selected only a few columns for display purposes, and filtered
for players in the top 150 projected value, since I would be unlikely to
target anyone outside of the top 150 for my team.

``` r
Top25 <- Hashtag_merge %>%
  select(PLAYER, POS.r, TEAM.r, TOTAL.diff, RANK.p, MPG.r, MPG.p) %>%
  filter(RANK.p < 150) %>%
  slice_max(TOTAL.diff, n = 25)
Top25
```

    ##                PLAYER    POS.r TEAM.r TOTAL.diff RANK.p MPG.r MPG.p
    ## 1       Thomas Bryant        C    WAS       8.78    142  11.7  25.2
    ## 2  Michael Porter Jr.    SF,PF    DEN       3.75    125  29.8  30.0
    ## 3        Clint Capela        C    ATL       2.75     46  29.8  29.5
    ## 4     Anfernee Simons    PG,SG    POR       2.65     66  26.4  30.3
    ## 5         Jalen Suggs    PG,SG    ORL       2.47    147  27.4  29.2
    ## 6    Kevin Porter Jr. PG,SG,SF    HOU       2.40    130  29.8  31.1
    ## 7        Kyrie Irving    PG,SG    BRO       2.33     18  32.1  33.1
    ## 8        Nikola Jokic     PF,C    DEN       2.10      1  32.6  34.1
    ## 9       Stephen Curry    PG,SG    GSW       2.02      3  34.6  34.0
    ## 10       James Harden    PG,SG    BRO       1.91      2  37.0  35.8
    ## 11      Fred VanVleet    PG,SG    TOR       1.89      6  38.0  37.8
    ## 12     Alperen Sengun        C    HOU       1.89    117  18.3  21.2
    ## 13       Bradley Beal    SG,SF    WAS       1.88     21  36.1  35.6
    ## 14      Klay Thompson    SG,SF    GSW       1.74    106  20.0  26.8
    ## 15        Luka Doncic    PG,SG    DAL       1.63     19  34.6  34.1
    ## 16      Pascal Siakam     PF,C    TOR       1.61     36  36.1  36.5
    ## 17     Damian Lillard       PG    POR       1.47     15  36.5  32.8
    ## 18       Kelly Olynyk     PF,C    DET       1.45    120  23.1  27.2
    ## 19       Monte Morris    PG,SG    DEN       1.44    105  29.6  30.4
    ## 20       Jakob Poeltl        C    SAS       1.43     89  28.7  31.3
    ## 21     Scottie Barnes    SF,PF    TOR       1.41     52  35.5  35.9
    ## 22  Russell Westbrook       PG    LAL       1.40     74  35.3  34.1
    ## 23  Jaren Jackson Jr.     PF,C    MEM       1.38     27  27.6  29.4
    ## 24     Reggie Jackson    PG,SG    LAC       1.28    128  31.4  32.2
    ## 25  Jarred Vanderbilt     PF,C    MIN       1.21     68  25.8  28.8

And now let’s do the same for the bottom 25, using slice\_min.

``` r
Bottom25 <- Hashtag_merge %>%
  select(PLAYER, POS.r, TEAM.r, TOTAL.diff, RANK.p, MPG.r, MPG.p) %>%
  filter(RANK.p < 150) %>%
  slice_min(TOTAL.diff, n = 25)
kable(Bottom25)
```

| PLAYER             | POS.r    | TEAM.r | TOTAL.diff | RANK.p | MPG.r | MPG.p |
|:-------------------|:---------|:-------|-----------:|-------:|------:|------:|
| Brook Lopez        | C        | MIL    |      -2.80 |    109 |  28.2 |  25.1 |
| Malcolm Brogdon    | PG,SG    | IND    |      -2.39 |     82 |  33.7 |  29.1 |
| Josh Hart          | SG,SF    | NOP    |      -2.28 |    135 |  33.0 |  31.7 |
| Seth Curry         | PG,SG    | PHI    |      -2.22 |    107 |  35.0 |  31.2 |
| Al Horford         | PF,C     | BOS    |      -2.00 |     97 |  29.1 |  28.8 |
| LeBron James       | PG,SG,SF | LAL    |      -1.76 |      5 |  36.7 |  33.4 |
| Gary Trent Jr.     | SG,SF    | TOR    |      -1.72 |     98 |  34.0 |  34.1 |
| Alex Caruso        | PG,SG    | CHI    |      -1.60 |    122 |  28.2 |  29.7 |
| Harrison Barnes    | SF,PF    | SAC    |      -1.49 |    121 |  33.5 |  32.1 |
| Anthony Edwards    | SG,SF    | MIN    |      -1.43 |     41 |  35.6 |  35.8 |
| Patrick Beverley   | PG,SG    | MIN    |      -1.26 |    133 |  26.8 |  24.1 |
| DeMar DeRozan      | SF,PF    | CHI    |      -1.22 |     47 |  34.8 |  34.2 |
| Mo Bamba           | C        | ORL    |      -1.20 |     64 |  27.9 |  25.5 |
| Will Barton        | SG,SF    | DEN    |      -1.19 |    123 |  32.5 |  32.5 |
| Zach LaVine        | SG,SF    | CHI    |      -1.18 |     33 |  34.0 |  34.0 |
| Tobias Harris      | SF,PF    | PHI    |      -1.14 |     73 |  35.1 |  33.5 |
| Kyle Kuzma         | SF,PF    | WAS    |      -1.10 |    127 |  33.4 |  32.1 |
| Lonzo Ball         | PG,SG    | CHI    |      -1.09 |     38 |  34.7 |  35.1 |
| Jonas Valanciunas  | C        | NOP    |      -1.08 |     44 |  31.6 |  33.4 |
| Kristaps Porzingis | PF,C     | DAL    |      -1.07 |     23 |  30.5 |  29.5 |
| Derrick White      | PG,SG    | SAS    |      -1.03 |     60 |  30.5 |  33.0 |
| LaMelo Ball        | PG,SG    | CHA    |      -1.03 |     20 |  31.9 |  32.8 |
| Donovan Mitchell   | PG,SG    | UTA    |      -1.02 |     22 |  33.5 |  33.4 |
| Carmelo Anthony    | SF,PF    | LAL    |      -0.99 |    144 |  27.0 |  28.5 |
| Jrue Holiday       | PG,SG    | MIL    |      -0.90 |     43 |  33.2 |  33.5 |

And that’s it! I’ll hold off discussing any of the results. I’m sure the
fantasy basketball managers out there can draw their own conclusions (or
bug me for the underlying data). Whether you’re into fantasy sports or
not, I hope there were some helpful Nuggets in the data cleaning and
results (pun absolutely intended).

------------------------------------------------------------------------
