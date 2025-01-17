# sqlalchemy-challenge
Module 10 Challenge. Part 1: Analyze and Explore the Climate Data. Part II: Design Your Climate App. 
# Part 1 - Analyze and Explore the Climate Data - refer to hawaii_climate_analysis.ipynb  
# Libary 
%matplotlib inline
from matplotlib import style
style.use('fivethirtyeight')
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
import datetime as dt
import sqlalchemy
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, func, inspect
# Analysis
# create engine to hawaii.sqlite
engine = create_engine("sqlite:///Resources/hawaii.sqlite")
# Create the inspector and connect it to the engine
inspector = inspect(engine)
# Declare a Base using `automap_base()`
Base = automap_base()
# Use the Base class to reflect the database tables
Base.prepare(autoload_with=engine)
# View all of the classes that automap found
Base.classes.keys()
# Collect the names of tables within the database
inspector.get_table_names()
# view columns in 'measurement'

inspector = inspect(engine)
columns = inspector.get_columns('measurement')
for c in columns:
    print(c['name'], c['type'])
# view columns in 'station

inspector = inspect(engine)
columns = inspector.get_columns('station')
for c in columns:
    print(c['name'], c['type'])
# Save references to each table
Station = Base.classes.station
Measurement = Base.classes.measurement
# Create our session (link) from Python to the DB
session = Session(engine)

# Exploratory Precipitation Analysis
# Find the most recent date in the data set.
recent_date =(session.query(Measurement.date)
             .order_by(Measurement.date.desc())
                .first())
recent_date
# Design a query to retrieve the last 12 months of precipitation data and plot the results. 
end_date= dt.date(2017, 8, 23)-dt.timedelta(days=365)
print (end_date)
print (recent_date)
# Starting from the most recent data point in the database
# Calculate the date one year from the last date in data set.
start_date = dt.date(2017,8,23)-dt.timedelta(days=365)
print(start_date)
# Perform a query to retrieve the data and precipitation scores
results= session.query(Measurement.date, Measurement.prcp).filter(Measurement.date >= start_date).order_by(Measurement.date).all()
# Save the query results as a Pandas DataFrame. Explicitly set the column names
df = pd.DataFrame(results, columns=['date', 'precipitation'])
print (df)
# Clean dataframe of precipitation that is NaN
clean_df = df.dropna()
print (clean_df)
# Sort the dataframe by date
clean_df.set_index(clean_df['date'], inplace = True)
clean_df.head()
# Use Pandas Plotting with Matplotlib to plot the data
clean_df.plot(figsize=(10,5))
plt.tight_layout()
plt.xlabel ("Date", size = 10)
plt.xticks(rotation=90, size = 10)
plt.ylabel("Inches", size = 10)
plt.yticks(size = 10)
plt.show()
# Use Pandas to calculate the summary statistics for the precipitation data
clean_df.describe()
# Exploratory Station Analysis
# Design a query to calculate the total number of stations in the dataset
session.query(Station.id).count()
# Design a query to find the most active stations (i.e. which stations have the most rows?)
# List the stations and their counts in descending order.
session.query(Measurement.station,func.count(Measurement.station)).group_by(Measurement.station)\
.order_by(func.count(Measurement.station).desc()).all()
# Using the most active station id from the previous query, calculate the lowest, highest, and average temperature.
session.query(func.min(Measurement.tobs),func.max(Measurement.tobs),func.avg(Measurement.tobs))\
    .filter(Measurement.station=='USC00519281').all()
# Using the most active station id
# Query the last 12 months of temperature observation data for this station
year_temp = session.query(Measurement.tobs).filter(Measurement.station=='USC00519281')\
    .filter(Measurement.date>=start_date).all()
# plot the results as a histogram
USC00519281=pd.DataFrame(year_temp,columns=['tobs'])
USC00519281.plot.hist(bins=12)
plt.tight_layout()
plt.xlabel("Temperature")
plt.ylabel("Frequency")
plt.show()
# Close Session
session.close()

# Part 2 - Design Your Climate App - refer to hawaii_climate_app.py
# Import the dependencies.
from matplotlib import style
style.use('fivethirtyeight')
import matplotlib.pyplot as plt

import numpy as np
import pandas as pd
import datetime as dt

import sqlalchemy
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, func, inspect

