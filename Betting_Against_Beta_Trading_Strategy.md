## Betting Against Beta

**Systematic Risk**: Part of risk that cannot be diversified away.

**Beta**: Sensitivity of a firm to systematic risk.

Comapnies with **high beta** are known as **aggressive stocks** or **growth stocks**. In other words, companies which have high growth or high uncertainty or high exposure to systmeatic risk have high beta. On the other hand, companies which have low growth usually have **low beta** and are known as **defensives**.

## Betting Against Beta Trading strategy
The strategy of *Betting against Beta* was proposed by Andrea Frazzini & Lasse Heje Pederson in 2014 in their paper titled "Betting against Beta". The idea is that **constrained investors bid-up high-beta assets** so the assets are **overvalued** and therefore their **alphas (returns above the expected value)** are **low** (possibly negative) => **potential losers**. On the other hand firms with **low-beta** have relatively **high alphas** => **potential winners**. The trading strategy is to **long leveraged low-beta assets** and **short high-beta assets**.

**Capital Asset Pricing Model (CAPM)**:\
Expected Return = E(R) = R$_f$ + $\beta$ (R$_m$ - R$_f$)

where\
R$_f$ = Risk-free Return; R$_m$ = Market Return; $\beta$ = Historical beta of the stock

Alpha or $\alpha$ = Extraordinary Return =  Actual Return - Expected Return

**Steps in strategy**:
1. Get historical beta values for all the firms.
2. Sort companies based on the beta values. One can sort the companies by a) industry b) market c) asset class or d) country.
3. Find the median value of beta.
4. Divide the companies on above median and below median.
5. Go long on low beta stocks and short on high beta stocks.

### Importing necessary modules


```python
import pandas as pd
from bs4 import BeautifulSoup
import bs4
import requests
from datetime import date
import plotly.express as px
import numpy as np
import seaborn as sns
import matplotlib.pyplot as plt
from format import format
```

### Gettting the list of stocks from NASDAQ


```python
# getting a list of all the stocks from NASDAQ with Market Cap > 0

df = pd.read_csv('nasdaq_screener.csv')
df = df[df['Market Cap'] > 0]
df.reset_index(inplace=True, drop=True)
stocks = list(df['Symbol'])
len(stocks)
```




    5743



### Functions for web-scraping the beta values of companies


```python
# function for getting the historical beta values of stocks from marketwatch.com using the stock ticker
def get_beta_values(ticker):
    '''
    The function returns the historical value of a stock ticker 
    
    inputs:
    ticker: stock ticker of the firm
    
    output:
    returns the hostorical beta value of the stock 
    '''
    
    urlstock = 'https://www.marketwatch.com/investing/stock/'+ticker
    text_soup_stock = BeautifulSoup(requests.get(urlstock, allow_redirects=False).text,"lxml") #read in
    titles_stock = text_soup_stock.findAll('small', {'class': 'label'})
    
    betalist = []
    
    for title in titles_stock:
        if 'Beta' in title.text:
            betalist.append([td.text for td in title.findNextSiblings(attrs={'class': 'primary'}) if td.text])

    return text_parse(betalist[0][0])



def text_parse(text):
    '''
    This function to convert the string outputs of the 
    financial statements to float values  
    '''
    if len(text) == 2:
        text = text[0]
    text = text.strip('(').strip(')').strip('$').strip(' ')
    if text == '-' or text == '' or text =='N/A':
        return None
    elif text[-1] == 'T':
        return float(text.strip('T'))*1e12
    elif text[-1] == 'B':
        return float(text.strip('B'))*1e9
    elif text[-1] == 'M':
        return float(text.strip('M'))*1e6
    elif text[-1] == 'K':
        return float(text.strip('K'))*1e3
    else:
        return float(text)
        
```

### Getting the beta values for all the stocks


```python
import time
columns = ['symbol', 'beta']
beta_df = pd.DataFrame(columns=columns)

for symbol in stocks:
    #print(symbol)
    try:
        beta = get_beta_values(symbol)
        if beta == None:
            continue
        else:
            beta_df = beta_df.append(pd.DataFrame([[symbol, beta]],columns=columns),ignore_index=True)
    except IndexError:
        pass
```

### Results


