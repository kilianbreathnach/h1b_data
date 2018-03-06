H-1B visa Labor Condition Application analysis
======================================

This report outlines my approach to completing Elucd's H-1B assignment. In this document, I will explain my approach to understanding, cleaning, and running statistical analysis on the data given in order to answer the questions asked. This document acts as a kind of "lab notebook", while the end results are displayed in jupyter notebooks in the `notebooks` directory.

Data Exploration
-----------------------

The first thing I did, upon downloading the data set, was to use pandas to open the csv file and start looking at what it contained. There was some description on the web page of the various fields and I did some research on government websites about the LCA part of the H-1B process: what it involves and what the meaning behind each data variable was and I started forming ideas about how it might relate to answering the questions.

I looked at the format of the data, what kind of values it had, what kind of values were missing or where it seemed unreliable. A lot of the fields contained text and these fields in particular were very messy, with poor spelling and punctuation and a lot of errors. I could see there would be some serious hurdles to overcome, particularly in figuring out which data points were in NYC and SF, as many of the addresses had been broken down to the neighbourhood level, which meant I would have to search a little harder than just for "NYC" or "SF" data points.

Many of the first steps I took to understanding the data in its raw state can be seen in the notebook named `data exploration.ipynb` in the `notebooks` directory.

Data Cleaning
-------------------

Each field had its own peculiarities which I had to deal with in its own ways and then there were broader issues that required me to make decisions as well. I got bogged down in the company address data for a while before realising that the location data which was more pertinent to a lot of the questions was the work location. But then there were many data points which had multiple work locations and I had to decide how that would be treated for the purpose of the analysis.

####First steps and employer data

Before getting to work location, I decided to clean all the company and other data variables first, since I had already worked through understanding some of it anyway. I did a lot of this in the notebook named `data cleaning 1 - employer.ipynb` where the steps and results can be seen in detail.

I first made the decision to restrict the analysis to only *CERTIFIED* applications which had not been withdrawn. Since this was the LCA data, these applications would be the only ones that could have become valid H-1Bs (if they made it through the lottery and H-1B process). Thus for economic questions, it seemed to be the most relevant.

After figuring out the rules for LCA and how they relate to H-1B, I also omitted applications made too early (more than 6 months before work start) to be used for H-1Bs that were granted in 2014, which is what the dataset purportedly held. In hindsight, I should have checked if these dates were wrong because of some spelling or formatting error, but at the time I trusted that they made since since the `datetime` function had read them correctly. After both of these steps, I had only reduced the dataset by around 10%.

The employer data was much cleaner than the address data in terms of spelling mistakes but there were some employers which were possibly related but had been included in the application with extra or missing words such as "INC", "CO", "LLP" etcetera. Many companies had a large number of applications, so I figured they could be useful for prediction/analysis and I wanted to remove this ambiguity and merge companies that were probably the same. I did this with fuzzy string-matching, using the python `fuzzywuzzy` package, which matches strings by measuring the Levenshtein distance. This is a metric for strings based on the number of insertions and deletions of characters which are necessary to change from one to another. After finding possible duplicates in the company name field, I noticed that some were possibly not duplicates at all, as they could perhaps have similar names or be different parts of a larger company family, e.g. New York University, and New York University School of Medicine. In order to distinguish between these, I cross-correlated matches according to their addresses and considered them the same only if they had the same address (modulo some Levenshtein distance). Some companies have several addresses associated with them in the data set, but this is the best I could do without digging much further.

Matching the addresses themselves was difficult because so many of them were poorly input into the data set. Many zipcode or city fields had street address or office number data, for instance. To overcome this problem, I made a string of the full address data, and matched on that, using the `fuzzywuzzy` "token" score, which matched on similar words in a string. This was probably not a fantastic idea as it didn't buy a whole lot of corrections but took a long time due to the length of the strings.

After finishing the company name and address cleaning, I stored an interim dataframe `clean_naics_employer.csv` in the `dat` directory.

####Job data

For the job fields, I used similar strategies. I looked up the definitions of NAICS and SOC numbers in order to understand what to expect in the data for these values. Some were poorly formatted again and needed to be fixed. The SOC codes were often completely backwards or wrong, but by cross-checking with the SOC name field, I was able to fix many of the numbers and be quite sure which were correct and made sense, at least for the common ones. A lot of this had to be done by hand but now that I know the problem values, I have written code that can do this again automatically. Once I had done this, I restricted the values to only keep the first most informative digits, so as to have less than 100 unique job categories.

I then investigated the job titles and found that there were to many common titles to keep them all as separate variables, particularly because many of the information was redundant with the SOC category. I instead created a new variable to indicate seniority, for those jobs which had higher levels or were management or lead positions. I expected that these would have higher salaries despite being in the same category and could thus be informative without adding an intractable number of dimensions to the data.

Finally, I standardised the pay data by transforming it all into annual rates. I made the approximation that I could multiply monthly pay by 12, bi-weekly pay by 26, weekly pay by 52, and hourly pay by 40 hours by 52 weeks. The resulting interim data frame was stored in `clean_emp_jobs.csv` in the `dat` directory.

####Work location data

