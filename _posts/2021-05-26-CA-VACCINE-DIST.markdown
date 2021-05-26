---
layout: post
title:  "Algorithmic COVID-19 Vaccine Distribution in California"
date:   2021-05-26
---
I saw an interesting (and concerning) case of algorithmic decision-making on Twitter a couple weeks ago. The [thread](https://twitter.com/snowjake/status/1390349304246665218) and [blog post](https://www.aclunc.org/blog/californias-equity-algorithm-could-leave-2-million-struggling-californians-without-additional) are from Jabob Snow, a Technology and Civil Liberties Attorney at the ACLU of Northern California. He details the COVID-19 vaccine allocation approach of the California Department of Public Health. From the blog post: 

>“To advance the goal of equity in vaccine distribution, the state decided in January to use something called the “Healthy Places Index”—a metric that assigns scores to communities across California according to health outcomes—in order to identify areas of the state where additional supply of the vaccine is necessary. Under the state's plans, additional vaccine supply would go to the areas with Healthy Places Index scores in the bottom 25%.
>
>A few weeks later, the state announced that Blue Shield would build an algorithm allocating vaccines based on ZIP codes rather than census tracts—the generally smaller, census-based areas that the Healthy Places Index scores with health outcomes.”

This is an immediate issue, because as the article points out:

>“ZIP codes often represent much larger geographical areas, containing both low-income and wealthy communities. As a result, a low-income, underserved neighborhood with a very low Healthy Places Index score can end up erased from the state’s priority list by virtue of the fact that they are in the same ZIP code with higher-income households.”

There is an excellent map in the article where you can see the effects of using ZIP codes instead of census tracts. The decision leaves out approximately 2 million people living in “high priority” census tracts, but “low priority” ZIP codes.

This is a clear example of how seemingly innocuous choices in an algorithm’s design can have significant impacts on the individuals affected. The state’s response, according to the article, was that “tracking vaccine delivery to ZIP codes is operationally simpler than using census tracts”. There is obviously a difficult balance to strike here between prioritizing vulnerable communities for vaccination, and allocating vaccines as quickly as possible.

It’s also a clear example why transparency around an algorithm’s design AND its deployment is crucial to understand its impact (the article gives the state credit for this). Without knowledge of how the Healthy Places Index is assigned, or how Blue Shield decided to proceed with their own algorithm, the issue could not have been raised in the first place. It also highlights the importance of listening to the voices of those most affected by the technology.

Vaccine allocation is also in the news in Calgary, Canada where I am currently staying. Last week the province opened up vaccine booking for everyone ages 12+. The [early data](https://twitter.com/ab_vax/status/1394750935520346112) has shown that vaccination rates in high-income neighbourhoods are quickly outpacing low-income ones (the ACLU blog post also mentioned [this study](https://www.medrxiv.org/content/10.1101/2021.03.25.21254272v2)  on the age-based approach). So the algorithmic approach in California appears to be well-intentioned and rational, but there are still crucial decisions that determine how equitable the vaccine distribution truly is.






