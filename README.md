## How to Build a Live Trading Bot on dYdX wiht Python (Stoch, RSI, MACD)

In the below code we will build a trading bot that uses 3 technical indicators (stochastic oscillators, RSI and MACD) to trade crytocurrencies. We will then deploy our trading bot on the dYdX decentralised perpetuals exchange.

First we import some dependencies that we will need to handle data. We also import the dYdX python module


```python
import pandas
import ta
import numpy as np
import time
import pandas as pd
import datetime as datetime

from dydx3 import Client
from web3 import Web3
```

We then initialise two dYdX client instances: one public instance to get candle data and a private instance where we can submit orders 


```python
#Public instance with dYdX exchange
client = Client(
    host='https://api.dydx.exchange'
)

ETHEREUM_ADDRESS = '#INSERT YOU ETHEREUM ADDRESS HERE'
private_client = Client(
    host='https://api.dydx.exchange',
    api_key_credentials={ 'key': '#INSERT YOUR API KEY HERE', 
                         'secret': '#INSERT YOUR API SECRET HERE', 
                         'passphrase': '#INSERT YOUR API PASSPHRASE HERE'},
    stark_private_key='#INSERT YOUR STARK PRIVATE KEY HERE',
    default_ethereum_address=ETHEREUM_ADDRESS,
)

account_response = private_client.private.get_account()
position_id = account_response.data['account']['positionId']
```

We then build some functions we will need for our trading bot. Firstly, we create an "technicals" function which takes the dYdX price dataframe and creates columns with our TA indicators. The indicators we will be using are described below:
<br>

The Stochastic Oscillator seeks to find oversold and overbought zones by incorporating highs and lows in a normalisation formula:

%K = (Close - Low)/(High - Low)

It is a contrarian indicator that seeks to signal reations of extreme movements. A bullish signal is triggered whenever the indicator reaches or breaks the lower barrier

<br>

The RSI is a well-known indicator which indicates when markets are overbought or oversold. The RSI is the smoothed average of positive differences divided by the smoothed average of negative price differences. This gives us an RS which is then transformed into a measure between 0 and 100.

<br>

Moving average convergence divergence (MACD) is a trend-following momentum indicator that shows the relationship between two moving averages of a security’s price.


```python
def technicals(df):
    df['%K'] = ta.momentum.stoch(df.high, df.low, df.close, window=14, smooth_window=3)
    df['%D'] = df["%K"].rolling(3).mean()
    df['rsi'] = ta.momentum.rsi(df.close, window=14)
    df['macd'] = ta.trend.macd_diff(df.close)
    return df
```

We then create a "stoch_trigger" function which tells us when the Stochastic Oscillator has reached our buy level ( in this case the oscillator drops below 20). Lastly, the "decide" function will create a column in our dataframe to identify scenarios that match our TA buy criteria. Specifically, the bot will buy when (1) the stochastic oscialltor is oversold, (2) the RSI is less than 30 and (3) the MACD is greater than 0. We place these in a class called "Signals"


```python
class Signals:
    
    def __init__(self, df, lags):
        self.df = df
        self.lags = lags
    
    def stoch_trigger(self):
        dfx = pd.DataFrame()
        for i in range(self.lags + 1):
            mask = (self.df['%K'].shift(i) < 20) & (self.df['%D'].shift(i) < 20)
            dfx = dfx.append(mask, ignore_index=True)
        return dfx.sum(axis=0)
    
    def decide(self):
        self.df['trigger'] = np.where(self.stoch_trigger(), 1, 0)
        self.df['Buy'] = np.where((self.df.trigger) & 
                                  (self.df['%K'].between(20,80)) & (self.df['%D'].between(20,80))
                                  & (self.df.rsi < 30) & (self.df.macd >0), 1, 0)        
```

Lastly, we create a bot function which will place buy orders when the above conditions are met. We execute market orders to ensure execution and also create a function which monitors the stop-loss and take-profit of our existing submission. We set our targets 1% above entry price and stops 1% below entry price. The bot updates every minute to check entry prices. 


```python
def strategy(perp_contract, size, open_position=False):
    api_result = client.public.get_candles(
            market=str(perp_contract),
            resolution="1MIN",
        )
    df = pd.DataFrame(api_result.data['candles']).sort_values(by="startedAt")
    df.index = df['startedAt'] 
    df = df[['low', 'high', 'open', 'close']].astype(float)

    
    df = technicals(df)
    inst = Signals(df, 25)
    inst.decide()
    
    print("Current close for ", perp_contract, " : ", df.close.iloc[[-1]][0])
    
    if df.Buy.iloc[-1]:
        order_params = {
            'position_id': position_id,
            'market': perp_contract,
            'side': ORDER_SIDE_BUY,
            'order_type': ORDER_TYPE_MARKET,
            'post_only': FALSE,
            'size': size ,
            'price': str(df.close.iloc[-1][0]) ,
            'limit_fee': '0.0015',
            'expiration_epoch_seconds': time.time() + 120,
        }
        
        try:
            buy_order_dict = private_client.private.create_order(**order_params)
            buy_order_id = position_clear_sell_order_dict.data["order"]['id']
            buy_price = buy_order_dict.data['order']['price']
            print("buy order executed at " + buy_price)
        except Exception as e:
            print(e)
    
    while open_position:
        time.sleep(0.5)
        api_result = client.public.get_candles(
            market=str(perp_contract),
            resolution="1MIN",
        )
        df = pd.DataFrame(api_result.data['candles']).sort_values(by="startedAt")
        df.index = df['startedAt'] 
        df = df[['low', 'high', 'open', 'close']]
        print("current close: ", str(df.close.iloc[[-1]][0]))
        print("current Take Profit: ", str(buyprice * 1.01))
        print("current Take Stop-Loss: ", str(buyprice * 0.99))
        
        if df.close[-1] <= buyprice * 0.99 or df.close[-1] >= buyprice * 1.01:
            order_params = {
                'position_id': position_id,
                'market': perp_contract,
                'side': ORDER_SIDE_SELL,
                'order_type': ORDER_TYPE_MARKET,
                'post_only': FALSE,
                'size': size ,
                'price': str(df.close.iloc[-1][0]) ,
                'limit_fee': '0.0015',
                'expiration_epoch_seconds': time.time() + 120,
            }
            
            try:
                sell_order_dict = private_client.private.create_order(**order_params)
                sell_order_id = position_clear_sell_order_dict.data["order"]['id']
                sell_price = buy_order_dict.data['order']['price']
                print("sell order executed at " + sell_price)
            except Exception as e:
                print(e)
            
            break
```


```python
while True:
    strategy("AVAX-USD", 50)
    time.sleep(60)
```
