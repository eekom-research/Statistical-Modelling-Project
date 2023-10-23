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

Declarative Object Relational Mapping (ORM) techniques will be used to model the tables in Python,
which will then
be pushed to an SQLite file. This is important especially if there is an intention to continue 
further developments  of the model in Python. Advantages include having the same consistent 
model design accessible from both within your app and the external database and have a smoother
integration with the database, which can potentially quicken development time after the initial
design stage. Hence, I believe the syntax overhead is fully justified. The following code snippets show
the design with `sqlalchemy` framework in Python.

```python
# imports necessary to create a declarative entity model using sqlalchemy 2.0
import sqlalchemy as sqla
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import MappedAsDataclass
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy import ForeignKey
from typing import Optional
```
The engine creation and declarative mapping table design syntax are shown as follows.

```python
# create and connect to a local SQLite database
engine = sqla.create_engine("sqlite:///../data/bike_data.db")
# create the base class for table models
class Base(DeclarativeBase, MappedAsDataclass):
    pass
# build bike_station_info table model
class BikeStationInfo(Base):
    __tablename__ = 'bike_station_info'
    station_id: Mapped[str] = mapped_column(primary_key=True)
    station_name: Mapped[Optional[str]]
    free_bikes: Mapped[Optional[int]]
    latitude: Mapped[float]
    longitude: Mapped[float]
# build bike_station_poi_info table model
class BikeStationPoiInfo(Base):
    __tablename__ = 'bike_station_poi_info'
    station_id: Mapped[str] = mapped_column(ForeignKey('bike_station_info.station_id'),primary_key=True)
    poi_id: Mapped[str] = mapped_column(ForeignKey('poi_info.poi_id'),primary_key = True)
    distance_from_station: Mapped[Optional[float]]
# build poi_info table model
class PoiInfo(Base):
    __tablename__ = 'poi_info'
    poi_id: Mapped[str] = mapped_column(primary_key=True)
    station_id: Mapped[str] = mapped_column(ForeignKey('bike_station_info.station_id'))
    poi_name: Mapped[Optional[str]]
    poi_source: Mapped[Optional[str]]
    poi_rating: Mapped[Optional[float]]
    poi_price_level: Mapped[Optional[float]]
    poi_total_reviews: Mapped[Optional[float]]
    poi_formatted_address: Mapped[Optional[str]]
    poi_primary_category: Mapped[Optional[str]]
    poi_sub_category: Mapped[Optional[str]]
```
The necessary insert statements are then built using the following syntax

```python
insert_stmt_bike_station_info = sqla.insert(BikeStationInfo)
```
The built up insert statement 
```sql
INSERT INTO bike_station_info (station_id, station_name, free_bikes, latitude, longitude) VALUES (:station_id, :station_name, :free_bikes, :latitude, :longitude)
```
is very convenient as it sets up placeholders to enable batch insertion into the database using a list
of dictionaries.

The other insert statements are then defined similarly. Also, the data to be inserted has to be formatted
from the csv files obtained in the previous sections. Finally, table creation and data insertion are
performed using the following syntax
```python
# table creation and data insertion
with engine.begin() as conn:
    Base.metadata.create_all(conn)
    conn.execute(insert_stmt_bike_station_info,bike_station_info_values)
    conn.execute(insert_stmt_poi_info,poi_info_values)
    conn.execute(insert_stmt_bike_station_poi_info,bike_station_poi_info_values)
```
We can then join the datasets, read the joined data from the database, and use the output for 
onward processing in the next steps. 
```python
# we can execute SQL queries on the newly created database and read data from it
sql_query = '''select *
from (bike_station_info as bi
join poi_info as pi on bi.station_id = pi.station_id) as cte
join bike_station_poi_info as bpi on cte.station_id = bpi.station_id and cte.poi_id = bpi.poi_id; '''

stmt = sqla.text(sql_query)
with engine.connect() as conn:
    result = conn.execute(stmt).all()
```
### Exploratory Data Analysis (EDA)
After joining the data about the POIs to the City Bike data on the unique id of the bike stations, we then
proceed to do exploratory data analysis. First, the identifying columns such as the station id, 
poi id, and poi name were removed from the data. While identifying columns are important in other
contexts, they are not needed for the remainder of this project.

