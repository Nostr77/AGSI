# AGSI

## Overwiew

### Name of Project:
European Natural Gas Data Automatic Analytical Dashboard (Stock, Injection, Withdrawal, Withdrawal Capacity, Consumption)
### Source: 
[Gas Infrastructure Europe - GIE](https://agsi.gie.eu/)  Level of granularity - Country, Day - is used to get all data from API. Timeline: 2012's first day to the present. The Dashboard combines data from [pre-transformed data](https://github.com/Nostr77/AGSI/blob/main/base0.csv) and last dates (not present in pre-transformed data) directly from API for the sake of quick downloading and wise loading.
### Outcome file(s):
1) Based on archive data from Github - [PowerBI Github](https://github.com/Nostr77/AGSI/blob/main/agsi-github.pbix)
2) Based on SQL Server archive data (to be uploaded soon)


### Target Audience: 
Analysts, Economic Mass Media


## Data Flows (ETL schema) 

![Schema](https://github.com/Nostr77/AGSI/raw/main/Schema.JPG)


## Pace 1a. Data gathering (Python)
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


## Pace 1b. Cleaning and SQL injection (Python)

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

