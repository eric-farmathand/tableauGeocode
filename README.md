# Tableau Geocoding with Mapbox

This solution allows you to use Tableau Prep for performing forward and reverse geocoding, integrated via [TabPy](https://github.com/tableau/TabPy).

This repositoary also includes instructions for launching your own TabPy instance, there is a sample script for deploying this on AWS for testing and use at scale.

## Geocoding

Tableau Desktop and Prep already geocode for a specific set of geographies (known as Geographic Roles).

![roles](assets/roles.png)

There are times users require either more granular data, or additional context about a location-related data point. You can enrich your data through geocoding:

- Forward: Query with text and receive a coordinate pair
- Reverse: Query with a coordinate pair and receive a location.

The Mapbox Geocoding API provides global scale and data coverage. It is also available to all users (no Enterprise Agreement required). For more information, consult the [Mapbox pricing page](https://www.mapbox.com/pricing/#geocode).

## Usage

To simplify setup and deployment, this solution wraps [TabPy](https://github.com/tableau/TabPy) in Docker. This enables cross-platform usage with minimal setup, while also reducing the amount of code a Tableau Desktop user must write.

The architecture is displayed below

![arch](assets/arch.png)

### Prerequisites

1. Install Docker for your platform: [Mac](https://docs.docker.com/docker-for-mac/install/) / [Windows](https://docs.docker.com/docker-for-windows/)
2. Clone this repository

### Build and Connect

Once you have satistifed the prerequisites, run the following commands

```bash
cd src
docker build --tag tabpy
docker run -d -p 80:80 tabpy
```

- docker build --tag tabpy: Build and tag your image as `tabpy` for running
- docker run -d -p 80:80 tabpy: Run the imaged `tabpy` in detached (`d`) mode, mapping the container port 80 to localhost 80 (`p 80:80`).

This will build and run the TabPy instance on your machine. To confirm that the image is running, either run `curl localhost/info` from your terminal or:

```bash
docker ps
```

![ps](assets/dockerps.png)

Once you have confirmed that your image is running, open Tableau Prep 2019.3 and add a [Script node](https://help.tableau.com/current/prep/en-us/prep_scripts_TabPy.htm#add-a-script-to-your-flow).

Your configuration will look as follows:

![config](assets/pyconfig.png)

This will connect Prep to TabPy.

### Geocode with Prep

The Dockerized version of TabPy includes a validated script (`geocode.py`) for running geocodes at scale. This is copied to TabPy on build.

This sample script, `prepForward.py` demonstrates forward geocoding. For more details about the response object read the [Geocoding API documentation](https://docs.mapbox.com/api/search/#geocoding-response-object).

```python
# This allows your script to access to published function
import sys
sys.path.append('/TabPy')
from geocode import geocodeForward

# This is the function you reference in Tableau Prep
def prepGeo(input):
    prepData = geocodeForward(input)
    return prepData

# This is the output schema that is passed to Tableau Prep when geocoding is complete
def get_output_schema():
    return pd.DataFrame(
        {
            "InitialQuery": prep_string(),
            "ReturnAddress": prep_string(),
            "Accuracy": prep_string(),
            "Relevance": prep_decimal(),
            "Longitude": prep_decimal(),
            "Latitude": prep_decimal(),
        }
    )
```

This configuation configuration should look like this:

![prepconfig](assets/prepconfig.png)

Once input, your geocoding process will complete and return new data to your flow.

## Forward Geocoding Requirements

The `prepGeo` function expects a single variable `input`. This variable MUST contain a column with an address string, with a column name of `Address`.

## Reverse Geocoding Requirements

The `prepGeo` function expects a single variable `input`. This variable MUST contain two columns, `Longitude` and `Latitude`. These columns can be of type float or string.

## Customization

The forward and reverse scripts default to a global search and a limit of one. This means that the geocoder will find the *optimal single result* for your query.

If you would like to refine your results, the Mapbox Geocoding API supports the following set of options.

| Type                | Filter            | Description                                                            |
|---------------------|-------------------|------------------------------------------------------------------------|
| Forward and Reverse | Country Filtering | Specify country or list of countries to restrict results               |
| Forward and Reverse | Limit             | Upper bound on number of results (max 10, default 1)                   |
| Reverse             | Type              | Specify the type of result to return (country, postcode, address, etc) |
| Forward             | Bbox              | Include only results that are within a given bounding box              |
| Forward             | Proximity         | Bias results that are closer to a given location                       |

For more details on these options, consult the [Geocoding API documentation](https://docs.mapbox.com/api/search/).

### Customization Example

For reverse geocoding, if you wish to filter to specific countries, you will either need to include a column in your data called "countries" (or similar) or hard code it into your script.

**Column-based approach**

```python
def geocodeReverse(input,token):
    url = "https://api.mapbox.com/geocoding/v5/mapbox.places-permanent/%s.json?access_token=%s&types=address&country=%s&limit=1"
    access_token = token
    input['coords'] = input['longitude'].map(str)+","+input['latitude'].map(str)
    coordinates = list(input["coords"])
    countries = list(input["countries"])
    urls = [url % (quote(list_item), access_token,countries) for list_item in coordinates]
```

**Hard code approach**

```python
def geocodeReverse(input,token):
    url = "https://api.mapbox.com/geocoding/v5/mapbox.places-permanent/%s.json?access_token=%s&types=address&country=%s&limit=1"
    access_token = token
    input['coords'] = input['longitude'].map(str)+","+input['latitude'].map(str)
    coordinates = list(input["coords"])
    countries = 'us'
    urls = [url % (quote(list_item), access_token,countries) for list_item in coordinates]
```

### Rate Limiting

The default rate limit for Geocoding API requsts is 600/minute. The included Python scripts do not have an explicit rate limiter. Instead, the script uses threading to achieve scale.

```python
with ThreadPoolExecutor(max_workers=1) as executor:
```

The parameter `max_workers=1` is how to control scale. Parallelism is roughly linear, but there are diminishing returns as thread values increase. If you exceed your rate limit, your results will fail. If you would like to know more about rate limiting, read the [Mapbox API documentation](https://docs.mapbox.com/api/#rate-limits).

Our estimate is that a single thread will approach `1200` records per minute, so be cautious in geocoding large numbers of points. You can also contact Mapbox if you would discuss having your rate limit raised.
