# -*- coding: utf-8 -*-
"""
Created on Fri Jan 14 23:17:55 2022

@author: User
"""
import requests, ta, time, sys, csv, decimal
import pandas as pd
import numpy as np
# import matplotlib.pyplot as plt
from binance.client import Client
from datetime import datetime 
from pandas import json_normalize
from discord import Webhook, RequestsWebhookAdapter #, File

BinanceKey = {'api_key': 'KEY_HERE',
    'api_secret':'SECRET_KEY_HERE'}


client = Client(BinanceKey['api_key'], BinanceKey['api_secret'])
webhook = Webhook.from_url("WEBHOOK_HERE", adapter=RequestsWebhookAdapter())

url = 'https://rest.coinapi.io/v1/symbols?filter_symbol_id=BINANCE_SPOT_' #Pulling all symbols listed
headers = {'X-CoinAPI-Key' : 'API_KEY_HERE'}

#% API Call (only activate when necessary)
response = requests.get(url, headers=headers)

df_all_symbols = json_normalize(response.json())
df_all_BUSD = df_all_symbols.loc[df_all_symbols['asset_id_quote'] == 'BUSD']

#%%
def changepos(curr, openprice, quantity, buy_time, buy = True): #
    if buy:
        position_frame.loc[position_frame.currency == curr, 'position'] = 1
        position_frame.loc[position_frame.currency == curr, 'openprice'] = openprice
        position_frame.loc[position_frame.currency == curr, 'quantity'] = quantity
        position_frame.loc[position_frame.currency == curr, 'opentime'] = buy_time
    else:
        position_frame.loc[position_frame.currency == curr, 'position'] = 0
        position_frame.loc[position_frame.currency == curr, 'openprice'] = openprice
        position_frame.loc[position_frame.currency == curr, 'quantity'] = quantity
        position_frame.loc[position_frame.currency == curr, 'opentime'] = buy_time
    position_frame.to_csv('position.csv', index=False)
    
def getminutedata(symbol, interval, lookback):
    frame = pd.DataFrame(client.get_historical_klines(symbol,
                                                      interval,
                                                      lookback + ' min ago UTC')) 
    frame = frame.iloc[:,:6]
    frame.columns = ['Time','Open', 'High', 'Low', 'Close', 'Volume']
    frame = frame.set_index('Time')
    frame.index = pd.to_datetime(frame.index, unit = 'ms')
    frame = frame.astype(float)
    return frame

def applytechnicals(df):
    df['KLine'] = ta.momentum.stoch(df.High, df.Low, df.Close, window = 14, smooth_window=3) #Calculate K Line
    df['DLine'] = df['KLine'].rolling(3).mean() #D Line, the SMA over the K Line
    df['rsi']= ta.momentum.rsi(df.Close, window=14)
    df['rsi10']= ta.momentum.rsi(df.Close, window=10)
    df['macd']=ta.trend.macd_diff(df.Close)
    df['EMA50'] = ta.trend.EMAIndicator(df.Close, window = 50, fillna=True).ema_indicator()
    df['EMA200'] = ta.trend.EMAIndicator(df.Close, window = 200, fillna=True).ema_indicator()
    indicator_ichimoku = ta.trend.IchimokuIndicator(df.High, df.Low, window1 = 9, window2 = 26, window3 = 52, visual = False, fillna = True)
    df['ichi_senkouA']= indicator_ichimoku.ichimoku_a()
    df['ichi_senkouB']= indicator_ichimoku.ichimoku_b()
    df['ichi_base']= indicator_ichimoku.ichimoku_base_line() #Kijun
    df['ichi_conversion']= indicator_ichimoku.ichimoku_conversion_line() #Tenkan
    df['chikou_span'] = df.Close.shift(-26)
    # df.dropna(inplace=True)
    
class Signals:
    def __init__(self,df,lags):
        self.df = df
        self.lags = lags

    def getbuytrigger(self):
        dfx = pd.DataFrame()
        for i in range(self.lags +1):
            mask =(self.df.Close >=self.df.EMA50)#(self.df.chikou_span >= self.df.Close)
            dfx = dfx.append(mask, ignore_index = True)
        return dfx.sum(axis=0)
    
    def getselltrigger(self):
        dfx = pd.DataFrame()
        for i in range(self.lags +1):
            mask = (self.df.Close >=self.df.EMA50) #(self.df.chikou_span <= self.df.Close)#
            dfx = dfx.append(mask, ignore_index = True)
        return dfx.sum(axis=0)
    
    def decide(self):
        self.df['buytrigger']=np.where(self.getbuytrigger(),1,0)
        self.df['selltrigger']=np.where(self.getselltrigger(),1,0)
        self.df['Buy'] = np.where((self.df.buytrigger) #Conditions when buy condition met
                                    & (TECHNICALS CONDITION I)
                                    & (TECHNICALS CONDITION II)
                                   , 1, 0) #here is where we input the triggers/conditions for buying
        self.df['Sell'] = np.where((self.df.selltrigger) #Conditions when buy condition met
                                    & (TECHNICALS CONDITION I)
                                    & (TECHNICALS CONDITION II)
                                    ,1,0) # Define conditions for sell here, 1, 0) Then can change it so to have the if statement be based on those conditions
