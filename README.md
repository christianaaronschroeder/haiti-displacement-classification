**DS 6030 \| Spring 2021 \| University of Virginia**

**By Christian Schroeder**

Full Report: https://rpubs.com/christianaaronschroeder/788860


------------------------------------------------------------------------

# Introduction

In early 2010 the Caribbean nation of Haiti was devastated by a magnitude 7.0 earthquake. This catastrophe leveled many buildings, and resulted in numerous lives lost. Most people around the world are familiar with this disaster and its level of destruction, but few are as familiar with the after-effects it had on those that lived but their homes didn't.

In the wake of the earthquake, an estimated five million people, more than 50% of the population at the time, were displaced, with 1.5 million of them living in tent camps (<https://www.worldvision.org/disaster-relief-news-stories/2010-haiti-earthquake-facts>). This wide-spread displacement of people across a country with worsened infrastructure made relief efforts more difficult. Teams needed an accurate way to locate these individuals so they could provide aid.

In an effort to assist the search, a team from the Rochester Institute of Technology (RIT) collected aerial imagery of the country. These images were then converted into datasets of Red, Green, and Blue (RGB) values. Using this RGB data from the imagery, with the knowledge that many of the displaced people were using distinguishable blue tarps as their shelter, I attempted to predict the locations of these blue tarps using several classification models.

My goal in this analysis was to determine the optimal model for locating displaced people. To determine those models, I focused on two statistics; accuracy, and false negative rate (FNR). Given the context of the situation, I believed the FNR to be a very important metric, much more than the false positive rate (FPR), because I wanted to make sure no displaced individual was being overlooked. I would much rather have over-classified and found no one at a certain location than under-classify and not provide aid to someone in need. But, it is important to note that these efforts still needed to be made in a timely manner, so grossly over-classifying to get the smallest FNR was not the optimal solution. So, a combination of accuracy, FNR, and FPR was used to determine these models.
