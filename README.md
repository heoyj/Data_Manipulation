# On-time Performance Analysis Project 

## Yelp rating in Las Vegas in terms of cleariness

## 1. Introduction 

This project will be about investigation of relationship between Yelp rating and cleanliness.  After E.coli outbreak in Chipotle, customers seem to be more likely to consider cleanliness as well as restaurant’s stars in google map or Yelp. From my personal experience, it was not successful to choose a restaurant when I went to Las Vegas during a trip. Based on stars, it should have been nice and clean restaurant. It reality, it was quite disappointed in terms of tininess and cleanliness. In this aspect, it’s reasonable to think that someone may check the cleanliness when they select the restaurants. That is why Yelp launched the health score that can be shown in the search result and so that it provides a kind of measurement of cleanness for the restaurants. Currently, only 17 cities provide the scores in Yelp now. Moreover, many researches have been conducted on this topic with different approaches, such as text mining, machine learning, statistical test, and regression, even with unlisted datasets. Therefore, this project investigates the relationship between Yelp rating and food inspection result in Las Vegas, which is not set up for health score yet in Yelp. In order to see the relationship, focusing on the Yelp rating (stars) and the food inspection grade. If there is a significant relationship, creating a new measurement, which incorporates Yelp rating and food inspection grade and visualizing on the interactive map with the new score will be implemented. However, if not, only visualization will be done. 

## 2. Data Source

Two different datasets were obtained from the following resources:

### 2.1 Food Inspection Result in Las Vegas 

This dataset include the information about food inspection result at local stores located in Las Vegas. The file is called _‘Restaurant_Inspections_LV.csv’_ and it can be downloaded from <https://www.opendatanetwork.com/dataset/opendata.lasvegasnevada.gov/q8ye-5kwk>. The original file size is 34.2 MB and contains 113,416 observations with 23 variables. Among those variables, 6 variables were used for this project. The variables are: City, Zip, Permit Number, Restaurant Name, Inspection Grade, and Inspection Date. This dataset covers the inspection result conducted from 2010 to 2015. 

### 2.2 Yelp Business Dataset

Original Yelp dataset is about the local businesses information in 10 cities across 4 countries. In order to address questions, the file, called _‘yelp_academic_dataset_business.json’_, is withdrawn from a tar formatted file _‘yelp_dataset_challenge_academic_dataset.tar’_. It can be obtained from Yelp Dataset Challenge website. In <https://www.yelp.com/dataset_challenge>, scroll down a little bit and you can see “Get the Data” button. Click and fill up the form in order to access the full dataset. Then, the .tar file can be downloaded in the next page. After unzipping the .tar file, there is _‘yelp_academic_dataset_business.json’_ file. This .json file is 77.3 MB and consists of 85,901 observations that include 15 variables. In this project, 8 variables such as city, full_address, categories, latitude & longitude, name, stars and state are used. 

## 3. Data Manipulation Methods

In order to investigate the relationship between Yelp rating and food inspection grade, Yelp rating and food inspection grade for each store should be matched by one-to-one. For achieving the goal, two dataset have to be combined using a primary key. 

### 3.1 Primary Key : index in each dataset

Although there are Serial/permit number in Food_Insp and business_id in Yelp that are unique for each store or inspection result, they are not a common variable for both. Therefore, the alternative candidates of the primary key were narrowed down to the following: pairs of latitude & longitude, store name, address and index. Finally, the index was chosen to be the primary key and name and address were actually used while generating the pairs of indices for mapping each dataset. 

The reason that index was chosen to be the primary key is the following. Since latitude and longitude pair are different in each dataset even though the store is the same. Also, two stores where are located in next to each other share the same lat & long coordinate. Therefore it would not be a unique identifier for the store in two dataset. Another possibility is the address of store. However, the string itself is longer than store name and there are a number of modifications, so that it would take more time to match the stores in both dataset. Thus, it would be hard to implement ‘SequenceMatcher’ function, which will be described in later, since this modification are more likely to give incorrect matching. Store name seems the best primary key at first. However, franchise stores share the same store name so it can produce the duplicate matching that shouldn’t be when combining. Then, extra filtering would be required, that’s why name itself was not used as primary key. However, during building up primary key pairs in mapping SQL table that connects both dataset, the pair was generated from matching store name within the same zip code area. Therefore, primary key is index and the pair of indices that make a connection between two datasets is generated using SQLite, it therefore used while doing left join. Detail is discussed in 3.4. 

From now on, the manipulation of dataset is explained so that what kind of manipulation were done in order to make the pair of indices in mapping table. While describing, the abbreviated names of each dataset are used. Food_Insp stands for food inspection result dataset, and Yelp does for Yelp dataset.

### 3.2 Manipulation in Food_Insp dataset

1) Since the file is formatted as .csv, it is read into DataFrame using pandas. 
2) The result covers all the local area of South Nevada; ‘City’ named as Las Vegas filtered out the dataset.
3) For each store, the inspection test was conducted multiple times and therefore the latest result within a store was kept. Since ‘Inspection Date’ was originally sorted by increasing order (the recent date) within each ‘Permit Number’, this was used to drop the previous tests observations. 
4) In order to show the inspection date at the visualization step, only date part was extracted from ‘Inspection Date’ that is formatted as datetime.
5) Also, only first 5 digits in zip code were extracted from ‘Zip’ variable, that were used when build up the primary key.

### 3.3 Manipulation in Yelp dataset

1) The file was formatted as .json and it has a ValueError : Trailing data when using json.dump or json.load functions. There was a way to read the .json file, that is, read the lines, join separated by comma and make them into dataframe. 
2) After loading .json file, filtered out the observations based on the ‘state’ and ‘city’: NV and Las Vegas. Also, the dataset only keeps the observation which has the category as restaurant, bar, and food. (Full list of categories are found here: <https://www.yelp.com/developers/documentation/v2/category_list>)
3) The observations, which are only relevant to the food inspections, were kept in the last dataset. 
4) The first 5 digits of zip code were extracted for being used while building up the primary key. Full address is shortened, that is, city name, state, zip code were removed for being used in comparing the addresses so that validate the matching results after joining two dataset. 

### 3.4 Combining two dataset

For each restaurant name in Yelp, found the observation(s) from Food_Insp dataset, but within the same zip code area, with high ratio of matching name and returned the index of Food_Insp dataset. The reason why made it return the index, instead of the name, is that there are franchise restaurants with the same name and zip code, then joining datasets will generate duplicate rows which shouldn’t be. In order to avoid this, a user-defined function, called ‘matchingStore’, returns the index/indices for matched observation in Food_Insp dataset. 

If we got the pairs of indices in a list of tuples, which consists of one from Yelp, the other from Food_Insp, it can be used as a mapping table between Yelp and Food_Insp datasets. Using SQLite, they were combined together using left join and validated by comparing the first digits of street numbers. If there was no matching case, then it was removed from the final joined dataset. For further analysis, the inspection grades, which were coded as char type, were converted as number. According to the inspection grade, give the 5 to 3 numbers for ‘A’, ‘B’, and ‘C’. The meaning and the interpretation of the grade will be discussed in Analysis section later. 

While combining the two datasets, the unmatched cases were deleted and there was no incomplete observation for the final dataset. Thus, it was okay to go with it in analysis.  The main process was focused in building up the primary key based on the same zip code, store name and their address. While building up the mapping table, ‘SequenceMatcher’ in ‘similar’, ‘matchingStore’ and ‘matchingIdx’ played an important role and regular expression & SQLite were applied as well.