def neg_or_pos(number):
    if number >0:
        return "+"
    else:
        return "-"

def round_down(value, decimals):
    with decimal.localcontext() as ctx:
        d = decimal.Decimal(value)
        ctx.rounding = decimal.ROUND_DOWN
        return round(d, decimals)

def trader(trade_pair, qty):
    # qty = position_frame[position_frame.currency == target_pair].quantity.values[0]
    deploy_df = getminutedata(trade_pair, '1m', '200')
    applytechnicals(deploy_df)
    inst = Signals(deploy_df, 200) #again, be careful of how far back
    inst.decide()
    buyprice = position_frame.openprice[0]
    opentime = f"{datetime.strptime(position_frame.opentime[0], '%Y.%m.%d %H:%M:%S')}"
    if position_frame[position_frame.currency == trade_pair].position.values[0] == 0:
        sys.stdout.write(f'\r{trade_pair} at {datetime.now().strftime("%H:%M:%S")} is BUSD{str(deploy_df.Close.iloc[-1])} | Senkou Δ:{str(round(deploy_df.ichi_senkouA[-1]-deploy_df.ichi_senkouB[-1],3))} | EMA200:{str(round(deploy_df.EMA200.iloc[-1],3))} | RSI: {str(round(deploy_df.rsi.iloc[-1], 1))} | MACD {neg_or_pos(deploy_df.macd.iloc[-1])}')
        sys.stdout.flush()
        if deploy_df.Buy.iloc[-1]:
            # qty = round(trade_capital/unit_price,sizeprecision)
            order = client.create_order(symbol=trade_pair,
                                    side = 'BUY',
                                    type = 'MARKET',
                                    quantity = qty)
            print(order)
            webhook.send(order)
            buyprice = round(float(order['fills'][0]['price']),priceprecision)
            buy_value = round(float(order['cummulativeQuoteQty']),priceprecision)
            with open('order_log.csv', 'a', encoding='utf8') as f:
                writer = csv.writer(f)  # Note: writes lists, not dicts.
                writer.writerow(order.values())
                f.close()
            opentime = f"{datetime.now():%Y.%m.%d %H:%M:%S}"
            changepos(trade_pair, buyprice, qty, opentime, buy=True) # (curr, openprice, quantity, buy_time, buy = True)
            print(f"{round(float(order['executedQty']),2)} {trade_pair} purchased at BUSD{round(float(buyprice),priceprecision)} | Order value: BUSD{round(float(order['cummulativeQuoteQty']),priceprecision)} | Position is OPEN.")
            webhook.send(f"{round(float(order['executedQty']),2)} {trade_pair} purchased at BUSD{round(float(buyprice),priceprecision)} | Order value: BUSD{round(float(order['cummulativeQuoteQty']),priceprecision)} | Position is OPEN.")
            print()
    else:
        sys.stdout.write(f'\rBUSD {round(position_frame.openprice.values[0]*position_frame.quantity.values[0],2)} of {trade_pair} purchased at BUSD{round(float(buyprice),priceprecision)} ({opentime}) | [Current price: BUSD{str(deploy_df.Close.iloc[-1])} | {round((((float(deploy_df.Close.iloc[-1])-buyprice)/buyprice)*100),2)}%] | SenkouΔ {round((deploy_df.ichi_senkouA[-1]-deploy_df.ichi_senkouB[-1]),priceprecision)} | EMA50: {str(round(deploy_df.EMA50.iloc[-1],priceprecision))}')
        sys.stdout.flush()
        if (deploy_df['Sell'][-1] == 1 and deploy_df['Close'][-1] > (buyprice*(1.001))):
            pair_qty_marker = next(item for item in client.get_account()['balances'] if item['asset']==trade_pair.replace('BUSD',''))
            qty = round_down(float(pair_qty_marker['free']),sizeprecision)
            order = client.create_order(symbol=trade_pair,
                                    side = 'SELL',
                                    type = 'MARKET',
                                    quantity = qty)
            sellprice = round(float(order['fills'][0]['price']),priceprecision)
            sell_value = round(float(order['cummulativeQuoteQty']))
            qty = float(order['cummulativeQuoteQty'])
            with open('order_log.csv', 'a', encoding='utf8') as f:
                writer = csv.writer(f)  # Note: writes lists, not dicts.
                writer.writerow(order.values())
                f.close()
            closetime = f"{datetime.now():%Y.%m.%d %H:%M:%S}"
            buy_value = round(position_frame.openprice.values[0]*position_frame.quantity.values[0],2)
            changepos(trade_pair, sellprice, qty, closetime, buy = False)# (curr, openprice, quantity, buy_time, buy = True)
            if sell_value  > buy_value:
                print('Transaction was a win.')
            else:
                print('Transaction was a loss.')
            print(order)
            webhook.send(order)
            print(f'\n{trade_pair} sold.\n {round(float(order["executedQty"]),priceprecision)} units sold, worth BUSD{round(float(order["cummulativeQuoteQty"]),priceprecision)} \nTransaction profit/loss = {round((((sellprice-buyprice)/buyprice)*100),priceprecision)}%')
            webhook.send(f'{trade_pair} sold.| {round(float(order["executedQty"]),priceprecision)} units sold, worth BUSD{round(float(order["cummulativeQuoteQty"]),priceprecision)}. | Transaction profit/loss = {round((((sell_value-buy_value)/buy_value)*100),priceprecision)}%')
            print()

