# IB-Net-Implied-IV-

```python
from ibapi.client import EClient
from ibapi.wrapper import EWrapper
from ibapi.contract import Contract
import threading
import time
import pandas as pd

# tickers = ["INTC","AMZN","MSFT"]
tickers = ["AMZN"]

class TradingApp(EWrapper, EClient):
    def __init__(self):
        EClient.__init__(self,self)
        self.data = {}
        self.df_data = {}
        self.underlyingPrice = {}
        self.atmCallOptions = {}
        self.atmPutOptions = {}
        self.optionPrice = {}
        self.greeks = {}


        self.CALL_DELTA=0.25
        self.PUT_DELTA=0.25

        self.call_local_symbols=['AMZN  230505C00094000',
                                'AMZN  230505C00095000',
                                'AMZN  230505C00096000',
                                'AMZN  230505C00097000',
                                'AMZN  230505C00098000',
                                'AMZN  230505C00099000',
                                'AMZN  230505C00100000',
                                'AMZN  230505C00101000',
                                'AMZN  230505C00102000',
                                'AMZN  230505C00103000',
                                'AMZN  230505C00104000',
                                'AMZN  230505C00105000',
                                'AMZN  230505C00106000',
                                'AMZN  230505C00107000',
                                'AMZN  230505C00108000',
                                'AMZN  230505C00109000',
                                'AMZN  230505C00110000',
                                'AMZN  230505C00111000',
                                'AMZN  230505C00112000',
                                'AMZN  230505C00113000']
        self.call_local_symbols_delta=[0.97,1,0.97,0.95,0.97,0.93,0.86,0.80,0.70,0.59,0.46,0.33,0.22,0.14,0.08,0.05,0.03,0.02,0.01,0.01,0.01]

        self.put_local_symbols=['AMZN  230505P00094000',
                                'AMZN  230505P00095000',
                                'AMZN  230505P00096000',
                                'AMZN  230505P00097000',
                                'AMZN  230505P00098000',
                                'AMZN  230505P00099000',
                                'AMZN  230505P00100000',
                                'AMZN  230505P00101000',
                                'AMZN  230505P00102000',
                                'AMZN  230505P00103000',
                                'AMZN  230505P00104000',
                                'AMZN  230505P00105000',
                                'AMZN  230505P00106000',
                                'AMZN  230505P00107000',
                                'AMZN  230505P00108000',
                                'AMZN  230505P00109000',
                                'AMZN  230505P00110000',
                                'AMZN  230505P00111000',
                                'AMZN  230505P00112000',
                                'AMZN  230505P00113000',
                                'AMZN  230505P00114000']
        self.put_local_symbols_delta=[-0.01,-0.016,-0.02,-0.035,-0.04,-0.07,-0.12,-0.19,-0.28,-0.40,-0.53,-0.669,-0.77,-0.86,-0.90,-0.93,-0.948,-0.943,-0.956,-0.968,-0.89]



    def tickPrice(self, reqId, tickType, price, attrib):
        super().tickPrice(reqId, tickType, price, attrib)
        #print(price)
        if tickType == 4:
            if reqId < 100:
                self.underlyingPrice[tickers[reqId]] = price


        
    def error(self, reqId, errorCode: int, errorString: str, advancedOrderRejectJson = ""):
             super().error(reqId, errorCode, errorString, advancedOrderRejectJson)
             if advancedOrderRejectJson:
                 print("Error. Id:", reqId, "Code:", errorCode, "Msg:", errorString, "AdvancedOrderRejectJson:", advancedOrderRejectJson)
             else:
                 print("Error. Id:", reqId, "Code:", errorCode, "Msg:", errorString)        
    def contractDetails(self, reqId, contractDetails):
        if reqId not in self.data:
            self.data[reqId] = [{"expiry":contractDetails.contract.lastTradeDateOrContractMonth,
                                 "strike":contractDetails.contract.strike,
                                 "call/put":contractDetails.contract.right,
                                 "symbol":contractDetails.contract.localSymbol}]
        else:
            self.data[reqId].append({"expiry":contractDetails.contract.lastTradeDateOrContractMonth,
                                     "strike":contractDetails.contract.strike,
                                     "call/put":contractDetails.contract.right,
                                     "symbol":contractDetails.contract.localSymbol})
    
    def contractDetailsEnd(self, reqId):
        super().contractDetailsEnd(reqId)
        print("ContractDetailsEnd. ReqId:", reqId)
        self.df_data[tickers[reqId]] = pd.DataFrame(self.data[reqId])
        
        # fileName=tickers[reqId]+'.csv'
        # csvFile=self.df_data[tickers[reqId]].to_csv(fileName)
        contract_event.set()

    
    def tickOptionComputation(self, reqId, tickType, tickAttrib,impliedVol, delta, optPrice, pvDividend,gamma, vega, theta, undPrice):
            
            super().tickOptionComputation(reqId, tickType, tickAttrib, impliedVol, delta,optPrice, pvDividend, gamma, vega, theta, undPrice)
            print('ticktype',tickType)
            if reqId >=100 and reqId <200:
                if tickType == 11:
                   print('Greek Id :',reqId,'Delta : ',delta)
                   self.greeks[reqId] = {"Delta":delta, "IV":impliedVol,"OptPrice":optPrice}

             #self.call_local_symbols_delta.append(delta)


def specificOpt(local_symbol,sec_type="OPT",currency="USD",exchange="BOX"):
    contract = Contract()
    contract.symbol = local_symbol.split()[0]
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    contract.right = local_symbol.split()[1][6]
    contract.lastTradeDateOrContractMonth ="20"+ local_symbol.split()[1][:6]
    contract.strike = float(local_symbol.split()[1][7:])/1000
    return contract    

#STEP - 2/1    
def usTechOpt(symbol,sec_type="OPT",currency="USD",exchange="BOX"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    contract.lastTradeDateOrContractMonth = "20230505"
    return contract

def usTechStk(symbol,sec_type="STK",currency="USD",exchange="ISLAND"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    return contract 

def streamSnapshotData(tickers):
    """stream tick leve data"""
    for ticker in tickers:
        app.reqMktData(reqId=tickers.index(ticker), 
                       contract=usTechStk(ticker),
                       genericTickList="",
                       snapshot=False,
                       regulatorySnapshot=False,
                       mktDataOptions=[])
        time.sleep(1) 

        
def atm_call_option(contract_df,stock_price):
    # df=contract_df
    # df_call=df[df['call/put']=="C"]
    # df_call=df_call.sort_values(by='symbol')
    # symbolList=list(df_call['symbol'])
    
    # #STEP -1 ATM
    # atm=None
    # atm_index=None
    # LTP=stock_price
    # for local_symbol in symbolList:
    #     strike=float(local_symbol.split()[1][7:])/1000
    #     if strike>LTP:
    #         break
    #     atm=strike
    #     atm_index=symbolList.index(local_symbol)

    # print('atm:',atm)
    # #STEP -2 CALL SYMBOLS[ATM-10:ATM+10]
    # call_local_symbols=symbolList[atm_index-10:atm_index+11]
    # app.call_local_symbols=call_local_symbols

    # #DELTA APPEND FOR SYMBOLS
    # for opt in app.call_local_symbols:
    #     app.reqMktData(reqId=500+app.call_local_symbols.index(opt), 
    #                 contract=specificOpt(opt),
    #                 genericTickList="",
    #                 snapshot=False,
    #                 regulatorySnapshot=False,
    #                 mktDataOptions=[])
    #     time.sleep(5)
        

    #STEP 4 DELTA SYMBOL
    minn=float('inf')
    minnIndex=None
    for delta in app.call_local_symbols_delta:
        if abs(delta-app.CALL_DELTA) <minn:
            minn=abs(delta-app.CALL_DELTA)
            minnIndex=app.call_local_symbols_delta.index(delta)

    CALL_DELTA_SYMBOL=app.call_local_symbols[minnIndex]
    return CALL_DELTA_SYMBOL




        


   

    #return atm_option

def atm_put_option(contract_df,stock_price):
    # df=contract_df
    # df_put=df[df['call/put']=="P"]
    # df_put=df_put.sort_values(by='symbol')
    # symbolList=list(df_put['symbol'])
    
    # #STEP -1 ATM
    # atm=None
    # atm_index=None
    # LTP=stock_price
    # for local_symbol in symbolList:
    #     strike=float(local_symbol.split()[1][7:])/1000
    #     if strike>LTP:
    #         break
    #     atm=strike
    #     atm_index=symbolList.index(local_symbol)

    # print('atm:',atm)


    # #STEP -2 PUT SYMBOLS[ATM-10:ATM+10]
    # put_local_symbols=symbolList[atm_index-10:atm_index+11]
    # app.put_local_symbols=put_local_symbols

    # #DELTA APPEND FOR SYMBOLS
    # for opt in app.put_local_symbols:
    #     app.reqMktData(reqId=500+app.put_local_symbols.index(opt), 
    #                 contract=specificOpt(opt),
    #                 genericTickList="",
    #                 snapshot=False,
    #                 regulatorySnapshot=False,
    #                 mktDataOptions=[])
    #     time.sleep(5)
        

    #STEP 4 DELTA SYMBOL
    print(abs(-0.012-(-0.25)))

    minn=float('inf')
    minnIndex=None
    for delta in app.put_local_symbols_delta:
        delta=-(delta)
        if abs(delta-app.PUT_DELTA) <minn:
            minn=abs(delta-app.PUT_DELTA)
            minnIndex=app.put_local_symbols_delta.index(-delta)

    PUT_DELTA_SYMBOL=app.put_local_symbols[minnIndex]    
    return PUT_DELTA_SYMBOL
 

#STEP-1
def websocket_con():
    app.run()
    time.sleep(3) 

app = TradingApp()      
app.connect("127.0.0.1", 7497, clientId=1)
con_thread = threading.Thread(target=websocket_con, daemon=True)
con_thread.start()
time.sleep(3) 





streamThread = threading.Thread(target=streamSnapshotData, args=(tickers,))
streamThread.start()
time.sleep(3) 



#STEP -2/2
contract_event = threading.Event()
for ticker in tickers:
    contract_event.clear() 
    app.reqContractDetails(tickers.index(ticker), usTechOpt(ticker)) 
    contract_event.wait() 
    app.atmCallOptions[ticker] = atm_call_option(app.df_data[ticker],app.underlyingPrice[ticker])
    app.atmPutOptions[ticker] = atm_put_option(app.df_data[ticker],app.underlyingPrice[ticker])




#reqId >=100 and reqId <200   
def streamSnapshotGreeksOpt(opt_symbols):
        for opt in opt_symbols:
            app.reqMarketDataType
            app.reqMktData(reqId=100+opt_symbols.index(opt), 
                        contract=specificOpt(opt),
                        genericTickList="106",
                        snapshot=False,
                        regulatorySnapshot=False,
                        mktDataOptions=[])
            time.sleep(3)
options = list(app.atmCallOptions.values()) + list(app.atmPutOptions.values())
#['AMZN  230505C00106000', 'AMZN  230505P00102000']
optGreeksStreamThread = threading.Thread(target=streamSnapshotGreeksOpt, args=(options,))
optGreeksStreamThread.start()
time.sleep(3) 




['AMZN  230505C00106000', 'AMZN  230505P00102000']

# def optMarketData(symbol,right):
#     for opt in options:
#         if opt.split()[0]==symbol and opt.split()[1][6] == right:
#             greek=app.greeks[100+options.index(opt)]
#             greek_res={"LTP":greek["OptPrice"],
#                     "Delta":greek["Delta"],
#                     "IV":greek["IV"],
#                     }
#             return greek_res



# for ticker in tickers:
#     symbol=ticker
#     CallGreek=optMarketData(ticker,"C")
#     putGreek=optMarketData(ticker,"P")
#     netIv=CallGreek['IV']-putGreek['Iv']

#     print('Symbol : ',ticker,'Net IV',netIv)



# AMZN Call Greeks : {"LTP":1.2,"Delta":0.23,"IV":0.392} Put Greeks : {"LTP":1.2,"Delta":0.23,"IV":0.392}





```
