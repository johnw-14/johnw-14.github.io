---
layout: post
title: "Correlation of NBA Fantasy Statistics"
always_allow_html: yes
output: 
  md_document: 
    variant: gfm
    preserve_yaml: true
---

------------------------------------------------------------------------

## Introduction

Today’s post will continue with the NBA fantasy basketball projections I
used [last
time](https://johnw-14.github.io/2022/01/16/NBA-FANTASY.html). There is
plenty more I want to examine now that the data is nice and tidy. If you
are new to fantasy basketball, I included a short background in the
previous post.

Using the same player projections from [Hashtag
Basketball](https://hashtagbasketball.com/fantasy-basketball-projections),
I want to look at how the nine statistical categories in my NBA fantasy
league correlate to one another. I often find myself targeting a few
categories at once, either to chase wins for a week, or to improve my
team long-term. But I’m curious which categories typically go
hand-in-hand, and which can be rare combinations. For example, one would
expect players that score a lot of three-pointers (3PTM) to also rank
highly in total points (PTS), since three-pointers are worth the most.
Conversely, I’d expect those same players to typically have lower field
goal percentages (FG%), since three-pointers are harder to make. This
makes three-point shooters with strong field goal percentages very
valuable.

So today’s task is simple. First, compute and visualize the correlations
between the nine statistical categories. I will do this using players’
projected z-scores in each category, because I specifically want to look
at the correlations for the rest of this NBA season. With that in hand,
I want to identify players that are strong in two categories that don’t
normally correlate.

------------------------------------------------------------------------

## Load Packages in R

``` r
library(tidyverse)
library(knitr)
library(corrplot)
```

------------------------------------------------------------------------

## Calculating and Plotting Correlation

Following all the data cleaning steps from [last
time](https://johnw-14.github.io/2022/01/16/NBA-FANTASY.html), I have a
projections table with columns of players’ z-scores for each statistical
category. I denoted these with the prefix “z”. So first off, I’ll make a
dataframe of just those columns.

``` r
proj_z_scores <- select(Hashtag_proj, starts_with("z"))
```

With that done, I can calculate the correlation between each column by
simply applying the cor function to the entire dataframe.

``` r
proj_z_corr <- cor(proj_z_scores, method = "spearman")
```

You’ll notice I specifically set the method to Spearman’s rank
correlation coefficient. The default for the function is Pearson. But
Pearson requires that the variables are normally distributed, which is
an issue here.

Let’s take a look at the distribution of z-scores for points for
example:

``` r
ggplot(proj_z_scores, aes(x = zPTS)) + 
  geom_histogram() + 
  ggtitle("Distribution of NBA players' projected \n z-scores for total points per game")
```

![](/images/zPTS_distribution-1.png)<!-- -->

We see that it’s quite right-skewed (a long tail of players with high
z-scores), which makes sense. There are many players who play relatively
low minutes and don’t score a whole lot, and then you have a long tail
of strong to very strong scorers.

This is also quite pronounced for a category like blocks, where most
players get one or fewer per game.

``` r
ggplot(proj_z_scores, aes(x = zBLK)) + 
  geom_histogram() + 
  ggtitle("Distribution of NBA players' projected \n z-scores for blocks per game")
```

![](/images/zBLK_distribution-1.png)<!-- -->

Given that, we proceed with our calculation of Spearman’s correlation
coefficient, which doesn’t require that the variables have normal
distributions.

I can now use the handy corrplot function (from the package of the same
name) to plot everything. I added a few arguments to suit my preference.

``` r
corrplot(proj_z_corr, 
         method = "circle", 
         type = "upper", 
         diag = FALSE, 
         tl.col = "black")
```

![](/images/9cat_corrplot-1.png)<!-- -->

Let’s start off with some of the strongest correlations. As expected,
three-pointers and total points show a strong correlation. Those two
categories also correlate strongly with assists and steals. There are
also correlations with rebounds and blocks mixed in. It’s important to
note that all of these categories are total counts of their statistic.
So if a player plays a lot of minutes, they will tend to score higher
across all of the total categories.

With that said, we do see two sub-groups of the strongest correlations.
The first group is three-pointers, points, assists and steals. The
second is rebounds and blocks. Those match with traditional player roles
in basketball. Smaller plays tend to shoot more threes and pass the
ball, while larger players operate closer to the rim and collect
rebounds and blocks. Obviously there are lots of exceptions, which is
why I’m looking at this in the first place.

What’s more interesting to me is the negative correlations. I’m going to
mostly ignore turnovers here, since it’s well known players that have
the ball more are going to turn it over more frequently. We see that
represented in the chart from the strong negative correlations between
turnovers and “ball-heavy” categories like points and assists.

Starting at the top left, we see a couple interesting slightly negative
correlations. I have heard that larger players often struggle with free
throws. They also traditionally play in positions that operate close to
the net and would tend towards higher field goal percentages. We see
that negative correlation between FG% and FT% bear out in the chart. I
recently added a player who shoots well in both categories, so this is
pleasantly affirming. Similarly, we see a slight negative correlation
between blocks and free throw percentage. As I alluded to in the
introduction, we also see a negative correlation between field goal
percentage and three-pointers made, which makes sense.

Of course, if we were being thorough we would also want to test the
statistical significance of the correlations. But for my purposes those
slightly negative correlations are enough for today.

------------------------------------------------------------------------

## Identifying Players

Armed with this knowledge, let’s see which players buck those
statistical trends. First up, let’s see the players within the top 200
of Hashtag Basketball’s projections that have z-scores greater than 0.5
in both field goal and free throw percentage.

``` r
FG_FT_0.5 <- Hashtag_proj %>%
  select(PLAYER, POS, TEAM, RANK, 
         `FG%`, `zFG%`, `FT%`, `zFT%`) %>%
  filter(RANK < 200, 
         `zFG%` > 0.5, 
         `zFT%` > 0.5)
kable(FG_FT_0.5)
```

| PLAYER             | POS   | TEAM | RANK |   FG% | zFG% |   FT% | zFT% |
|:-------------------|:------|:-----|-----:|------:|-----:|------:|-----:|
| Nikola Jokic       | PF,C  | DEN  |    1 | 0.561 | 3.49 | 0.806 | 0.76 |
| Kevin Durant       | SF,PF | BRO  |    3 | 0.515 | 2.37 | 0.877 | 3.19 |
| Joel Embiid        | PF,C  | PHI  |    4 | 0.519 | 2.10 | 0.829 | 2.85 |
| Karl-Anthony Towns | C     | MIN  |    8 | 0.505 | 1.55 | 0.828 | 1.17 |
| Jimmy Butler       | SF,PF | MIA  |   10 | 0.516 | 1.64 | 0.860 | 3.02 |
| Chris Paul         | PG    | PHX  |   17 | 0.490 | 0.54 | 0.843 | 0.82 |
| Kyrie Irving       | PG,SG | BRO  |   26 | 0.492 | 1.09 | 0.904 | 1.62 |
| Zach LaVine        | SG,SF | CHI  |   37 | 0.485 | 0.93 | 0.843 | 1.50 |
| John Collins       | PF,C  | ATL  |   38 | 0.539 | 1.68 | 0.825 | 1.00 |
| DeMar DeRozan      | SF,PF | CHI  |   47 | 0.481 | 0.78 | 0.838 | 2.08 |
| LaMarcus Aldridge  | PF,C  | BRO  |   88 | 0.573 | 2.09 | 0.847 | 0.58 |
| Seth Curry         | PG,SG | PHI  |  101 | 0.527 | 1.48 | 0.857 | 0.64 |

Not surprisingly, most of the players who fit the criteria fall inside
the top 50 overall. It’s part of what makes them so good. It’s also fun
to see LaMarcus Aldridge in there, since he’s the player I recently
added.

Next up, let’s see the players with strong field goal percentages and
three-pointers made (I’ll hazard a guess that we’ll see some similar
names).

``` r
FG_3PTM_0.5 <- Hashtag_proj %>%
  select(PLAYER, POS, TEAM, RANK, 
         `FG%`, `zFG%`, `3PTM`, z3PTM) %>%
  filter(RANK < 200, 
         `zFG%` > 0.5, 
         z3PTM > 0.5)
kable(FG_3PTM_0.5)
```

| PLAYER             | POS      | TEAM | RANK |   FG% | zFG% | 3PTM | z3PTM |
|:-------------------|:---------|:-----|-----:|------:|-----:|-----:|------:|
| Kevin Durant       | SF,PF    | BRO  |    3 | 0.515 | 2.37 |  2.3 |  0.68 |
| LeBron James       | PG,SG,SF | LAL  |    6 | 0.523 | 2.51 |  2.8 |  1.27 |
| Karl-Anthony Towns | C        | MIN  |    8 | 0.505 | 1.55 |  2.6 |  1.04 |
| Kyrie Irving       | PG,SG    | BRO  |   26 | 0.492 | 1.09 |  2.5 |  0.89 |
| Zach LaVine        | SG,SF    | CHI  |   37 | 0.485 | 0.93 |  2.6 |  1.05 |
| Seth Curry         | PG,SG    | PHI  |  101 | 0.527 | 1.48 |  2.2 |  0.60 |

This exercise is a great look for Seth Curry. Finally, let’s try free
throw percentage and blocks:

``` r
FT_BLK_0.5 <- Hashtag_proj %>%
  select(PLAYER, POS, TEAM, RANK, 
         `FT%`, `zFT%`, BLK, zBLK) %>%
  filter(RANK < 200, 
         `zFT%` > 0.5, 
         zBLK > 0.5)
kable(FT_BLK_0.5)
```

| PLAYER                  | POS   | TEAM | RANK |   FT% | zFT% | BLK | zBLK |
|:------------------------|:------|:-----|-----:|------:|-----:|----:|-----:|
| Nikola Jokic            | PF,C  | DEN  |    1 | 0.806 | 0.76 | 0.9 | 0.75 |
| James Harden            | PG,SG | BRO  |    2 | 0.872 | 4.56 | 0.9 | 0.68 |
| Kevin Durant            | SF,PF | BRO  |    3 | 0.877 | 3.19 | 0.9 | 0.83 |
| Joel Embiid             | PF,C  | PHI  |    4 | 0.829 | 2.85 | 1.4 | 1.90 |
| Karl-Anthony Towns      | C     | MIN  |    8 | 0.828 | 1.17 | 1.1 | 1.22 |
| Jaren Jackson Jr.       | PF,C  | MEM  |   18 | 0.839 | 1.28 | 2.6 | 4.92 |
| Anthony Edwards         | SG,SF | MIN  |   20 | 0.822 | 0.81 | 0.9 | 0.61 |
| Kristaps Porzingis      | PF,C  | DAL  |   23 | 0.886 | 2.50 | 1.5 | 2.24 |
| Shai Gilgeous-Alexander | PG,SG | OKL  |   36 | 0.837 | 2.08 | 0.9 | 0.80 |
| John Collins            | PF,C  | ATL  |   38 | 0.825 | 1.00 | 1.0 | 1.02 |
| Jerami Grant            | SF,PF | DET  |   52 | 0.864 | 2.25 | 1.0 | 1.02 |
| Derrick White           | PG,SG | SAS  |   57 | 0.857 | 1.19 | 1.0 | 0.93 |
| LaMarcus Aldridge       | PF,C  | BRO  |   88 | 0.847 | 0.58 | 1.1 | 1.19 |
| Brook Lopez             | C     | MIL  |  103 | 0.847 | 0.53 | 1.5 | 2.22 |

It’s fun to see my first round pick Karl-Anthony Towns showing up on
each list. This will definitely shape how I adjust my roster over the
rest of the season as well. I will wrap things up there. Thanks for
reading, as always.
