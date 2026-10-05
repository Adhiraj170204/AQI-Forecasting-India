# Data

`city_day.csv`: daily city-level air quality for Indian cities, 2015-01-01 to 2020-07-01,
compiled from CPCB (Central Pollution Control Board) monitoring stations. Published on Kaggle
as "Air Quality Data in India (2015 - 2020)".

Columns: City, Date, PM2.5, PM10, NO, NO2, NOx, NH3, CO, SO2, O3, Benzene, Toluene, Xylene,
AQI, AQI_Bucket. Units are ug/m3, except CO (mg/m3).

This study uses the 7 cities with near-complete daily coverage (about 2,009 days each,
14,060 rows in total): Ahmedabad, Bengaluru, Chennai, Delhi, Hyderabad, Lucknow, Mumbai.