from flask import Flask, jsonify
# Database Setup
engine = create_engine("sqlite:///Resources/hawaii.sqlite")
# reflect an existing database into a new model
Base = automap_base()
# reflect the tables
Base.prepare(autoload_with=engine)
# Save references to each table
Station = Base.classes.station
Measurement = Base.classes.measurement
# Create our session (link) from Python to the DB
session = Session(engine)
# Flask Setup
app = Flask(__name__)
# Flask Routes
# 1. Start at homepage. List all the available routes.

@app.route("/")
def welcome():
    """List all available api routes."""
    return (
        f"Available Routes:<br/>"
        f"/api/v1.0/precipitation<br/>"
        f"/api/v1.0/stations<br/>"
        f"/api/v1.0/tobs<br/>"
        f"/api/v1.0/start<br/>"
        f"/api/v1.0/start/end<br/>)"
    )
# 2. Convert the query results from your precipitation analysis. (i.e. retrieve only the last 12 months of data) to a dictionary using date as the key and prcp as the value. Return the JSON representation of your dictionary.

@app.route("/api/v1.0/precipitation")
def precipitation():
    session = Session(engine)
    recent_date = (session.query(Measurement.date).order_by(Measurement.date.desc()).first())
    start_date = dt.date(2017,8,23)-dt.timedelta(days=365)

    results= session.query(Measurement.date, Measurement.prcp).filter(Measurement.date >= start_date).order_by(Measurement.date).all()

    p_dict = dict(results)

    print(f"Results for Precipitation -{p_dict}")
    print(f"Out of Precipitation section.")
    return jsonify (p_dict)
# 3. Return a JSON list of stations from the dataset.
@app.route("/api/v1.0/stations")
def stations():
    session = Session(engine)
    sel = [Station.station, Station.name, Station.latitude, Station.longitude, Station.elevation]
    queryresult = session.query(*sel).all()
    session.close()

    stations = []
    for station,name,lat,lon,el in queryresult:
        station_dict = {}
        station_dict["Station"] = station
        station_dict["Name"] = name
        station_dict["Lat"] = lat
        station_dict["Lon"] = lon
        station_dict["Elevation"] = el
        stations.append(station_dict)

    return jsonify (stations)
# 4. Query the dates and temperature observations of the most-active station for the previous year of data.Return a JSON list of temperature observations for the previous year.

@app.route("/api/v1.0/tobs")
def tobs():
    session = Session(engine)

    queryresult = session.query(Measurement.date,Measurement.tobs).filter(Measurement.station=='USC00519281')\
    .filter(Measurement.date>='2016-08-23').all()

    tob_obs = []
    for date, tobs in queryresult:
        tobs_dict = {}
        tobs_dict["Date"] = date
        tobs_dict["Tobs"] = tobs
        tob_obs.append(tobs_dict)

    return jsonify(tob_obs)
# 5. Return a JSON list of the minimum temperature, the average temperature, and the maximum temperature for a specified start or start-end range.
# For a specified start, calculate TMIN, TAVG, and TMAX for all the dates greater than or equal to the start date.
# For a specified start date and end date, calculate TMIN, TAVG, and TMAX for the dates from the start date to the end date, inclusive.

@app.route("/api/v1.0/<start>")
def specified_start(start):
    session = Session(engine)
    results = session.query(func.min(Measurement.tobs), func.max(Measurement.tobs), func.avg(Measurement.tobs)).\
              filter(Measurement.date >= start).all()
    session.close()

    temp = []
    for min_temp, max_temp, avg_temp in results:
        temp_dict = {}
        temp_dict['Minimum temperature'] = min_temp
        temp_dict['Maximum temperature'] = max_temp
        temp_dict['Average temperature'] = avg_temp
        temp.append(temp_dict)

    return jsonify(temp)

@app.route("/api/v1.0/<start>/<end>")
def get_temps_start_end(start, end):
    session = Session(engine)
    results = session.query(func.min(Measurement.tobs), func.max(Measurement.tobs), func.avg(Measurement.tobs)).\
              filter(Measurement.date >= start).filter(Measurement.date <= end).all()
    session.close()

    temps = []
    for min_temp, max_temp, avg_temp in results:
        temps_dict = {}
        temps_dict['Minimum Temperature'] = min_temp
        temps_dict['Maximum Temperature'] = max_temp
        temps_dict['Average Temperature'] = avg_temp
        temps.append(temps_dict)

    return jsonify(temps)


if __name__ == '__main__':
    app.run(debug=True)
    

