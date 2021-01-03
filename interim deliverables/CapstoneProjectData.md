# Capstone Project Proposal: Battle of the Neighborhoods

## Data sources
In order to solve the problem posed by the project description, I intend to use the following data sources.

### Pools:
My daughter is a swimmer, and needs to train often, so having a nearby swimming pool is of key importance. Thankfully, the **City of Austin** publishes a JSON file with that data:  
https://data.austintexas.gov/Recreation-and-Culture/Pool-Map/jfqh-bqzu  
This data file contains all 45 of the public pools and play areas for Austin.  I will need to filter out the latter ones (pool_type = "Splashpad") and retain the ones where pool_type = "Neighborhood".
Each pool has latitude and longitude coordinates so I can compute a distance from each to any location. Geopy can do this for me using the lat/long values:  
https://geopy.readthedocs.io/en/stable/#module-geopy.distance  
[example response data]
```
{
  "pool_type" : "Neighborhood",
  "weekend" : "Closed",
  "closure_days" : "Closed for Winter",
  "weekday" : "Closed",
  "status" : "Closed",
  "pool_name" : "Patterson",
  "website" : {
    "description" : "Patterson Pool",
    "url" : "http://www.austintexas.gov/department/patterson-pool"
  },
  "phone" : "512-974-9331",
  "location_1" : {
    "latitude" : "30.296592644000043",
    "human_address" : "{\"address\": \"4200 Brookview Dr\", \"city\": \"Austin\", \"state\": \"TX\", \"zip\": \"\"}",
    "needs_recoding" : false,
    "longitude" : "-97.71400165799997"
  }
}
```

### Walkability and Transit:
My wife prefers to avoid driving (especially in Austin traffic!) and relies on public transportation whenever possible. Thankfully, the site **WalkScore.com** offers metrics for this, and allows
up to 5000 queries daily for free:  
https://www.walkscore.com/professional/public-transit-api.php  
Their service takes a latitude, a longitude, and a city, state and returns a 0-100 transit_score for each location. This can be used to find places that have the best transit scores for her ease of travel.  
[example response data]
```
{
    "scored_lat": 47.610135900000003,
    "scored_lon": -122.3420567,
    "transit_score": 100,
    "description": "World-Class",
    "summary": "266 nearby routes: 260 bus, 4 rail, 2 other",
    "ws_link": "http://www.walkscore.com/...",
    "help_link": "https://www.redfin.com/how-walk-score-works"
}
```

### Affordability:
Clearly there may be some neighborhoods that are simply out of my price range despite meeting my other criteria. So to keep my search limited to places I might actually be able to afford, I would like to incorporate rent data.
Fortunately, Austin is one of the areas included in **federal HUD data**, which offers a spreadsheet of "Fair Market Rents" per zip code, so I can use that as a weighted criteria as well.
https://www.huduser.gov/portal/datasets/fmr.html#2021_data  
Their data is organized per zip code and offers the median rents for 2 bedroom, 3 bedroom, and 4 bedroom lodgings, along with a number of extraneous columns I can safely discard.
Since these are medians, I will assume (hope) that a given zip code will have a few offerings that I could afford even if the median itself is over my budget. So if I decide my budget for a 3 bedroom home to be $1500, I will consider areas where the median is up to 20% higher, or $1800 monthly.
As a stretch goal, I may opt to include places just a few hundred higher than this but mark them in a different color so I can consider if I might be excluding a more perfect place if I'm willing to economize in other areas of my life for an ideal home.  
[example CSV data]
```"ZIP Code",HUD Area Code,HUD Metro Fair Market Rent Area Name,"SAFMR 0BR","SAFMR 1BR","SAFMR 2BR","SAFMR 3BR","SAFMR 4BR"
76511,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76527,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76530,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76537,METRO12420M12420,"Austin-Round Rock, TX MSA",$940,"$1,070","$1,270","$1,640","$1,950"
76571,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76573,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76574,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
76577,METRO12420M12420,"Austin-Round Rock, TX MSA",$890,"$1,030","$1,230","$1,590","$1,920"
```

### Bonus points: Food!
All of my family is quite a fan of certain foods, in particular sushi and pizza. Having a home close to such places would be a distinct bonus. So I would like to query the **Foursquare** data for Austin specifically looking for locations that have pizza and/or sushi nearby.
I will use a 2km search radius for each zipcode centroid using the 'explore' endpoint, and passing in category ids for 'Sushi Restuarant' (example CA id = 4bf58dd8d48988d1d2941735) and 'Pizza Place' (example CA id = 4bf58dd8d48988d1ca941735) and get a count of how many are near each zip code centroid.
This will be scaled using feature scaling to produce 0-1 scores for each, and then since both are important in our family, I will compute the harmonic mean for each zip code, which should promote the places with both, and penalize the places with only one.
I may need to renormalize those means depending on the ranges I see after computing the means.

## Combining the data, order of operations.
1. I will seed the dataset with a list of all the zip codes in the Austin metro area. I will either find source data that includes lat/long for the centroids (like https://public.opendatasoft.com/explore/dataset/us-zip-code-latitude-and-longitude/table/?refine.state=TX) or I will use Geopy Nominatim to query that data for each.
2. To reduce the number of API calls needed in the dynamic queries, I will first filter the zip codes based on the affordability data. This should cull any locations that are cost-prohibitive and not worth exploring further. I will generate a scoring metric for the keepers that prioritizes if they are particularly cheap and would save me money while meeting all my other needs.
3. Next I will apply the swimming pool data, since it a single downloaded JSON file, compute the distances, and I can see if any more of the zip codes can be eliminated for not having any pools nearby.
4. Then I will perform the Transit score queries for each of what remains, and normalize them as well (they are a 100 point scale, but I might find that none come close to that), appending them to each zip code. I may be able to exclude some locations that simply have too little transit coverage to be viable.
5. Following this, I will query the Foursquare data for the remaining zip codes, making two passes for Pizza and Sushi, and compute the harmonic means for those.
6. Finally, I am able to produce a composite score that takes all the preceding metrics using (most likely) an average for them. I will produce a Folium heatmap so I can visualize which locations are most optimal, and generate a top 5 sorted list that I can furnish to a realtor to help with the search for a home in that zip code.
