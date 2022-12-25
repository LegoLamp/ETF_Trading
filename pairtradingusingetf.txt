from alpaca.data.live import  StockDataStream
import requests
import json
import asyncio
import pandas as pd
import numpy as np
from alpaca.data.historical import CryptoHistoricalDataClient, StockHistoricalDataClient
from alpaca.data.requests import CryptoBarsRequest, StockBarsRequest
from alpaca.data.timeframe import TimeFrame
from alpaca.trading.client import TradingClient
from alpaca.trading.enums import OrderSide, OrderStatus, TimeInForce
from alpaca.trading.requests import MarketOrderRequest


api_key = "The users alpaca api key"
secret_key = "The users secret alpaca api key"
wss_client = StockDataStream(api_key, secret_key)
trading_client = TradingClient(api_key, secret_key, paper = True)

#calculates geometric mean                    
def mean(Data):
    n = len(Data)
    
    Ans = 1.0
    for x in Data:
        Ans *= x
        
    Ans = Ans ** (1/n)
    return Ans

# calculates population deviation
def sd(Data):
    Xbar = mean(Data)
    n= len(Data)
    Ans = 0.0
    for x in Data:
        Ans += (x - Xbar)**2
        
    Ans /= n
    Ans = Ans**0.5
    
    return Ans

# calculates the moving average formula based on investopedias definition of moving average
def movingaverage(Data):
    n = len(Data)
    Ans = 0.0
    for x in Data:
        Ans += x
    
    Ans /= n
    return Ans

# number of deviations
m = 2

#calculates Bollingerbands upper bands and lower band formula baseed on the definitions in investopedia
def bbandupper(Data):
    return movingaverage(Data) + (m*sd(Data))
def bbandlower(Data):
    return movingaverage(Data) - (m*sd(Data))


# checks whether account has active positions and places trades

def place_trade(up, lo, ma, pr):
    active_position = len(trading_client.get_all_positions()) != 0
    
    if pr > up and not active_position:
        buying_power = float(trading_client.get_account().buying_power)
        TQQQ_notional_size = buying_power // 2
        # variable assinged to other half of buying power 
        SPXS_notional_size = buying_power // 2
        # buy thing with notional size
        market_order_data1 = MarketOrderRequest(symbol = "TQQQ", notional = TQQQ_notional_size, side = OrderSide.BUY, time_in_force = TimeInForce.DAY)
        market_order_data2 = MarketOrderRequest(symbol = "SPXS", notional = SPXS_notional_size, side = OrderSide.BUY, time_in_force = TimeInForce.DAY)
        # buy with corresponding assets
        trading_client.submit_order(order_data=market_order_data1)
        trading_client.submit_order(order_data=market_order_data2)
        # print what you just brought
        print("brought tqqq and spxs")
        
        
    if pr < lo and not active_position:
        buying_power = float(trading_client.get_account().buying_power)
        SQQQ_notional_size = buying_power // 2

        # variable assinged to other half of buying power 
        SPXL_notional_size = buying_power // 2

        # buy other thing with notional size
        market_order_data = MarketOrderRequest(symbol = "SPXL", notional = SQQQ_notional_size, side = OrderSide.BUY, time_in_force = TimeInForce.DAY)
        market_order_data3 = MarketOrderRequest(symbol = "SQQQ", notional = SPXL_notional_size, side = OrderSide.BUY, time_in_force = TimeInForce.DAY)

        # buy corresponding assets
        trading_client.submit_order(order_data=market_order_data)
        trading_client.submit_order(order_data=market_order_data3)

        # print what you just brought
        print("brought sqqq and spxl")
        
        
    if pr <= ma and active_position:
	#gets all the current positions in the account
        positions = trading_client.get_all_positions()

        #makes a dataframe of positions
        g = pd.DataFrame(positions)
        f = g[1]

        #assigns a variable with the symbol in the format of the dataframe above
        h = ('symbol', 'TQQQ')
        j = ('symbol', 'SPXS')

        #searches through the dataframe and checks if the variable is in the dataframe if so then it closes the positions
        for i in f:
            if (i == h) or (i == j ):
                # close all positions
                trading_client.close_all_positions(cancel_orders=True)
                # prints closes x and y  position
                print("closed tqqq and spxs")
        
        
    if pr >= ma and active_position:
	#same as above gets all current positions in the account
        positions = trading_client.get_all_positions()
        #makes a dataframe of positions
        g = pd.DataFrame(positions)
        f = g[1]
        #assigns a variable with the symbol in the format of the dataframe above
        h = ('symbol', 'SQQQ')
        j = ('symbol', 'SPXL')
        #searches trouh the dataframe and checks if the variable is in the dataframe if so then it closes the positions
        for i in f:
            if (i == h) or (i == j ):
                # close all positions
                trading_client.close_all_positions(cancel_orders=True)
                # prints closes x and y  position
                print("closed SQQQ and SPXL")
        
        
    

# gets the latests bar data 15 minute intervals and computes data
def get_more_data(data):
    # gets data through api and makes a data frame
    query_headers = {"APCA-API-KEY-ID": api_key, "APCA-API-SECRET-KEY": secret_key }
    query_params = {"timeframe": "15Min"}
    response = requests.get(url = "https://data.alpaca.markets/v2/stocks/bars?symbols=SPY,QQQ", params = query_params, headers= query_headers)
    d = response.json()

    #creates a dataframe called df for qqq
    df = pd.DataFrame(d["bars"]["QQQ"])

    #creates a dataframe called dff for spy
    dff = pd.DataFrame(d["bars"]["SPY"])

    #makes a new column fore the closing price
    df["qqq_close"] = df["c"]
    dff["spy_close"] = dff["c"]

    #sets a variable for the columns
    q = df["qqq_close"]
    s = dff["spy_close"]
    g = pd.DataFrame()

    # compute price ratios and typical price ratios
    # make a new column in the data frame called typical price ratio and another column called price ratio
    g["price_ratio"] = round(s/q, 4)
    df["qqq_typical_price"] = (df["c"] + df["l"] + df["h"]) / 3
    dff["spy_typical_price"] = (dff["c"] + dff["l"] + dff["h"]) / 3
    g["typical_price_ratio"] = round(dff["spy_typical_price"]/ df["qqq_typical_price"], 4)
      
    #calculates data in dataframe
    # calculate using typical price ratios 
    up = round(bbandupper(g.tail(20)["typical_price_ratio"]), 4)
    lo = round(bbandlower(g.tail(20)["typical_price_ratio"]), 4)
    ma = round(movingaverage(g.tail(20)["typical_price_ratio"]), 4)

    # this will be the price ratio
    pr = movingaverage(g.tail(1)["price_ratio"])

    #prints the calculated data
    print(f"upper: {up}, movavg: {ma}, price ratio: {pr}, low: {lo}")
    place_trade(up, lo, ma, pr)



# async data handler
async def quote_data_handler(data):
    # quote data will arrive here
    # print(data)
    get_more_data(data)
    
#calls the websocket and subscribes to the bar
wss_client.subscribe_bars(quote_data_handler, "SPY")

# runs the client
wss_client.run()


