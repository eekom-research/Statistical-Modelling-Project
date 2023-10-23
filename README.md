# Statistical Modelling with Python

## Project Goals
The overarching goals of the project are to practice data analysis
and statistical modelling using Python. Detailed learning 
objectives include, but are not limited to,

* Accessing data using APIs,
* Cleaning and transforming data using Python,
* Loading data into a database using Python,
* Performing EDA, including using both statistics and visualizations,
* Identifying trends and patterns in data using statistical models,
* Interpreting the results of the statistical models.


## Process
To achieve the above learning objectives, data will be sourced from the City Bikes API and then
enriched with details from location services providers such as Foursquare and Yelp. Simple statistical
models such as linear regression and multinomial classification will then be applied on these
enriched datasets for the purposes of identifying patterns and building predictive capabilities. The
specific detailed processes are covered as follows.

### Obtaining Data from the City Bikes API
First, the `networks` endpoint was queried to get information about the bike networks coverage
for the City Bikes API. 

```python
city_bike_base_url = 'http://api.citybik.es'
networks_endpoint = '/v2/networks'
# define helper function to make to avoid repeated code for subsequent calls
def get_city_bike_data(base_url,endpoint,params=None):
    url = city_bike_base_url + endpoint
    return requests.get(url,params=params)
response = get_city_bike_data(city_bike_base_url,networks_endpoint)
result = response.json()
```
The returned payload was then processed to explore the relative distribution of networks across cities.
Berlin, as a representative city with one of the highest coverage, was then selected as the choice
location for the remainder of this project. 

```python
df_networks= pd.json_normalize(result['networks'], sep='_') #sep used to format column names during flattening
df_networks['location_city'].value_counts().head(10)
df_networks.loc[df_networks['location_city']=='Berlin',:]
Berlin_endpoints = df_networks.loc[df_networks['location_city']=='Berlin',:]['href'].to_list()
```
For each of the `endpoints` in `Berlin_endpoints`, the City Bikes API was then queried to get all the
bike stations. The results were then processed and information about the unique station id, name of the
station, number of bikes in the station and the station location details (latitude and longitude coordinates)
were retained. The processed data was then exported to a csv file for onward processing.

### Enrich City Bikes Datasets with Foursquare and Yelp Location Services

Using the `latitude` and `longitude` coordinates of the bike stations from the previous step, Foursquare
and Yelp location services were queried to get data about the points of interests in three 
categories (`Restaurant`,`Bar`, and `Fashion Retail`) that are within 1000 metres from each bike station.
In order to maintain uniformity in the collected attributes, effort was made to ensure that all fields
are consistent between the two sources. The following code snippets show the procedures for querying
the Foursquare API and then processing and formatting the returned payload. The procedure is done
similarly for Yelp.
```python
# build a helper function to avoid repeated calls
def get_venues_fs(latitude, longitude, radius,categories=None):
    """
    Get venues from foursquare with a specified place type and coordinates.
    Args:
        latitude (float): latitude for query (must be combined with longitude)
        longitude (float): longitude for query (must be combined with latitude)
        api_key (str): foursquare API to use for query
        categories (str) : Foursquare-recognized place type. If not passed no place_type will be specified. Separate ids with commas
    
    Returns:
        response: response object from the requests library.
    """
    url = 'https://api.foursquare.com/v3/places/search'
    
    params = {
    'll':f"{latitude},{longitude}",
    'radius':str(int(radius)),
    'categories':categories,
    'fields':'fsq_id,categories,name,location,rating,distance,stats,price'
    }
    
    headers = {
    'Authorization': str(FOURSQUARE_KEY)
    }
    response = requests.get(url,params=params,headers=headers)
    return response
```
For parsing the returned payload, we have:
```python
# build helper function to parse the json output for the POIs
def parse_fs_response(response_json,reference_station_id):
    store_record = []
    for poi in response_json['results']:
        poi_id = poi.get('fsq_id',None)
        poi_name = poi.get('name',None)
        poi_distance = poi.get('distance',None)
        poi_source = 'foursquare'
        poi_rating = poi.get('rating',None)
        poi_location = poi.get('location',{}).get('formatted_address',None)
        poi_price_level = poi.get('price',None)
        poi_total_reviews = poi.get('stats',{}).get('total_tips',None)
        fs_categories_keys = [key for key in fs_categories.keys()]
        poi_primary_category = ""
        for category in poi['categories']:
            if str(category['id']) in fs_categories_keys:
                poi_primary_category = fs_categories[str(category['id'])].strip()
                poi_sub_category = category.get('name',None)
                break
        if not poi_primary_category:
            poi_primary_category = 'Unknown'
            poi_sub_category = np.nan
        record = {
        'poi_reference_station_id':reference_station_id,
        'poi_id':poi_id if poi_id is not None else np.nan,
        'poi_name':poi_name if poi_name is not None else np.nan,
        'poi_distance':poi_distance if poi_distance is not None else np.nan,
        'poi_source':poi_source,
        'poi_rating':round((float(poi_rating)/10)*100) if poi_rating is not None else np.nan,
        'poi_price_level':poi_price_level if poi_price_level is not None else np.nan,
        'poi_total_reviews':poi_total_reviews if poi_total_reviews is not None else np.nan,
        'poi_location':poi_location if poi_location is not None else np.nan,
        'poi_primary_category':poi_primary_category,
        'poi_sub_category':poi_sub_category if poi_sub_category is not None else np.nan,
        }
        store_record.append(record)
    return store_record
```
The parsed response is then placed in a DataFrame and exported to csv for onward processing. However,
due to time restrictions and restrictive API limits on the free tier for the location services providers,
the number of unique bike stations for which POIs information was sought was limited to 300. This decision
had marked implications for the remainder of this project. This will be discussed in later sections.

