## Part 1: Connecting to CityBikes API

For this part, we will work with an API that you have not seen before: [CityBikes](https://citybik.es/) 

Citybikes is an API that provides bike sharing data for apps, research and projects.
CityBikes supports more than 400 cities and the Citybikes API is an interesting dataset for building bike-sharing transportation projects.

Your tasks are as follows:
1. Explore the structure of the API, query the API and understand the data returned. 
2. Choose a city covered by the CityBikes API and retrieve all available bike stations in that city. 
3. For each bike station, use the API to call the latitude, longitude and number of bikes. 
4. Parse the JSON object into a Pandas dataframe. 

Complete the **city_bikes.ipynb** notebook to demonstrate how you executed the tasks above. 

## Part 2: Connecting to Foursquare and Yelp APIs

Your tasks are as follows:
1. Connect to the  [Foursquare](https://developer.foursquare.com/places) API
2. Connect to the [Yelp](https://www.yelp.com/developers/documentation/v3/get_started) API. This API offers similar services as Foursquare.
3. For each of the bike stations in Part 1, query both APIs to retrieve information for the following in that location:
 - Restaurants or bars
 - Various POIs (points of interest) of your choice
4. Create a DataFrame for the Yelp results and Foursquare results. 
5. Compare the quality of the Yelp and Foursquare API. For your location, which API gives you the most complete information/better coverage? *NOTE:* Your definition of 'coverage' is up to you. It could be simple 'number of POIs in the area', but it could also be something more specific like 'number of reviews per POI', or 'number of different attributes of each POI'.


Complete the **yelp_foursquare_EDA.ipynb** notebook to demonstrate how you executed the tasks above.

## Part 3: Joining Data

1. Join the data from Part 1 with the data from Part 2 to create a new dataframe. 
2. Use data visualization to explore the data. 
3. Create your own SQLite database and store the data you've collected on the POIs. **Put some thought into the structure of your database.** We've used and created sqlite3 databases before in the activity [**SQL in Python**](https://data.compass.lighthouselabs.ca/b9e08cd5-68c6-490c-a32b-a66f01bf53e1).
Validate your data.

Complete the **joining_data.ipynb** notebook to demonstrate how you executed the tasks above.


## Part 4: Building a Model

1. Build a regression model using Python’s `statsmodels` module that demonstrates a relationship between the number of bikes in a particular location and the characteristics of the POIs in that location.  
2. Interpret results. Expand on the model output, and derive insights from your model.
3. Stretch: can you think of a way to turn the above regression problem into a classification one? Without coding, can you sketch out how you would cast the problem specifically, and lay out your approaches?

Complete the **model_building.ipynb** notebook to demonstrate how you executed the tasks above.