For *Task 1*, I decided to include all locations were in an NYC or SF area, both primary and secondary work locations. I found all unique work addresses from states "NY" and "CA" and matched the city names with data I obtained from the NYC and SF open data websites, which are found in the `dat/ext` directory. This I did with fuzzy string-matching again and I had to do some close inspection and a little bit of munging by hand to get rid of or fix problematic addresses.

I saved the interim dataframes separately for NYC and SF in the `dat` directory as `nycdf.csv` and `sfdf.csv`. The process of inspecting and cleaning the data can be seen in the notebook `geocode work location for Task 1 plus more data exploration.ipynb` in the `notebooks` directory. This notebook also contains some data exploration of the city dataframes and first attempts at making hypotheses and getting a solution for *Task 1*.

For *Task 2* I took a similar but somewhat less detailed approach. I decided it would be useful to geoencode placenames to longitude and latitude and make clusters of locations so as to reduce the data dimension. I found some data on the *census.gov* website that offered placenames with their corresponding coordinates. By matching the city and state values in the data set (or fuzzily matched, to account for city name misspells) with the same names in the census data, I could get a set of coordinates for most of the data.

These coordinates were then used with a spatial clustering implementation named DBSCAN, which is intended specifically for latitude and longitude coordinates. By exploring different values for the length scale and group membership size of the clusters, I was able to arrive at a reduction that seemed by inspection to capture the spatial structure of the data well, while keeping the number of clusters relatively low at around 80.

These steps for Task 2 can be seen, along with visualisation of the location clusters, in `geocode and clustering work location for task 2.ipynb`. I stored this interim data in the `dat` directory in the file `clean_jobloc.csv`.

####Standardising and Normalising

Finally I made the finished feature matrix for the analysis by encoding and transforming all the data I had cleaned. This can be seen in `Task 2 - first attempts.ipynb` I dummy-encoded all the class data into binary variable columns. I made cutoffs for employer, at first keeping it to the top ~100. I noticed that the prevailing wage survey source data had not been cleaned yet and proceeded to find the most prevalent survey categories and create dummy variables for them too.

The only numeric data is the salary, dates, number of workers per application, and another number I counted which was the number of unique addresses a particular employer had associated with it. I normalised the number counts and dates to be between 0 and 1, after realising the large numbers may have affected my early attempts at modeling.

For the pay data, I had to make a decision on how to deal with people that worked in multiple locations, since the prevailing wage values were different in each and there were *NaNs* for the second location if a worker only had one location. In order to use standardised (transformed to have zero mean, unit variance) values for the pay, I needed to decided what to do about the missing values. I decided to go with what seemed like an easy first attempt: replace NaNs in the second prevailing wage field with duplicates from the first prevailing wage field, and then, to be safe, add another binary classifier column that tells whether it has one location or not.

The "first attempts" notebook also shows some attempts at modeling the data. I quickly found that my design matrix of >300k rows and >500 columns was too large for my laptop so I moved onto a larger computer that could handle it. The later attempts are in the "first attempts _B" notebook.


Analysis
-----------

The analysis for both tasks is documented in the `Task 1` and `Task 2 - final` notebooks, which give an abridged version of my data cleaning approach and go through the steps in the analysis and their motivations. For *Task 2*, I chose to classify the H-1B applicants into high and low earners and investigate what features, if any, determine which class an applicant falls into.


Conclusion
---------------

I successfully counted the number of H-1B visas applied for in the NYC and SF regions and found them to be very different - a factor of 2 greater in the Bay Area. I came up with three hypotheses to try to determine what causes this difference and used various statistical tests to answer them, with varying success.

In Task 2, I used upper and lower quartiles to determine high and low earners and used a variety of statistical and machine learning modeling approaches to classify applications into these classes. My classifiers reached up to ~95% accuracy in doing this and provided a number of sensible features which can explain why certain applicants earned more or less than others.


#### Things I would do differently or in further work

There are a number of things I pointed out in the cleaning section where I could have improved or at least saved time. Gven more time and/or resources I could definitely improve the analysis. I think there were a lot of issues with geocoding in particular that could have been made easier with the right utilities. Google maps is a much more reliable geocoder than the approach I used but has limits on the number of API requests one can make for free. It would also help with company name too, as it is likely aware of most of them and very tolerant to spelling/formatting errors. In terms of geocoding I also think I could have made more exploration on how my results depended on the cluster parameters I chose.

In Task 1, I could have posed my second question a lot better, knowing that the counts were likely to have a median at such small numbers. This made the K-S test kind of unjustified, due to its assumptions. I think the results still gave me an answer that was intelligible though. I think my final question could easily be investigated further to see what proportion specifically of applications in SF are due to the correlation I found, which could be interesting to do as future work.

In Task 2, I would be interested to see if different quantile cuts could produce more accurate or more intelligible results. There is no clear distinction between what counts has high or low earners so there's no reason to believe that anything much closer to 100% accuracy could be achieved unless you dealth with clear outliers. However, I'm sure a less interpretable model that exposes more non-linear features, such as a neural network, could probably manage some gain on my classification accuracy. I would certainly be interested in investigating such implementations with further attempts.