### Building the Data Model and Database Interactions
The database data model design in this section is guided by two main paradigms:
1. Each table (or entity) in the database should contain only attributes specific to a single theme
2. Every attribute in a single table must depend on the primary key(s) and nothing but the whole primary key(s).
Hence, three tables will be created.

![Database ERD](https://github.com/eekom-research/Statistical-Modelling-Project/blob/main/images/bike_data_ERD.png)

3. bike_station_info --> |'station_id'|'station_name'|'latitude'|'longitude'|'free_bikes'|
4. bike_station_poi_info --> |'station_id'|'poi_id'|'distance_from_station'|
5. poi_info --> |'poi_id'|'poi_name'|'poi_source'|'poi_rating'|'poi_price_level'|'poi_total_reviews'|
                |'poi_formatted_address'|'poi_primary_category'|'poi_sub_category'|

I I am working on a statistical modeling project to get my first real practice 
with working with data and building simple statistical models 
like linear regression and logistic regression.

I have firstly collected data from city bikes api, 
collecting data about bike stations in berlin, 
the number of bikes they have available, 
and their latitude an longitude. 
Using the location data(ll), 
I have queried two  location services 
 Foursquare and Yelp to get data about 
 points of interests in three categories (restaurant,bars, and fashion retail)
 that are within 1000m from each bike station to enrich the dataset.  
 In other to maintain uniformity in attributes, 
 I have ensured that all fields are consistent between the two sources.

The data about the POIs are then joined to the city bike data on the unique id of the bike stations. 
Since the goal is to build a simple stats model for the data, 
I have removed some identifying columns, which while useful in other contexts, 
may not neccesarily add much to the problem at hand. 
I have removed information such as the bike stations id, name,
 poi_id, poi_name and poi_formated_address.

I am now left with the following columns: 
the number of free_bikes in each bike station,
 the latitude and longitude of the bike station, 
 the distance of each POI from the bike station, 
 the source of the POI information, the POI rating, 
 price level, total reviews, primary and sub categories.  
 For these remaining categories, I have done some preprocessing.
 I have dropped rows with an overall poi rating, 
 as I feel the poi rating will most likely be the 
 dependent variable for my modeling purpose. 
 Hence this is critical. For missing values on price level (which depicts affordability),
 I have filled those with the category median values because the price level data seemed skewed, 
 making median appropriate. The price level missing values are 
 12 percent of the total data but a significant proportion of 
 some categories such as bars and fashion retail. 
 This is because the initial data contains a 
 disporportionate amount of POIs in the restaurant business. 
 Finally, information like category and data source seems 
 to be relevant to the missing price level information,
 therefore, I have filled while taking into cognizance group level median values.

NOw I want to start some exploratory data analysis 
on this processed data using seaborn.
 The aim is to find some initial patterns 
 or relationship in the current dataframe.
 I want you to help with some structured way
 or checklist for proceeding with this EDA. 
 The essence is not to be painstakingly thorough,
 as this is only a weekend learning project. 
 But I would want the steps/checklist/categories of 
 EDA to be well structured and representative considering the end goal. 
 Basically, I want some process that I would be able to remember 
 and follow through when I continue learning more about the process.
### (your step 1)
### (your step 2)

## Results
(fill in what you found about the comparative quality of API coverage in your chosen area and the results of your model.)

## Challenges 
(discuss challenges you faced in the project)

## Future Goals
1. The limited time and API limits on the free tier of the location services (Foursquare and Yelp)
placed a ceiling on the number of data points that could conceivably be analyzed. As such, the analysis 
was limited to the point of interests in three pre-selected categories that are within a radius
of 1000 m of 300 unique bike stations in Berlin. It would be interesting to
repeat the process with significantly more data points.
2. Model building and data analysis is an inherently iterative process.
Hence, it is important to revisit assumptions made during the data collection and processing
stages, given the outcome of a derived model. As this can directly affect the quality of the model,
more time is needed to improve model accuracy.
3. The decision to harmonize the details collected for each POI was made early in the process.
With more time, relevant details exclusively available in one location provider but that could be reasonably
estimated for the other would be collected, thereby increasing the richness
of the datasets.
4. Raw location data like latitude and longitude of the POIs
can be encoded into more useful forms like postal/zip codes. With such information,
insights about location clusters could enrich the datasets. More time
would allow for such encoding and analysis.


(what would you do if you had more time?)