First, since the rating for each POI is going to be critical for further assessments, we want
to focus our exploration on those POIs that have enough data such that the location
services were able to generate a rating for them. Consequently, we will drop those POIs
without ratings. We then perform exploratory analysis on how to handle null values in the `poi_price_level` column, which
captures the relative degree of affordability of the POI, with 1 being cheap and 4 very expensive. 
From the analyses, the following conclusions were made:

* The price level data is right-skewed and using the median as the measure of central tendency
will be appropriate
* Although the missing values for the price level are relatively small (~12% of the total),
dropping them will result in loss of a significant amount of data from the Bar and Fashion Retail
categories, since most data is dominated from the Restaurant category. Hence, the decision
to replace missing values with representative values has been made.
* It can be inferred that the price level values are related by group, such as the categories and
the source of data. Hence, representative values will be computed based on this grouping.
* Specifically, the missing price level in each row will be determined by the median 
of the category and from the source it was obtained.

After handling the missing values for the affordability measure, the final column with missing values, `poi_total_reviews`
was then handled by dropping the missing values as they formed an insignificant proportion of the dataset.
Univariate, multivariate and pattern analysis were performed on the dataset. The conclusions from this
phase are summarized below.

* Despite limiting POI search to within 1000m of the bike station, the distance attribute returned shows 
some POIs exceeding the cutoff.
* The POIs are significantly restaurants. It might not be meaningful, for example, to get more granularity into the fashion retail category, due to limited data.
* From the box plots, I can glean that, at a high level, the data might not be representative as many datasets are skewed towards some categories or levels.
* There are some genuine outliers like for poi_total_reviews and number of free_bikes. These are handled in the next section
* There is no discernible linear (or otherwise) relationship between variables.
* No significant variation in poi rating noted between the datasets obtained from the two location services. 
* The average ratings for all primary categories are similar
* At first glance, more expensive POIs seem to have higher average ratings. However, due to the limited data
from that grouping, there is more margin of error in drawing that conclusion.

The important takeaway from this section is that the variables in the dataset do not show any discernible
inherent linear relationship.

### Model Building and Results
#### Linear Regression
The initial observation from the EDA process is confirmed in this section as the linear model is shown
not to be a good fit to model the variable relationships in the dataset. Despite taking the following 
corrective measures to improve the model performance:
* normalizing the numeric values to address conditioning errors and other numerical issues
* removing the significant outliers,
* pruning the independent variables to only those with statistically significant coefficients,

the predictive power of the model remained poor. This strongly suggests that either linear regression is a poor model 
for the underlying relationship or the collected data is not representative enough to fully embed the relationships.
The following tests were further performed to confirm that the assumptions for using a linear regression
model were indeed not met.

* The residuals of the model violate the normality assumptions of the OLS regression, as captured by the
histogram plot and the Shapiro-Wilk's test for normality.
* From the plot of the residuals vs fitted values, the variance of the residuals do not seem randomly 
scattered around the horizontal line at zero, showing a clear violation of homoscedasticity.

#### Multinomial Classification
Additional effort was made to predict the ratings using multinomial regression classification.
This is justifiable as the ratings can be converted to fall in discrete buckets with specified units of increments from the min to max.
However, while the LLR p-value seems to suggest that this classification model is statistically better than the null predictor, 
the model performance as validated from the prediction table show that the model is a poor predictor. 
This strongly suggests that the collected data is not representative enough to embed any discernible relationship between attributes.



## Results
The model results are discussed in the previous section. The comparative quality of API coverage
in my chosen categories will be discussed here. Qualitatively, both Foursquare and Yelp provide comprehensive coverage of POIs in Berlin, my city of choice.
They both provide rich details, including ratings, reviews and classification based on price level. In addition,
both sources include even more details which were not captured due to the rate limits and the
need for uniformity in the selected attributes, such as popularity factor for
Foursquare and whether the POI is open overnight for Yelp.

Quantitatively though, Yelp provides almost 100 percent more coverage as indicated by the total number of POIs
returned in the industries of interest : bars, restuarant, and fashion retail. Yelp returned
a total of 5637 POIs compared to the 2945 for Foursquare.
## Challenges 
The main challenges for this project were the short timeframe and the API rate limits imposed by
the location service providers for the free tier users. As such, only limited datasets were collected.
From the analyses in the previous sections, this was found to have significant impact on the modeling,
as the collected data is not representative enough to embed any discernible relationship between attributes.


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
