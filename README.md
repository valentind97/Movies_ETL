# Movies-ETL
## Project Overview

This project aims to construct an ETL pipeline that delivers a final comprehensive and merged dataset of movies data. A list of movies and their available details on Wikipedia from 1990 to 2018 was extracted from the sidebar into a JSON, and their corresponding ratings and metadata from the zip file downloaded from [The MovieLens](https://www.themoviedb.org/) website. This composed the extraction method of the ETL pipline. In the transforamtion stage, these data were loaded into pandas DataFrames, cleaned, then merged. The cleaning steps taken can be seen in the code snippet below. For the load method, the cleaned and merged data were uploaded into a Postgres database using SQLAlchemy. This method is also included in the second cleaning function provided below.

## Resources:
- Software: Postgres and PgAdmin 4
  - Python 3.7
    -   
## Results
### The count for the movies data was 6052.

### The count for the ratings data was 24289.

Movies
![Movies_DB](/Resources/movies_query.png)

Ratings
![Ratings_DB](/Resources/ratings_query.png)