```python
beta_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>symbol</th>
      <th>beta</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>A</td>
      <td>0.99</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AA</td>
      <td>1.53</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AACG</td>
      <td>0.19</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AACQ</td>
      <td>0.30</td>
    </tr>
    <tr>
      <th>4</th>
      <td>AAIC</td>
      <td>1.47</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>5015</th>
      <td>ZUO</td>
      <td>1.15</td>
    </tr>
    <tr>
      <th>5016</th>
      <td>ZVO</td>
      <td>1.23</td>
    </tr>
    <tr>
      <th>5017</th>
      <td>ZYME</td>
      <td>0.92</td>
    </tr>
    <tr>
      <th>5018</th>
      <td>ZYNE</td>
      <td>0.98</td>
    </tr>
    <tr>
      <th>5019</th>
      <td>ZYXI</td>
      <td>0.85</td>
    </tr>
  </tbody>
</table>
<p>5020 rows × 2 columns</p>
</div>



### Distribution of beta values


```python
fig, ax = plt.subplots(figsize=(10,5))
ax.set_xlabel('Beta value of firms',fontsize=20)
ax.set_ylabel('Frequency',fontsize=20)
beta_df['beta'].plot.hist(bins=15,log=True,ax=ax)
```




    <AxesSubplot:xlabel='Beta value of firms', ylabel='Frequency'>




    
![png](Betting_Against_Beta_Trading_Strategy_files/Betting_Against_Beta_Trading_Strategy_12_1.png)
    


### Median beta value


```python
median_beta = beta_df['beta'].median()
median_beta
```




    1.0



### High-Beta firms -> Aggressives (Potential Losers)
These are the firms above the median beta value


```python
list(beta_df['symbol'][beta_df['beta'] > median_beta].values)[:10] # printing only first 50 values
```




    ['AA', 'AAIC', 'AAL', 'AAOI', 'AAPL', 'AAT', 'AAWW', 'AB', 'ABCB', 'ABCL']



### Low-Beta firms -> Defensives (Potential Winners)
These are the firms below the median beta value


```python
list(beta_df['symbol'][beta_df['beta'] < median_beta].values)
```




    ['A',
     'AACG',
     'AACQ',
     'AAME',
     'AAN',
     'AAON',
     'AAP',
     'ABB',
     'ABBV',
     'ABC',
     'ABCM',
     'ABEV',
     'ABGI',
     'ABIO',
     'ABMD',
     'ABNB',
     'ABST',
     'ABT',
     'AC',
     'ACAC',
     'ACEL',
     'ACET',
     'ACEV',
     'ACHV',
     'ACI',
     'ACIU',
     'ACND',
     'ACRS',
     'ACST',
     'ACTG',
     'ADAG',
     'ADC',
     'ADCT',
     'ADIL',
     'ADM',
     'ADMA',
     'ADMP',
     'ADN',
     'ADOC',
     'ADT',
     'ADTN',
     'ADTX',
     'ADUS',
     'ADV',
     'ADXN',
     'AEE',
     'AEHL',
     'AEHR',
     'AEM',
     'AEMD',
     'AENZ',
     'AEP',
     'AESE',
     'AEY',
     'AEYE',
     'AFGC',
     'AFI',
     'AFIB',
     'AFRM',
     'AG',
     'AGBA',
     'AGFS',
     'AGI',
     'AGMH',
     'AGNC',
     'AGR',
     'AGRO',
     'AGX',
     'AHAC',
     'AHC',
     'AHPI',
     'AIH',
     'AIHS',
     'AIKI',
     'AIRC',
     'AIRT',
     'AIV',
     'AIZ',
     'AJG',
     'AKAM',
     'AKER',
     'AKIC',
     'AKR',
     'AKTX',
     'AKU',
     'AKUS',
     'ALAC',
     'ALC',
     'ALCO',
     'ALE',
     'ALGS',
     'ALIM',
     'ALJJ',
     'ALL',
     'ALLK',
     'ALLT',
     'ALNY',
     'ALPN',
     'ALRM',
     'ALRS',
     'ALSK',
     'ALSN',
     'ALT',
     'ALUS',
     'ALX',
     'ALXN',
     'ALXO',
     'ALYA',
     'AM',
     'AMCX',
     'AMED',
     'AMGN',
     'AMH',
     'AMHC',
     'AMN',
     'AMOV',
     'AMPH',
     'AMRB',
     'AMRK',
     'AMSF',
     'AMSWA',
     'AMT',
     'AMTBB',
     'AMTI',
     'AMTX',
     'AMWL',
     'AMX',
     'AMYT',
     'AMZN',
     'ANAT',
     'ANDA',
     'ANDE',
     'ANGI',
     'ANGO',
     'ANIK',
     'ANIX',
     'ANNX',
     'ANPC',
     'ANTE',
     'ANY',
     'AON',
     'AONE',
     'AOS',
     'AP',
     'APD',
     'APDN',
     'APEI',
     'APM',
     'APOP',
     'APPH',
     'APR',
     'APRE',
     'APRN',
     'APSG',
     'APTS',
     'APTX',
     'APWC',
     'APXT',
     'AQB',
     'AQN',
     'ARBG',
     'ARC',
     'ARCC',
     'ARCE',
     'ARCH',
     'ARCT',
     'ARDS',
     'ARE',
     'AREC',
     'ARGO',
     'ARGX',
     'ARKO',
     'ARKR',
     'ARL',
     'ARPO',
     'ARQT',
     'ARTL',
     'ARTNA',
     'ARTW',
     'ARYA',
     'ASAN',
     'ASAQ',
     'ASLE',
     'ASLN',
     'ASND',
     'ASO',
     'ASPL',
     'ASPU',
     'ASR',
     'ASRT',
     'ASRV',
     'ASTC',
     'ASX',
     'ASYS',
     'AT',
     'ATAC',
     'ATAX',
     'ATC',
     'ATCX',
     'ATEN',
     'ATGE',
     'ATHA',
     'ATHE',
     'ATHM',
     'ATHX',
     'ATIF',
     'ATNF',
     'ATNI',
     'ATO',
     'ATOS',
     'ATR',
     'ATRI',
     'ATSG',
     'ATTO',
     'ATVI',
     'ATXI',
     'AU',
     'AUBN',
     'AUDC',
     'AUPH',
     'AUTO',
     'AUVI',
     'AUY',
     'AVA',
     'AVAN',
     'AVB',
     'AVCT',
     'AVDL',
     'AVEO',
     'AVGR',
     'AVIR',
     'AVLR',
     'AVNW',
     'AVO',
     'AWH',
     'AWK',
     'AWR',
     'AWRE',
     'AXR',
     'AXS',
     'AY',
     'AYLA',
     'AYTU',
     'AZEK',
     'AZN',
     'AZO',
     'AZRE',
     'AZRX',
     'AZYO',
     'BABA',
     'BAH',
     'BAND',
     'BAP',
     'BASI',
     'BATRA',
     'BATRK',
     'BAX',
     'BBDC',
     'BBGI',
     'BBI',
     'BBIG',
     'BBQ',
     'BBW',
     'BCAB',
     'BCBP',
     'BCDA',
     'BCE',
     'BCH',
     'BCLI',
     'BCML',
     'BCOV',
     'BCOW',
     'BCPC',
     'BCTG',
     'BCTX',
     'BCYC',
     'BDSI',
     'BDX',
     'BEDU',
     'BEEM',
     'BEKE',
     'BELFA',
     'BENE',
     'BEP',
     'BEPC',
     'BERY',
     'BEST',
     'BFC',
     'BFI',
     'BFIN',
     'BFLY',
     'BFRA',
     'BG',
     'BGNE',
     'BGS',
     'BHAT',
     'BHSE',
     'BHTG',
     'BIDU',
     'BIIB',
     'BILI',
     'BIMI',
     'BIO',
     'BIOC',
     'BIOL',
     'BIPC',
     'BIVI',
     'BJ',
     'BKE',
     'BKEP',
     'BKH',
     'BKI',
     'BKSC',
     'BKYI',
     'BL',
     'BLBD',
     'BLI',
     'BLIN',
     'BLL',
     'BLPH',
     'BLRX',
     'BLSA',
     'BLTS',
     'BLU',
     'BLUW',
     'BMRA',
     'BMY',
     'BNGO',
     'BNL',
     'BNR',
     'BNS',
     'BNSO',
     'BNTC',
     'BNTX',
     'BOCH',
     'BOMN',
     'BOSC',
     'BOTJ',
     'BOWX',
     'BOXL',
     'BPMP',
     'BPT',
     'BPTH',
     'BPTS',
     'BR',
     'BRBR',
     'BREZ',
     'BRID',
     'BRLI',
     'BRMK',
     'BRO',
     'BROG',
     'BRP',
     'BRPA',
     'BRQS',
     'BRT',
     'BSAC',
     'BSBK',
     'BSGM',
     'BSM',
     'BSPE',
     'BSQR',
     'BTAQ',
     'BTAQU',
     'BTI',
     'BTRS',
     'BUD',
     'BVN',
     'BVXV',
     'BWAC',
     'BWB',
     'BWEN',
     'BWFG',
     'BWMX',
     'BWXT',
     'BXRX',
     'BYFC',
     'BYND',
     'BYSI',
     'CAAS',
     'CABO',
     'CACI',
     'CAG',
     'CAH',
     'CAJ',
     'CALB',
     'CALM',
     'CAN',
     'CANG',
     'CAP',
     'CAPL',
     'CAPR',
     'CARR',
     'CARV',
     'CAS',
     'CASI',
     'CASY',
     'CATB',
     'CATC',
     'CATO',
     'CB',
     'CBAH',
     'CBAN',
     'CBAT',
     'CBAY',
     'CBB',
     'CBFV',
     'CBLI',
     'CBMB',
     'CBPO',
     'CBSH',
     'CBTX',
     'CBU',
     'CBZ',
     'CCAC',
     'CCAP',
     'CCCC',
     'CCEP',
     'CCI',
     'CCIV',
     'CCJ',
     'CCM',
     'CCNC',
     'CCOI',
     'CCRC',
     'CCU',
     'CCV',
     'CCX',
     'CD',
     'CDK',
     'CDTX',
     'CDXC',
     'CDZI',
     'CEA',
     'CELC',
     'CELP',
     'CEMI',
     'CENT',
     'CENTA',
     'CERE',
     'CERN',
     'CETX',
     'CFB',
     'CFBK',
     'CFFN',
     'CFII',
     'CFIV',
     'CFRX',
     'CGA',
     'CGBD',
     'CGIX',
     'CGNT',
     'CGRO',
     'CHCI',
     'CHCO',
     'CHD',
     'CHE',
     'CHEK',
     'CHFS',
     'CHGG',
     'CHH',
     'CHK',
     'CHKP',
     'CHMA',
     'CHNR',
     'CHPM',
     'CHPT',
     'CHRW',
     'CHS',
     'CHT',
     'CHWY',
     'CIDM',
     'CIH',
     'CIIC',
     'CIM',
     'CINR',
     'CIO',
     'CIXX',
     'CIZN',
     'CJJD',
     'CL',
     'CLA',
     'CLAR',
     'CLBK',
     'CLBS',
     'CLDB',
     'CLDX',
     'CLEU',
     'CLFD',
     'CLGN',
     'CLGX',
     'CLI',
     'CLIR',
     'CLNN',
     'CLOV',
     'CLPS',
     'CLPT',
     'CLRB',
     'CLRO',
     'CLSK',
     'CLSN',
     'CLVT',
     'CLW',
     'CLWT',
     'CLX',
     'CM',
     'CMBM',
     'CMCM',
     'CMCSA',
     'CMCT',
     'CME',
     'CMG',
     'CMI',
     'CMRX',
     'CMS',
     'CNBKA',
     'CNET',
     'CNF',
     'CNFR',
     'CNI',
     'CNNB',
     'CNSL',
     'CNST',
     'CNTG',
     'CNX',
     'CNXC',
     'CNXN',
     'CO',
     'COCP',
     'CODA',
     'CODI',
     'CODX',
     'COE',
     'COFS',
     'COG',
     'COGT',
     'COKE',
     'COLD',
     'COMS',
     'CONE',
     'COO',
     'COR',
     'CORE',
     'CORT',
     'COST',
     'CPAC',
     'CPB',
     'CPHC',
     'CPIX',
     'CPK',
     'CPLP',
     'CPRT',
     'CPSH',
     'CPSI',
     'CPSR',
     'CPSS',
     'CPST',
     'CPT',
     'CPTA',
     'CRDF',
     'CREG',
     'CRESY',
     'CREX',
     'CRHC',
     'CRI',
     'CRIS',
     'CRK',
     'CRSA',
     'CRT',
     'CRTD',
     'CRTO',
     'CRU',
     'CRWD',
     'CRWS',
     'CSCW',
     'CSGP',
     'CSGS',
     'CSII',
     'CSL',
     'CSPI',
     'CSR',
     'CSSE',
     'CSTE',
     'CSTL',
     'CSV',
     'CSWC',
     'CSWI',
     'CTAC',
     'CTAQ',
     'CTG',
     'CTHR',
     'CTIB',
     'CTIC',
     'CTK',
     'CTO',
     'CTRM',
     'CTSO',
     'CTXR',
     'CTXS',
     'CUBE',
     'CUEN',
     'CULP',
     'CURI',
     'CVBF',
     'CVGW',
     'CVLT',
     'CVLY',
     'CVS',
     'CVV',
     'CWBC',
     'CWBR',
     'CWCO',
     'CWEN',
     'CWST',
     'CWT',
     'CXDC',
     'CXDO',
     'CYAN',
     'CYBE',
     'CYD',
     'CYRN',
     'CYRX',
     'CYTH',
     'CZWI',
     'D',
     'DAIO',
     'DAKT',
     'DAO',
     'DARE',
     'DASH',
     'DAVA',
     'DBDR',
     'DBTX',
     'DBVT',
     'DBX',
     'DCT',
     'DCTH',
     'DDMX',
     'DDOG',
     'DEA',
     'DEH',
     'DEI',
     'DEN',
     'DEO',
     'DFFN',
     'DFH',
     'DFHT',
     'DFNS',
     'DFPH',
     'DG',
     'DGICA',
     'DGICB',
     'DGLY',
     'DGNR',
     'DGNS',
     'DGX',
     'DHIL',
     'DHR',
     'DHT',
     'DHX',
     'DIS',
     'DISCA',
     'DISCB',
     'DISCK',
     'DJCO',
     'DL',
     'DLHC',
     'DLNG',
     'DLPN',
     'DLR',
     'DLTR',
     'DM',
     'DMLP',
     'DMS',
     'DMTK',
     'DMYD',
     'DNB',
     'DNK',
     'DNMR',
     'DOC',
     'DOCU',
     'DOGZ',
     'DORM',
     'DOX',
     'DOYU',
     'DPZ',
     'DRD',
     'DRE',
     'DRIO',
     'DRTT',
     'DSAC',
     'DSGX',
     'DSPG',
     'DSSI',
     'DSWL',
     'DTE',
     'DTSS',
     'DUK',
     'DUO',
     'DUOT',
     'DVA',
     'DVD',
     'DWIN',
     'DWSN',
     'DX',
     'DXCM',
     'DXYN',
     'DYAI',
     'DYN',
     'DYNT',
     'EA',
     'EAC',
     'EARS',
     'EAST',
     'EBAY',
     'EBC',
     'EBF',
     'EBMT',
     'EBON',
     'ECL',
     'ECOL',
     'ED',
     'EDAP',
     'EDN',
     'EDRY',
     'EDSA',
     'EDTK',
     'EDTX',
     'EDTXU',
     'EDUC',
     'EEX',
     'EFC',
     'EFOI',
     'EFX',
     'EGO',
     'EGOV',
     'EGRX',
     'EH',
     'EHC',
     'EIG',
     'EIX',
     'EKSO',
     'EL',
     'ELAN',
     'ELDN',
     'ELS',
     'ELSE',
     'ELTK',
     'ELYS',
     'EMCF',
     'EMKR',
     'EMPW',
     'ENB',
     'ENG',
     'ENIA',
     'ENIC',
     'ENLV',
     'ENOB',
     'ENPC',
     'ENTX',
     'ENZ',
     'EOSE',
     'EPC',
     'EPD',
     'EPHY',
     'EPIX',
     'EPSN',
     'EQ',
     'EQC',
     'EQD',
     'EQIX',
     'EQOS',
     'EQR',
     'EQT',
     'ERES',
     'ERIC',
     'ERIE',
     'ERYP',
     'ES',
     'ESBK',
     'ESCA',
     'ESEA',
     'ESGR',
     'ESLT',
     'ESRT',
     'ESS',
     'ESSA',
     'ESSC',
     'ESXB',
     'ETAC',
     'ETH',
     'ETON',
     'ETR',
     'ETRN',
     'ETTX',
     'ETWO',
     'EURN',
     'EVA',
     'EVAX',
     'EVBG',
     'EVC',
     'EVFM',
     'EVGN',
     'EVK',
     'EVLO',
     'EVOK',
     'EVOL',
     'EVRG',
     'EW',
     'EXC',
     'EXFO',
     'EXK',
     'EXPC',
     'EXPD',
     'EXPO',
     'EXR',
     'EYEG',
     'EYEN',
     'EYES',
     'EZPW',
     'FAF',
     'FAII',
     'FAMI',
     'FANH',
     'FAST',
     'FAT',
     'FBMS',
     'FBSS',
     'FCAC',
     'FCAP',
     'FCCO',
     'FCCY',
     'FCFS',
     'FCN',
     'FDP',
     'FDS',
     'FE',
     'FEDU',
     'FEIM',
     'FENC',
     'FENG',
     'FF',
     'FFBW',
     'FFHL',
     'FFIV',
     'FFNW',
     'FGF',
     'FGNA',
     'FHB',
     'FIBK',
     'FIII',
     'FINV',
     'FIVN',
     'FIZZ',
     'FLAC',
     'FLIR',
     'FLMN',
     'FLO',
     'FLUX',
     'FLWS',
     'FLXS',
     'FMAC',
     'FMNB',
     'FMS',
     'FMTX',
     'FMX',
     'FNF',
     'FNHC',
     'FNV',
     'FONR',
     'FORD',
     'FORTY',
     'FOX',
     'FOXA',
     'FOXW',
     'FPAC',
     'FPI',
     'FR',
     'FRAF',
     'FREE',
     'FRG',
     'FRHC',
     'FRLN',
     'FRO',
     'FRPH',
     'FRSX',
     'FRT',
     'FSBW',
     'FSEA',
     'FSFG',
     'FSII',
     'FSK',
     'FSKR',
     'FSM',
     'FSR',
     'FSRV',
     'FSS',
     'FST',
     'FSTX',
     'FSV',
     'FTDR',
     'FTEK',
     'FTFT',
     'FTIV',
     'FTOC',
     'FTS',
     'FUBO',
     'FUNC',
     'FUSB',
     'FUSE',
     'FUSN',
     'FUTU',
     'FUV',
     'FVAM',
     'FVE',
     'FVRR',
     'FWAA',
     'FWP',
     'FWRD',
     'FXNC',
     'GABC',
     'GAIA',
     'GAIN',
     'GASS',
     'GATO',
     'GB',
     'GBDC',
     'GBNY',
     'GCBC',
     'GCMG',
     'GD',
     'GDYN',
     'GECC',
     'GEG',
     'GENC',
     'GENE',
     'GEO',
     'GERN',
     'GEVO',
     'GFED',
     'GFI',
     'GFL',
     'GHC',
     'GHG',
     'GHLD',
     'GHM',
     'GHSI',
     'GIB',
     'GIFI',
     'GIGM',
     'GIK',
     'GIL',
     'GILD',
     'GILT',
     'GIS',
     'GIX',
     'GLAD',
     'GLBZ',
     'GLDD',
     'GLEO',
     'GLG',
     'GLOP',
     'GLP',
     'GLPG',
     'GLRE',
     'GLSI',
     'GLTO',
     'GLUU',
     'GMAB',
     'GMBL',
     'GMDA',
     'GME',
     'GMED',
     'GMRE',
     'GMTX',
     'GNE',
     'GNFT',
     'GNMK',
     'GNOG',
     'GNRS',
     'GNSS',
     'GNTX',
     'GNTY',
     'GNUS',
     'GO',
     'GOAC',
     'GOCO',
     'GOEV',
     'GOLD',
     'GOLF',
     'GOSS',
     'GOVX',
     'GPK',
     'GRA',
     'GRAY',
     'GRCY',
     'GRFS',
     'GRIL',
     'GRIN',
     'GRMN',
     ...]




```python

```
