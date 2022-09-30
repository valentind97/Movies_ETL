# Movies-ETL
## Project Overview

This project aims to construct an ETL pipeline that delivers a final comprehensive and merged dataset of movies data. A list of movies and their available details on Wikipedia from 1990 to 2018 was extracted from the sidebar into a JSON, and their corresponding ratings and metadata from the zip file downloaded from [The MovieLens](https://www.themoviedb.org/) website. This composed the extraction method of the ETL pipline. In the transforamtion stage, these data were loaded into pandas DataFrames, cleaned, then merged. The cleaning steps taken can be seen in the code snippet below. For the load method, the cleaned and merged data were uploaded into a Postgres database using SQLAlchemy. This method is also included in the second cleaning function provided below.

## Resources:
- Software: Postgres and PgAdmin 4
  - Python 3.7
    -   
```python
import json
import pandas as pd
import numpy as np
import re
from sqlalchemy import create_engine
import psycopg2
import time
```

- Data:
  - /Resources/movies_metadata.csv
  - /Resources/wikipedia-movies.json
  - [The MovieLens](https://www.themoviedb.org/) zip file : ratings.csv

## Tranformation and Load Code Snippets:

### Python Function to Read in Data:
- Used in subsequent function
```Python
# Add the clean movie function that takes in the argument, "movie".
def clean_movie(movie):
    movie = dict(movie) #create a non-destructive copy
    alt_titles = {}
    # combine alternate titles into one list
    for key in ['Also known as','Arabic','Cantonese','Chinese','French',
                'Hangul','Hebrew','Hepburn','Japanese','Literally',
                'Mandarin','McCune-Reischauer','Original title','Polish',
                'Revised Romanization','Romanized','Russian',
                'Simplified','Traditional','Yiddish']:
        if key in movie:
            alt_titles[key] = movie[key]
            movie.pop(key)
    if len(alt_titles) > 0:
        movie['alt_titles'] = alt_titles

    # merge column names
    def change_column_name(old_name, new_name):
        if old_name in movie:
            movie[new_name] = movie.pop(old_name)
    change_column_name('Adaptation by', 'Writer(s)')
    change_column_name('Country of origin', 'Country')
    change_column_name('Directed by', 'Director')
    change_column_name('Distributed by', 'Distributor')
    change_column_name('Edited by', 'Editor(s)')
    change_column_name('Length', 'Running time')
    change_column_name('Original release', 'Release date')
    change_column_name('Music by', 'Composer(s)')
    change_column_name('Produced by', 'Producer(s)')
    change_column_name('Producer', 'Producer(s)')
    change_column_name('Productioncompanies ', 'Production company(s)')
    change_column_name('Productioncompany ', 'Production company(s)')
    change_column_name('Released', 'Release Date')
    change_column_name('Release Date', 'Release date')
    change_column_name('Screen story by', 'Writer(s)')
    change_column_name('Screenplay by', 'Writer(s)')
    change_column_name('Story by', 'Writer(s)')
    change_column_name('Theme music composer', 'Composer(s)')
    change_column_name('Written by', 'Writer(s)')

    return movie
```
### Python Function to Clean/Parse data, Merge DataFrames, and Push to the SQL database

```Python
# 1 Add the function that takes in three arguments;
# Wikipedia data, Kaggle metadata, and MovieLens rating data (from Kaggle)

def extract_transform_load():
    # 1. Read in the kaggle metadata and MovieLens ratings CSV files as Pandas DataFrames.
    kaggle_metadata = pd.read_csv(kaggle_file, low_memory=False)
    ratings = pd.read_csv(ratings_file)
    
    # 2. Open the read the Wikipedia data JSON file.
    with open(wiki_file, mode='r') as file:
        wiki_movies_raw = json.load(file)
    
    
    # 3. Write a list comprehension to filter out TV shows.
    wiki_movies = [movie for movie in wiki_movies_raw
               if ('Director' in movie or 'Directed by' in movie)
                   and 'imdb_link' in movie
                   and 'No. of episodes' not in movie]

    # 4. Write a list comprehension to iterate through the cleaned wiki movies list
    # and call the clean_movie function on each movie.
    clean_movies = [clean_movie(movie) for movie in wiki_movies]


    # 5. Read in the cleaned movies list from Step 4 as a DataFrame.
    wiki_movies_df = pd.DataFrame(clean_movies)

    # 6. Write a try-except block to catch errors while extracting the IMDb ID using a regular expression string and
    #  dropping any imdb_id duplicates. If there is an error, capture and print the exception.
    try:
        wiki_movies_df['imdb_id'] = wiki_movies_df['imdb_link'].str.extract(r'(tt\d{7})')
        wiki_movies_df.drop_duplicates(subset='imdb_id', inplace=True)
    except: 
        print('Encountered Error in IMDB Extraction phase')
        
    #  7. Write a list comprehension to keep the columns that don't have null values from the wiki_movies_df DataFrame.
    wiki_columns_to_keep = [column for column in wiki_movies_df.columns if wiki_movies_df[column].isnull().sum() < len(wiki_movies_df) * 0.9]
    wiki_movies_df = wiki_movies_df[wiki_columns_to_keep]

    # 8. Create a variable that will hold the non-null values from the “Box office” column.
    box_office = wiki_movies_df['Box office'].dropna()
    
    # 9. Convert the box office data created in Step 8 to string values using the lambda and join functions.
    box_office = box_office.apply(lambda x: ' '.join(x) if type(x) == list else x)
    box_office = box_office.str.replace(r'\$.*[-—–](?![a-z])', '$', regex=True)

    # 10. Write a regular expression to match the six elements of "form_one" of the box office data.
    form_one = r'\$\s*\d+\.?\d*\s*[mb]illi?on'

    # 11. Write a regular expression to match the three elements of "form_two" of the box office data.
    form_two = r'\$\s*\d{1,3}(?:[,\.]\d{3})+(?!\s[mb]illion)'

    # 12. Add the parse_dollars function.
    def parse_dollars(s):
    # if s is not a string, return NaN
        if type(s) != str:
            return np.nan

        # if input is of the form $###.# million
        if re.match(r'\$\s*\d+\.?\d*\s*milli?on', s, flags=re.IGNORECASE):

            # remove dollar sign and " million"
            s = re.sub('\$|\s|[a-zA-Z]','', s)

            # convert to float and multiply by a million
            value = float(s) * 10**6

            # return value
            return value

        # if input is of the form $###.# billion
        elif re.match(r'\$\s*\d+\.?\d*\s*billi?on', s, flags=re.IGNORECASE):

            # remove dollar sign and " billion"
            s = re.sub('\$|\s|[a-zA-Z]','', s)

            # convert to float and multiply by a billion
            value = float(s) * 10**9

            # return value
            return value

        # if input is of the form $###,###,###
        elif re.match(r'\$\s*\d{1,3}(?:[,\.]\d{3})+(?!\s[mb]illion)', s, flags=re.IGNORECASE):

            # remove dollar sign and commas
            s = re.sub('\$|,','', s)

            # convert to float
            value = float(s)

            # return value
            return value

        # otherwise, return NaN
        else:
            return np.nan
    
        
    #Clean the box office column in the wiki_movies_df DataFrame.
    wiki_movies_df['box_office'] = box_office.str.extract(
        f'({form_one}|{form_two})', 
        flags=re.IGNORECASE
    )[0].apply(parse_dollars)
    
    #We no longer need the Box Office column, so we'll just drop it:
    wiki_movies_df.drop('Box office', axis=1, inplace=True)

    #Clean the budget column in the wiki_movies_df DataFrame.
    budget = wiki_movies_df['Budget'].dropna()
    
    #Convert any lists to strings:
    budget = budget.map(lambda x: ' '.join(x) if type(x) == list else x)
    
    #remove any values between a dollar sign and a hyphen (for budgets given in ranges)
    budget = budget.str.replace(r'\$.*[-—–](?![a-z])', '$', regex=True)
    
    #Remove the citation references with the following:
    budget = budget.str.replace(r'\[\d+\]\s*', '')
    
    #extract and replace
    wiki_movies_df['budget'] = budget.str.extract(
        f'({form_one}|{form_two})', 
        flags=re.IGNORECASE
    )[0].apply(parse_dollars)
    
    #Drop the old column
    wiki_movies_df.drop('Budget', axis=1, inplace=True)

    #Clean the release date column in the wiki_movies_df DataFrame.
    
    #make a variable that holds the non-null values of Release date in the DataFrame, converting lists to strings
    release_date = wiki_movies_df['Release date'].dropna().apply(lambda x: ' '.join(x) if type(x) == list else x)
    
    # Define regex forms for the date format
    date_form_one = r'(?:January|February|March|April|May|June|July|August|September|October|November|December)\s[123]?\d,\s\d{4}'
    date_form_two = r'\d{4}.[01]\d.[0123]\d'
    date_form_three = r'(?:January|February|March|April|May|June|July|August|September|October|November|December)\s\d{4}'
    date_form_four = r'\d{4}'

    #Extract the dates
    wiki_movies_df['release_date'] = pd.to_datetime(
        release_date.str.extract(
            f'({date_form_one}|{date_form_two}|{date_form_three}|{date_form_four})'
        )[0], 
        infer_datetime_format=True
    )
    
    #Clean the running time column in the wiki_movies_df DataFrame.
    running_time = wiki_movies_df['Running time'].dropna().apply(lambda x: ' '.join(x) if type(x) == list else x)
    running_time_extract = running_time.str.extract(r'(\d+)\s*ho?u?r?s?\s*(\d*)|(\d+)\s*m')
    running_time_extract = running_time_extract.apply(lambda col: pd.to_numeric(col, errors='coerce')).fillna(0)
    wiki_movies_df['running_time'] = running_time_extract.apply(lambda row: row[0]*60 + row[1] if row[2] == 0 else row[2], axis=1)
    wiki_movies_df.drop('Running time', axis=1, inplace=True)   
     
    # 2. Clean the Kaggle metadata.
    kaggle_metadata = kaggle_metadata[kaggle_metadata['adult'] == 'False'].drop('adult',axis='columns')
    kaggle_metadata['video'] = kaggle_metadata['video'] == 'True'
    kaggle_metadata['budget'] = kaggle_metadata['budget'].astype(int)
    kaggle_metadata['id'] = pd.to_numeric(kaggle_metadata['id'], errors='raise')
    kaggle_metadata['popularity'] = pd.to_numeric(kaggle_metadata['popularity'], errors='raise')
    kaggle_metadata['release_date'] = pd.to_datetime(kaggle_metadata['release_date'])
    ratings['timestamp'] = pd.to_datetime(ratings['timestamp'], unit='s')

    # 3. Merged the two DataFrames into the movies DataFrame.
    movies_df = pd.merge(wiki_movies_df, kaggle_metadata, on='imdb_id', suffixes=['_wiki','_kaggle'])


    # 4. Drop unnecessary columns from the merged DataFrame.
    # Drop wikipedia from the titles, release date, language and production companies
    movies_df.drop(columns=['title_wiki','release_date_wiki','Language','Production company(s)'], inplace=True)

    # 5. Add in the function to fill in the missing Kaggle data.
    def fill_missing_kaggle_data(df, kaggle_column, wiki_column):
        df[kaggle_column] = df.apply(
            lambda row: row[wiki_column] if row[kaggle_column] == 0 else row[kaggle_column]
            , axis=1)
        df.drop(columns=wiki_column, inplace=True)

    # 6. Call the function in Step 5 with the DataFrame and columns as the arguments.
    fill_missing_kaggle_data(movies_df, 'runtime', 'running_time')
    fill_missing_kaggle_data(movies_df, 'budget_kaggle', 'budget_wiki')
    fill_missing_kaggle_data(movies_df, 'revenue', 'box_office')

    # 7. Filter the movies DataFrame for specific columns.
    movies_df = movies_df.loc[:, ['imdb_id',
                                  'id',
                                  'title_kaggle',
                                  'original_title',
                                  'tagline',
                                  'belongs_to_collection',
                                  'url','imdb_link',
                                  'runtime',
                                  'budget_kaggle',
                                  'revenue',
                                  'release_date_kaggle',
                                  'popularity',
                                  'vote_average',
                                  'vote_count',
                                  'genres',
                                  'original_language',
                                  'overview',
                                  'spoken_languages',
                                  'Country',
                                  'production_companies',
                                  'production_countries',
                                  'Distributor',
                                  'Producer(s)',
                                  'Director',
                                  'Starring',
                                  'Cinematography',
                                  'Editor(s)',
                                  'Writer(s)',
                                  'Composer(s)',
                                  'Based on'
                          ]]

    # 8. Rename the columns in the movies DataFrame.
    movies_df.rename({'id':'kaggle_id',
                      'title_kaggle':'title',
                      'url':'wikipedia_url',
                      'budget_kaggle':'budget',
                      'release_date_kaggle':'release_date',
                      'Country':'country',
                      'Distributor':'distributor',
                      'Producer(s)':'producers',
                      'Director':'director',
                      'Starring':'starring',
                      'Cinematography':'cinematography',
                      'Editor(s)':'editors',
                      'Writer(s)':'writers',
                      'Composer(s)':'composers',
                      'Based on':'based_on'
                     }, axis='columns', inplace=True)

    # 9. Transform and merge the ratings DataFrame.
    rating_counts = ratings.groupby(['movieId','rating'], as_index=False).count()
    rating_counts = ratings.groupby(['movieId','rating'], as_index=False).count() \
                .rename({'userId':'count'}, axis=1)
    # pivot this data so that movieId is the index, 
    #the columns will be all the rating values, 
    #and the rows will be the counts for each rating value.
    rating_counts = ratings.groupby(['movieId','rating'], as_index=False).count() \
                    .rename({'userId':'count'}, axis=1) \
                    .pivot(index='movieId',columns='rating', values='count')
    rating_counts.columns = ['rating_' + str(col) for col in rating_counts.columns]
    #Left merge with movies_df
    movies_with_ratings_df = pd.merge(movies_df, rating_counts, left_on='kaggle_id', right_index=True, how='left')
    #Fill any rows without ratings
    movies_with_ratings_df[rating_counts.columns] = movies_with_ratings_df[rating_counts.columns].fillna(0)
    
    #Load the movies_df and ratings to the Postgres database
    db_string = f"postgresql://postgres:{db_password}@127.0.0.1:5432/movie_data"
    engine = create_engine(db_string)
    movies_df.to_sql(name='movies', con=engine, if_exists='replace')

    rows_imported = 0
    # get the start_time from time.time()
    start_time = time.time()
    for data in pd.read_csv(ratings_file, chunksize=1000000):
        print(f'importing rows {rows_imported} to {rows_imported + len(data)}...', end='')
        data.to_sql(name='ratings', con=engine, if_exists='append')
        rows_imported += len(data)

        # add elapsed time to final print out
        print(f'Done. {time.time()/60 - start_time/60} total minutes elapsed')
    
    return wiki_movies_df, movies_with_ratings_df, movies_df
```
## Postgres Database
### *ratings_db must be dropped each time it is uploaded because it is built by appending 1,000,000 chucksizes at a time, so the `if_exists='replace'` command cannot be used to replaced this table*

![Movies_DB](/Resources/movies_query.png)
![Ratings_DB](/Resources/ratings_query.png)
