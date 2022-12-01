# AGSI-GitHub (PowerBI + Python)

## [!! PowerBI file is here !!](https://github.com/Nostr77/AGSI/blob/main/agsi-github.pbix?raw=true)


## Overwiew

### Name of Project:
European Natural Gas Data Automatic Analytical Dashboard (Stock, Injection, Withdrawal, Withdrawal Capacity, Consumption)
### Source: 
[Gas Infrastructure Europe - GIE](https://agsi.gie.eu/)  Level of granularity - Country, Day - is used to get all data from [API](https://agsi.gie.eu/api?country=AT&from=2022-11-17&to=2022-11-20&page=1&size=1500). 

Timeline: 2012's first day to the present. The PoweBI Dashboard combines data from [pre-transformed data](https://github.com/Nostr77/AGSI/blob/main/base0.csv) and last dates (not present in pre-transformed data) directly from API for the sake of quick and wise downloading.

### Data Flows 
![Schema](https://github.com/Nostr77/AGSI/raw/main/Schema.JPG)

### Outcome file(s):
File is based on archive data from Github [PowerBI Github](https://github.com/Nostr77/AGSI/blob/main/agsi-github.pbix)
![Schema](https://github.com/Nostr77/AGSI/raw/main/Capture1.JPG)
![Schema](https://github.com/Nostr77/AGSI/raw/main/Capture3.JPG)
![Schema](https://github.com/Nostr77/AGSI/raw/main/Capture2.JPG)





### Target Audience: 
Analysts, Economic Mass Media




## Py1a. Data gathering (Python)
Input: API 
Output: CSV

```
import requests
import json 
import time
import pandas as pd

countrylist=['AT','BE','BG','CZ','DE','DK','ES','FR','GB','HR','HU','IE','IT','LV','NL','PL','PT','RO','SE','SK','UA']

b=[]
for country in countrylist:
    url='https://agsi.gie.eu/api?country='+country+'&from=2020-01-01&to=2022-11-10&page=1&size=15000' #+
    response = requests.get(url)
    print(response.content)
    copia=response.content
    a=json.loads(response.content)['data']
    print(country)
    b=a+b
    time.sleep(3)

c=b

adf=pd.DataFrame()
#for k,v in b[0].items():
#    if k=='full':
#        k='full_'
#    if k!='info':
#        adf[k]=v
#        print(v)

for i in range(0,len(b)):
    for k,v in b[i].items():
        if k=='full':
            k='full_'
        if k!='info':
            #adf[k]=v
            adf.loc[i,k]=v
    if i%100==0:
        print(i)

adf.to_csv(r'c:\work\agsi\prom2022.csv')
```


## Py1b. Cleaning and SQL injection (Python)

Input: CSV 
Output: SQL

```
import pyodbc
import pandas as pd
pd.options.mode.chained_assignment = None

# Read income dataframe
df = pd.read_csv("c:\\work\\agsi\prom2022.csv")

# Income Dataframe Cleaning
df=df.drop(['Unnamed: 0'], axis=1)
df.gasInStorage[df.gasInStorage=='-']=0
df.injection[df.injection=='-']=0
df.consumption[df.consumption=='-']=0
df.consumptionFull[df.consumptionFull=='-']=0
df.withdrawal[df.withdrawal=='-']=0
df.netWithdrawal[df.netWithdrawal=='-']=0
df.workingGasVolume[df.workingGasVolume=='-']=0
df.injectionCapacity[df.injectionCapacity=='-']=0
df.withdrawalCapacity[df.withdrawalCapacity=='-']=0
df.trend[df.trend=='-']=0
df.full_[df.full_=='-']=0


# Calculate buckets for 1000 records inserting
bucket = list(range(0,len(df.code)-1,1000))
bucket.append(len(df.code)-1)

# Connect to SQL database via ODBC connection
password = r'XXXXXX' 
cnxn = pyodbc.connect(r'Driver={SQL Server};Server=tcp:XXXXX.database.windows.net,1433;Database=agsi;Uid=XXXXX;Pwd='+password+';Encrypt=yes;TrustServerCertificate=no;Connection Timeout=30;')
cursor = cnxn.cursor()

# Insert Dataframe into SQL Server:
for bu in bucket:
    string=r"INSERT INTO agsi (code,url,gasDayStart,gasInStorage,consumption,consumptionFull,injection,withdrawal,netWithdrawal,workingGasVolume,injectionCapacity,withdrawalCapacity,status,trend,full_) values"
    i=bu
    string=string+" ('"+df.code[i]+"','"+df.url[i]+"','"+str(df.gasDayStart[i])+"',"+str(df.gasInStorage[i])+","+str(df.consumption[i])+","+str(df.consumptionFull[i])+","+str(df.injection[i])+","+str(df.withdrawal[i])+","+str(df.netWithdrawal[i])+","+str(df.workingGasVolume[i])+","+str(df.injectionCapacity[i])+","+str(df.withdrawalCapacity[i])+",'"+df.status[i]+"',"+str(df.trend[i])+","+str(df.full_[i])+")"
    for i in range(bu+1,bu+min(1000, len(df.code)-bu-1)): #len(df.code)):
        string=string+", ('"+df.code[i]+"','"+df.url[i]+"','"+str(df.gasDayStart[i])+"',"+str(df.gasInStorage[i])+","+str(df.consumption[i])+","+str(df.consumptionFull[i])+","+str(df.injection[i])+","+str(df.withdrawal[i])+","+str(df.netWithdrawal[i])+","+str(df.workingGasVolume[i])+","+str(df.injectionCapacity[i])+","+str(df.withdrawalCapacity[i])+",'"+df.status[i]+"',"+str(df.trend[i])+","+str(df.full_[i])+")"
        #print(i)
    print('Progress: '+str(bu)+' / '+str(len(df.code)-1))
    cursor.execute(string)

cnxn.commit()
cursor.close()

```

## Py2: Data injection on schedule

Input: API 
Output: CSV & SQL

```
import requests
import json 
import time
import pyodbc
import pandas as pd
pd.options.mode.chained_assignment = None

# Connect to SQL database via ODBC connection
password = r'XXX' 
cnxn = pyodbc.connect(r'Driver={SQL Server};Server=tcp:XXX.database.windows.net,1433;Database=agsi;Uid=XXX;Pwd='+password+';Encrypt=yes;TrustServerCertificate=no;Connection Timeout=30;')
cursor = cnxn.cursor()

# Grab start date and last date from SQL and today
data_last=cursor.execute('select max(gasDayStart) from agsi').fetchall()
date1=data_last[0][0]
date2=str(pd.to_datetime('today').normalize().date())

# scraping API
countrylist=['AT','BE','BG','CZ','DE','DK','ES','FR','GB*','HR','HU','IE','IT','LV','NL','PL','PT','RO','SE','SK','UA']

b=[]
for country in countrylist:
    #country="RO"
    url='https://agsi.gie.eu/api?country='+country+'&from='+date1+'&to='+date2+'&page=1&size=15000' #+
    response = requests.get(url)
    #print(response.content)
    copia=response.content
    a=json.loads(response.content)['data']
    print(country)
    b=a+b
    time.sleep(3)

c=b

adf=pd.DataFrame()

for i in range(0,len(b)):
    for k,v in b[i].items():
        if k=='full':
            k='full_'
        if k!='info':
            #adf[k]=v
            adf.loc[i,k]=v
    if i%100==0:
        print(i)

adf.to_csv(r'c:\work\agsi\prom'+date2+'.csv')
adf=adf[adf.gasDayStart!=date1].reset_index(drop=True)

df = adf

# Income Dataframe Cleaning
df.gasInStorage[df.gasInStorage=='-']=0
df.injection[df.injection=='-']=0
df.consumption[df.consumption=='-']=0
df.consumptionFull[df.consumptionFull=='-']=0
df.withdrawal[df.withdrawal=='-']=0
df.netWithdrawal[df.netWithdrawal=='-']=0
df.workingGasVolume[df.workingGasVolume=='-']=0
df.injectionCapacity[df.injectionCapacity=='-']=0
df.withdrawalCapacity[df.withdrawalCapacity=='-']=0
df.trend[df.trend=='-']=0
df.full_[df.full_=='-']=0

df.code[df.code=='GB*']='GB'
df.url[df.url=='GB*']='GB'

# Insert Dataframe into SQL Server:
string=r"INSERT INTO agsi (code,url,gasDayStart,gasInStorage,consumption,consumptionFull,injection,withdrawal,netWithdrawal,workingGasVolume,injectionCapacity,withdrawalCapacity,status,trend,full_) values"
i=0
string=string+" ('"+df.code[i]+"','"+df.url[i]+"','"+str(df.gasDayStart[i])+"',"+str(df.gasInStorage[i])+","+str(df.consumption[i])+","+str(df.consumptionFull[i])+","+str(df.injection[i])+","+str(df.withdrawal[i])+","+str(df.netWithdrawal[i])+","+str(df.workingGasVolume[i])+","+str(df.injectionCapacity[i])+","+str(df.withdrawalCapacity[i])+",'"+df.status[i]+"',"+str(df.trend[i])+","+str(df.full_[i])+")"
for i in range(1,len(df.code)): 
    string=string+", ('"+df.code[i]+"','"+df.url[i]+"','"+str(df.gasDayStart[i])+"',"+str(df.gasInStorage[i])+","+str(df.consumption[i])+","+str(df.consumptionFull[i])+","+str(df.injection[i])+","+str(df.withdrawal[i])+","+str(df.netWithdrawal[i])+","+str(df.workingGasVolume[i])+","+str(df.injectionCapacity[i])+","+str(df.withdrawalCapacity[i])+",'"+df.status[i]+"',"+str(df.trend[i])+","+str(df.full_[i])+")"
#print(string)
cursor.execute(string)

cnxn.commit()
cursor.close()

```

## Power Query M 1. Finding last date in the Archive and API Online Direct Query update in PowerBI 

```
let
     SourceD = Csv.Document(Web.Contents("https://raw.githubusercontent.com/Nostr77/AGSI/main/base0.csv"),[Delimiter=",", Columns=3, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted HeadersD" = Table.PromoteHeaders(SourceD, [PromoteAllScalars=true]),
    #"Changed TypeD" = Table.TransformColumnTypes(#"Promoted HeadersD",{{"code", type text}, {"url", type text}, {"gasDayStart", type datetime}}),
    #"Changed Type1D" = Table.SelectRows(#"Changed TypeD", each [code]="SK" ),
    gasDayStart = #"Changed Type1D"[gasDayStart],
    #"Sorted ItemsD" = Date.ToText(Date.AddDays(DateTime.Date(List.First(List.Sort(gasDayStart,Order.Descending))),+1),[Format="yyyy-MM-dd"]),
    
    Source1 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=AT&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table1" = Table.FromRecords({Source1}),
    
    Source2 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=BE&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table2" = Table.FromRecords({Source2}),
    
    Source3 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=BG&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table3" = Table.FromRecords({Source3}),
    
    Source4 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=CZ&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table4" = Table.FromRecords({Source4}),
    
    Source5 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=DE&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table5" = Table.FromRecords({Source5}),
    
    Source6 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=DK&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table6" = Table.FromRecords({Source6}),
    
    Source7 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=ES&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table7" = Table.FromRecords({Source7}),
    
    Source8 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=FR&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table8" = Table.FromRecords({Source8}),
    
    Source9 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=HR&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table9" = Table.FromRecords({Source9}),
    
    Source10 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=HU&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table10" = Table.FromRecords({Source10}),
    
    Source11 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=IT&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table11" = Table.FromRecords({Source11}),
    
    Source12 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=LV&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table12" = Table.FromRecords({Source12}),
    
    Source13 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=NL&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table13" = Table.FromRecords({Source13}),
    
    Source14 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=PL&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table14" = Table.FromRecords({Source14}),
    
    Source15 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=PT&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table15" = Table.FromRecords({Source15}),
    
    Source16 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=RO&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table16" = Table.FromRecords({Source16}),
    
    Source17 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=SE&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table17" = Table.FromRecords({Source17}),
    
    Source18 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=SK&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table18" = Table.FromRecords({Source18}),
    
    Source19 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=UA&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table19" = Table.FromRecords({Source19}),
    
    Source20 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=IE&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table20" = Table.FromRecords({Source20}),
    
    Source21 = Json.Document(Web.Contents("https://agsi.gie.eu/api?country=GB*&from="&#"Sorted ItemsD"&"&to="&Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()),0),"yyyy-MM-dd")&"&page=1&size=150")),
    #"Converted to Table21" = Table.FromRecords({Source21}),
    
    
    #"Converted to Table" = Table.Combine({ #"Converted to Table1", #"Converted to Table2", #"Converted to Table3", #"Converted to Table4", #"Converted to Table5", #"Converted to Table6", #"Converted to Table7", #"Converted to Table8", #"Converted to Table9", #"Converted to Table10", #"Converted to Table11", #"Converted to Table12", #"Converted to Table13", #"Converted to Table14", #"Converted to Table15", #"Converted to Table16", #"Converted to Table17", #"Converted to Table18", #"Converted to Table19",  #"Converted to Table21"}),
    
    #"Expanded data" = Table.ExpandListColumn(#"Converted to Table", "data"),
    
    #"Expanded data1" = Table.ExpandRecordColumn(#"Expanded data", "data", {"code", "url", "gasDayStart", "gasInStorage", "consumption", "consumptionFull", "injection", "withdrawal", "netWithdrawal", "workingGasVolume", "injectionCapacity", "withdrawalCapacity", "status", "trend", "full"}, {"data.code", "data.url", "data.gasDayStart", "data.gasInStorage", "data.consumption", "data.consumptionFull", "data.injection", "data.withdrawal", "data.netWithdrawal", "data.workingGasVolume", "data.injectionCapacity", "data.withdrawalCapacity", "data.status", "data.trend", "data.full"}),
    #"Changed Type" = Table.TransformColumnTypes(#"Expanded data1",{{"data.code", type text}, {"data.url", type text}, {"data.gasDayStart", type date}, {"data.gasInStorage", type number}, {"data.consumption", type number}, {"data.consumptionFull", type number}, {"data.injection", type number}, {"data.withdrawal", type number}, {"data.netWithdrawal", type number}, {"data.workingGasVolume", type number}, {"data.injectionCapacity", type number}, {"data.withdrawalCapacity", type number}, {"data.status", type text}, {"data.trend", type number}, {"data.full", type number}}),
    #"Added Custom" = Table.AddColumn(#"Changed Type", "data.code1", each Text.Range([data.code],0,2)),
    #"Removed Columns" = Table.RemoveColumns(#"Added Custom",{"last_page", "total", "dataset", "gas_day"}),
    #"Renamed Columns" = Table.RenameColumns(#"Removed Columns",{{"data.code1", "code"}, {"data.url", "url"}, {"data.gasDayStart", "gasDayStart"}, {"data.gasInStorage", "gasInStorage"}, {"data.consumption", "consumption"}, {"data.consumptionFull", "consumptionFull"}, {"data.injection", "injection"}, {"data.withdrawal", "withdrawal"}, {"data.netWithdrawal", "netWithdrawal"}, {"data.workingGasVolume", "workingGasVolume"}, {"data.injectionCapacity", "injectionCapacity"}, {"data.withdrawalCapacity", "withdrawalCapacity"}, {"data.status", "status"}, {"data.trend", "trend"}, {"data.full", "full_"}})
in
    #"Renamed Columns"
```


## Power Query M 2. Combining Update and Archive data in PowerBI 

```
let
    Source = Csv.Document(Web.Contents("https://raw.githubusercontent.com/Nostr77/AGSI/main/base0.csv"),[Delimiter=",", Columns=15, Encoding=65001, QuoteStyle=QuoteStyle.None]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",{{"code", type text}, {"url", type text}, {"gasDayStart", type datetime}, {"gasInStorage", type number}, {"consumption", Int64.Type}, {"consumptionFull", type number}, {"injection", type number}, {"withdrawal", type number}, {"netWithdrawal", type number}, {"workingGasVolume", type number}, {"injectionCapacity", type number}, {"withdrawalCapacity", type number}, {"status", type text}, {"trend", type number}, {"full_", type number}}),
    #"Appended Query" = Table.Combine({#"Changed Type", Query2}),
    #"Changed Type1" = Table.TransformColumnTypes(#"Appended Query",{{"gasDayStart", type date}})
in
    #"Changed Type1"
```

