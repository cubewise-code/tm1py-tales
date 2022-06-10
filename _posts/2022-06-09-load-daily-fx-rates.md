---
layout: post
title:  "Load Daily Fx Rates Using TM1Py"
date:   2022-06-09
categories: introduction
---

Load Daily Fx Rates Using TM1Py
=====

Manually updating foreign exchange rates can be bore; luckily, TM1Py allows us to load daily Fx rates directly into our model. By using TM1Py, we can reduce the person hrs required to input those rates and have greater confidence that the data in our model is the most accurate and up-to-date information that is available. 
Cubewise has 4 Fx API examples on GitHub, two using European Central Bank and two using FRED. In this example, we've elected to use Alphavantage because it is also free, and we've found that the data is updated daily.

THE DATA
Data is retrieved from Alphavantage by passing a "currency from" and a "currency to" in the URL. 
 
``` python 
def get_data(currency_from: str):
    # Create url using the currency from argument to retrieve conversion rates
    main_url = base_url + "&from_symbol=" + currency_from + "&to_symbol=" + to_currency + "&apikey=" + api_key + "&outputsize=full" + "&datatype=json"
    # return response object
    req_ob = requests.get(main_url)
```
_____

Alphavantage returns data in a nested dictionary that we extract and reformat using the code below.

``` python 
# result contains a list of nested dictionaries
result = req_ob.json()['Time Series FX (Daily)']
# Loop through JSON
dates = list()
close = list()
open = list()
for date in result:
    dates.append(date)
    close.append(result[date]['4. close'])
    open.append(result[date]['1. open'])
return dates, close
```
_____

Then in the "get_rates" function, we convert the data into a Dataframe using Pandas. Once the data is in a Dataframe, it becomes easy to filter the results to the values in the current month. After we filter, we use the ".mean" function in pandas to get the average rate for the current month. The most recent value (generally the current daily rate ) will be loaded as the spot rate.

``` python 
def get_rates(date_dict: dict, close_dict: dict, currency_from: str):
    df = pd.DataFrame.from_dict({'Date': date_dict, 'Close': close_dict})
    # Get the first value in the Dataframe (this will be the most recent date they have)
    spot_rate = df['Close'].iloc[0]
    # Filter using the current month from TM1
    df = df[df['Date'].apply(
        lambda x: x.startswith(year_month))]
    # some currencies don't have data for the 1st of the month on the day; this will handle for that
    try:
        avg_rate = df["Close"].mean()
    except Exception as e:
        print(e)
        avg_rate = spot_rate
    print('Spot Rate: ',spot_rate)
    print('Average Rate: ',avg_rate)
    return spot_rate, avg_rate
```
_____

Finally, we create another Dataframe that matches the dimension order of our cube and load the data into TM1.

``` python 
def load_rates(spot_rate: float, avg_rate: float, currency_from: str):
# Build Dataframe to load spot and avg rate
    datafx = {'Version': ['Actual', 'Actual'],
       'Currency': [currency_from, currency_from],
       'Time': [time, time],
       'Rate Type': ['Spot Rate', 'Average Rate'],
       'Value': [spot_rate, avg_rate]}
    df_fx = pd.DataFrame(datafx)
    tm1.cells.write_dataframe(cube_name='Exchange Rate', data=df_fx, use_ti=True, skip_non_updateable=True)
  ```
_____

ON-DEMAND EXECUTION
Using the "Execute Command" function in TI, we can run the python script on-demand or via a chore. Below is an example of how we can run it from a TI using a button in PAW.


![PAW Gif](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2022-06-09-load-fx-rates.gif?raw=true)  
