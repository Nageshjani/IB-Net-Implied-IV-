```python
from ibapi.client import EClient
from ibapi.wrapper import EWrapper
from ibapi.contract import Contract
import threading
import time
import pandas as pd
from collections import defaultdict



def usTechStk(symbol,sec_type="STK",currency="USD",exchange="ISLAND"):
    contract = Contract()
    contract.symbol = symbol
    contract.secType = sec_type
    contract.currency = currency
    contract.exchange = exchange
    return contract 

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
    contract.lastTradeDateOrContractMonth = "20230512"
    return contract


        

def atm_call_option(contract_df,stock_price):
    df=contract_df
    df_call=df[df['call/put']=="C"]
    df_call=df_call.sort_values(by='symbol')
    symbolList=list(df_call['symbol'])
    
    #STEP -1 ATM
    atm=None
    atm_index=None
    LTP=stock_price
    for local_symbol in symbolList:
        strike=float(local_symbol.split()[1][7:])/1000
        if strike>LTP:
            break
        atm=strike
        atm_index=symbolList.index(local_symbol)

    ticker=local_symbol.split()[0]
    print(ticker,'Atm:',atm)
    call_local_symbols=symbolList[atm_index-5:atm_index+5]
    return call_local_symbols

def atm_put_option(contract_df,stock_price):
    df=contract_df
    df_put=df[df['call/put']=="P"]
    df_put=df_put.sort_values(by='symbol')
    symbolList=list(df_put['symbol'])
    
    #STEP -1 ATM
    atm=None
    atm_index=None
    LTP=stock_price
    for local_symbol in symbolList:
        strike=float(local_symbol.split()[1][7:])/1000
        if strike>LTP:
            break
        atm=strike
        atm_index=symbolList.index(local_symbol)

    print('atm:',atm)
    put_local_symbols=symbolList[atm_index-5:atm_index+5]
    return put_local_symbols
 

# tickers = ["INTC","AMZN","MSFT"]
tickers = ["AMZN","TSLA"]

class TradingApp(EWrapper, EClient):
    def __init__(self):
        EClient.__init__(self,self)
        self.data = {}
        self.df_data = {}
        self.underlyingPrice = {}
        self.CALL_DELTA=0.25
        self.PUT_DELTA=0.25
        self.options=[]
        self.optionChainGreeks={}
        self.bidask=[]
        self.bidaskData={}
        self.optionDeltas=[]
        self.greekMain={}
        self.d_bid={}
        self.d_ask={}
        
        for ticker in tickers:
            self.optionChainGreeks[ticker]=defaultdict()
            self.optionChainGreeks[ticker]['C']=defaultdict()
            self.optionChainGreeks[ticker]['P']=defaultdict()
        

        print(self.optionChainGreeks)


    #reqId < 100:
    def tickPrice(self, reqId, tickType, price, attrib):
        super().tickPrice(reqId, tickType, price, attrib)
        # print('tickType:',tickType,'price:',price)
        if tickType == 4:
            if reqId < 100:
                    self.underlyingPrice[tickers[reqId]] = price
                    # print('wrapper price',price)
                    streaming_event.set()  # set the event to notify that we have received the last price
                    

        if reqId>=1000 and reqId<1100: 
            if tickType == 1:
                local_symbol=self.optionDeltas[reqId-1000]
                if price:     
                        # print(reqId,'bid',price) 
                        self.d_bid[local_symbol]  =price   
                        stream_bid_event.set()  

        if reqId>=1100 and reqId<2000 :
            if tickType == 2:
                local_symbol=self.optionDeltas[reqId-1100]
                if price:     
                        # print(reqId,'ask',price) 
                        self.d_ask[local_symbol]  =price   
                        stream_ask_event.set()     
                    
    def stop_streaming(self,reqId):
        super().cancelMktData(reqId)
        #print('streaming stopped for ',reqId)            
                
        
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
        #print("ContractDetailsEnd. ReqId:", reqId)
        self.df_data[tickers[reqId]] = pd.DataFrame(self.data[reqId])
        
        # fileName=tickers[reqId]+'.csv'
        # csvFile=self.df_data[tickers[reqId]].to_csv(fileName)
        contract_event.set()
    def tickOptionComputation(self, reqId, tickType, tickAttrib,impliedVol, delta, optPrice, pvDividend,gamma, vega, theta, undPrice):
            
            super().tickOptionComputation(reqId, tickType, tickAttrib, impliedVol, delta,optPrice, pvDividend, gamma, vega, theta, undPrice)
            #print('ticktype: ',tickType,'redId: ',reqId)
            if tickType == 11: 
                if delta and impliedVol and optPrice and gamma  and vega and theta:
                            greek={'delta':delta,'impliedVol':impliedVol,'optPrice':optPrice,'gamma':gamma,'vega':vega,'theta':theta,'bid':None,'ask':None }

                            if reqId >=500 and reqId<1000:
                                    local_symbol=self.options[reqId-500]
                                    ticker=local_symbol.split()[0]
                                    right = local_symbol.split()[1][6]
                                    self.optionChainGreeks[ticker][right][local_symbol]=greek
                                    greeks_event.set()

                            if reqId>=5000 and reqId<=6000: 
                                            local_symbol=self.optionDeltas[reqId-5000]
                                            if local_symbol in  self.d_bid and local_symbol in self.d_ask:
                                                    greek['bid']=self.d_bid[local_symbol]
                                                    greek['ask']=self.d_ask[local_symbol]
                                            self.greekMain[local_symbol]=greek
                                            stream_delta_event.set()


                                        



def websocket_con():
    app.run()
    time.sleep(3) 

app = TradingApp()      
app.connect("127.0.0.1", 7497, clientId=1)
con_thread = threading.Thread(target=websocket_con, daemon=True)
con_thread.start()
time.sleep(3) 
print('Connection Started')


#reqId 0-100
streaming_event = threading.Event()
def streamStockLtp(tickers):
    print('LTP Streaming Started')
    for ticker in tickers:    
        app.reqMktData(reqId=tickers.index(ticker), 
                        contract=usTechStk(ticker),
                        genericTickList="",
                        snapshot=False,
                        regulatorySnapshot=False,
                        mktDataOptions=[])
        time.sleep(1)
        # streaming_event.wait()
        # app.stop_streaming(tickers.index(ticker))
       

streamThread = threading.Thread(target=streamStockLtp, args=(tickers,))
streamThread.start()
time.sleep(3) 



#reqId/0-100/Contracts_Download
contract_event = threading.Event()
def Contracts_Download():
    for ticker in tickers:
        contract_event.clear() 
        app.reqContractDetails(tickers.index(ticker), usTechOpt(ticker)) 
        contract_event.wait() 
        print('here --> ',app.underlyingPrice[ticker])
        app.options+=atm_call_option(app.df_data[ticker],app.underlyingPrice[ticker])
        app.options+= atm_put_option(app.df_data[ticker],app.underlyingPrice[ticker])
    #options
    print('Total Options:',len(app.options))

Contracts_Download()
print('Succefuuly Dowloaded Contracts')


#reqId/500-1000/streamOptChain
greeks_event = threading.Event()
def streamOptChain(opt_symbols):
        while True:
            for opt in opt_symbols:
                #print(500+opt_symbols.index(opt))
                greeks_event.clear()
                app.reqMktData(reqId=500+opt_symbols.index(opt), 
                            contract=specificOpt(opt),
                            genericTickList="106",
                            snapshot=False,
                            regulatorySnapshot=False,
                            mktDataOptions=[])
                greeks_event.wait() 
                app.stop_streaming(500+opt_symbols.index(opt))
            
            
            time.sleep(300)


# options = ['AMZN 230512C106000','AMZN 230512P106000','AMZN 230512C105000','AMZN 230512P106000']
options = app.options
optGreeksStreamThread = threading.Thread(target=streamOptChain, args=(options,))
optGreeksStreamThread.start()
time.sleep(3) 
print('OptionChain Activated!')


#app.optionDeltas
def updateDeltas():
     while True:
        print('updateDelta')
        app.optionDeltas=[]
        for ticker in tickers:
            callGreeks=app.optionChainGreeks[ticker]['C']
            putGreeks=app.optionChainGreeks[ticker]['P']
            resCallLocalSymbol=None
            resPutLocalSymbol=None

            if app.optionChainGreeks[ticker]['C'] and app.optionChainGreeks[ticker]['P']:
                minn=float('inf')
                for local_symbol in  callGreeks:
                    delta=callGreeks[local_symbol]['delta']
                    if abs(delta-app.CALL_DELTA) <minn:
                        minn=abs(delta-app.CALL_DELTA)
                        resCallLocalSymbol=local_symbol
                       
                
                minn=float('inf')
                for local_symbol in  putGreeks:
                    delta=putGreeks[local_symbol]['delta']
                    delta=-(delta)
                    if abs(delta-app.PUT_DELTA) <minn:
                        minn=abs(delta-app.PUT_DELTA)
                        resPutLocalSymbol=local_symbol
                
                if resCallLocalSymbol and resPutLocalSymbol:
                    # print('L 305 Finally Found Deltas!')
                    # print(resCallLocalSymbol)
                    # print(resPutLocalSymbol)
                    app.optionDeltas.append(resCallLocalSymbol)
                    app.optionDeltas.append(resPutLocalSymbol)
        #print('L 310 app.optionDeltas',app.optionDeltas)
        time.sleep(60)
    

updateDeltaThread=threading.Thread(target=updateDeltas) 
updateDeltaThread.start()
time.sleep(2)    







#reqId/5000-6000/streamDeltas
stream_delta_event=threading.Event()
def streamDeltas():
        
        while True:
           
            opt_symbols=app.optionDeltas
            if len(app.optionDeltas)==(len(tickers)*2):
                for opt in opt_symbols:
                    stream_delta_event.clear()
                    app.reqMktData(reqId=5000+opt_symbols.index(opt), 
                                contract=specificOpt(opt),
                                genericTickList="106",
                                snapshot=False,
                                regulatorySnapshot=False,
                                mktDataOptions=[])
    
                    stream_delta_event.wait()
                    app.stop_streaming(5000+opt_symbols.index(opt))
            #print('Main Greek',app.greekMain)
            time.sleep(10)
            


options = app.optionDeltas
streamDeltasThread = threading.Thread(target=streamDeltas)
streamDeltasThread.start()
time.sleep(3) 


def printDeltas():
        while True: 
            if len(app.optionDeltas)==(len(tickers)*2):
                if len(app.greekMain)==len(app.optionDeltas):
                    i=0
                    while i<len(app.optionDeltas):
                        callGreek=app.greekMain[app.optionDeltas[i]]
                        putGreek=app.greekMain[app.optionDeltas[i+1]]
                        netIv=callGreek['impliedVol']-putGreek['impliedVol']
                        ticker=app.optionDeltas[i].split()[0]
                        callask=callGreek['ask']
                        callbid=callGreek['bid']
                        putask=putGreek['ask']
                        putbid=putGreek['bid']
                        
                        print(ticker,'NetIv: ',netIv,'Call/Bid: ',callbid,'Call/Ask: ',callask,'Put/Bid: ',putbid,'Put/Ask: ',putask)
                        i=i+2
            time.sleep(5)
                
        
printDeltasThread = threading.Thread(target=printDeltas)
printDeltasThread.start()
time.sleep(3) 

 







stream_bid_event=threading.Event()
def streamBid():
        
        while True:
            # print('Inside streambidask')
           
            opt_symbols=app.optionDeltas
            
            if len(app.optionDeltas)==(len(tickers)*2):
                # app.d_bid={}
                for opt in opt_symbols:
                        stream_bid_event.clear()
                        app.reqMktData(reqId=1000+opt_symbols.index(opt), 
                                    contract=specificOpt(opt),
                                    genericTickList="",
                                    snapshot=False,
                                    regulatorySnapshot=False,
                                    mktDataOptions=[])
        
                        stream_bid_event.wait()
                        app.stop_streaming(1000+opt_symbols.index(opt))
            # print('greek main with bidask',app.greekMain)  
            time.sleep(3)
            


streamBidAskThread = threading.Thread(target=streamBid)
streamBidAskThread.start()
time.sleep(3) 

stream_ask_event=threading.Event()
def streamAsk():
        
        while True:
           
            opt_symbols=app.optionDeltas
            if len(app.optionDeltas)==(len(tickers)*2):
                # app.d_ask={}
                for opt in opt_symbols:
                        stream_ask_event.clear()
                        app.reqMktData(reqId=1100+opt_symbols.index(opt), 
                                    contract=specificOpt(opt),
                                    genericTickList="",
                                    snapshot=False,
                                    regulatorySnapshot=False,
                                    mktDataOptions=[])
        
                        stream_ask_event.wait()
                        app.stop_streaming(1100+opt_symbols.index(opt))
           
            time.sleep(3)
            


streamAskThread = threading.Thread(target=streamAsk)
streamAskThread.start()
time.sleep(3) 

```