#%
position_frame = pd.read_csv('position.csv') #Tracker for currency positions
order_log_csv = pd.read_csv('order_log.csv')

if position_frame.position.values[0] == 1:
    trade_pair = position_frame.currency.values[0]
    amount = position_frame.quantity.values[0]
    buy_value = amount * position_frame.quantity.values[0]         
    sizeprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['size_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    priceprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['price_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    unit_price = float(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange'] == trade_pair]['price'].values).replace('[','').replace(']',''))
    print(f'\nBot resumed on {trade_pair} at {datetime.now().strftime("%Y.%m.%d - %H:%M:%S")}')
    while True:
        trader(trade_pair, amount)
        time.sleep(0.5)
else:
    trade_pair = position_frame.currency.values[0]
    amount = position_frame.quantity.values[0]
    buy_value = amount * position_frame.quantity.values[0]         
    sizeprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['size_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    priceprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['price_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    unit_price = float(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange'] == trade_pair]['price'].values).replace('[','').replace(']',''))
    start_account_values = client.get_account()  # Getting account data and a snapshot of when you started running the bot
    for i in range(0,len(start_account_values['balances'])):
        if float(start_account_values['balances'][i]['free']) > 0:
            if start_account_values['balances'][i]['asset'].startswith('BUSD'):
                print(f"\nYour spot account currently has ~{start_account_values['balances'][i]['asset']}{round(int(float(start_account_values['balances'][i]['free'])),2)} available for trading.")
    start_BUSD = next(item for item in start_account_values['balances'] if item['asset']=='BUSD')
    
    target_pair = []
    while len(target_pair) < 1:
        ele = input("Input BUSD paired coin here (Symbol Only): ").upper()
        valid_target = False
        if ele in df_all_BUSD.values: #df_all_BUSD.loc[df_all_BUSD['asset_id_base'] == i]
            target_pair.append(ele)
            valid_target = True
        else:
            print('Whoops, try again.')
        
    # Consider pulling error handling from SharpeBot
    trade_pair = ''.join(str(elem)for elem in target_pair)+'BUSD'            
    sizeprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['size_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    priceprecision = abs(decimal.Decimal(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange']==trade_pair]['price_precision'].values).replace('[','').replace(']','')).as_tuple().exponent)
    unit_price = float(str(df_all_BUSD.loc[df_all_BUSD['symbol_id_exchange'] == trade_pair]['price'].values).replace('[','').replace(']',''))
    
    amount = 0
    while amount == 0:
        if int(float(start_BUSD['free'])) <= 0:
            print('Double check your account balance and try again!')
            exit()
        else:
            trade_capital = int(input(f"Key in BUSD amount for trading on {target_pair[0]} (Currently BUSD {unit_price}/unit): "))
            if trade_capital < 10:
                print(f"Binance only allow trades of 10 BUSD or more, please enter a value greater than {trade_capital} BUSD")
                amount = 0
            elif trade_capital < int(float(start_BUSD['free'])):
                print(f"\n~BUSD{trade_capital} out of ~BUSD{int(float(start_BUSD['free']))} available in account set aside for this bot.")
                print("Please keep in mind that won't be the exact amount used because that is just an estimate based on current price.")
                amount = round(trade_capital/unit_price,sizeprecision)
            elif trade_capital >= int(float(start_BUSD['free'])):
                print(f"Please input an amount less than the ~BUSD{int(float(start_BUSD['free']))} available.")
                amount = 0
            else:
                print('Invalid entry, please try again.')
                amount = 0
    
    if trade_pair in position_frame.values:
        print(f'\nBot started at {datetime.now().strftime("%Y.%m.%d - %H:%M:%S")}')
        while True:
            trader(trade_pair, amount)
            time.sleep(0.5)
    else:
        print(f'\nBot started at {datetime.now().strftime("%Y.%m.%d - %H:%M:%S")}')
        position_frame = pd.DataFrame({'currency':[trade_pair], 'position':[0], 'quantity':[amount], 'openprice':[''], 'opentime':[f"{datetime.now():%Y.%m.%d %H:%M:%S}"]})
        position_frame.to_csv('position.csv', index=False)
        while True:
            trader(trade_pair, amount)
            time.sleep(0.5)

#%%
'''

To do:
    Error reporting (when internet disconnects)
    Remove need to call CoinAPI?
    Restarting the bot when error occurs
    Include backtesting
    
'''
