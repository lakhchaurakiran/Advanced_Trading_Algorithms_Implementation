## Accrual Anomaly

Accrual is the component of the income which is not received in cash. Therefore, there is a risk of the cash not being realized. Businesses can use this to artificially sore up their earnings. Stocks where earnings are dominated by accruals are expected to fall (potential losers) relative to the stocks where earnings are dominated by cash (potential winners). 

## Accrual Anomaly Trading Srategy

Based on the paper titled "Do stock prices fully reflect information in accruals and cash flows about future earnings" by Richard G. Sloan in 1996.\
The idea behind accrual anomaly trading strategy is that in the short-term markets fail to distinguish between earnings driven by accruals and earnings driven by cash, but after some time correct for the anomaly.

### Calculating Accruals

Accruals  = ($\Delta$CA - $\Delta$Cash) - ($\Delta$CL - $\Delta$STD - $\Delta$TP) - Dep.

where,\
$\Delta$CA = Change in Current Assets\
$\Delta$Cash = Change in Cash and Cash Equivalents\
$\Delta$CL = Change in Current Liabilities\
$\Delta$STD = Change in Short-Term Debt\
$\Delta$TP = Change in Tax Payable\
Dep. = Depreciation expense

Income = Income From Continuing Operations / Average Total Assets

Accrual Component of Income = Accruals / Average Total Assets\
Cash Component of Income = Income - Accrual Component

**Steps of trading strategy:**
1. Compute the accrual (or cash) components for each company.
2. Sort the companies based on accrual (or cash) components
3. Divide the companies into deciles based on the sorted accrual (or cash) components.
4. Long the low-accrual component companies and short the high-accrual component comapnies (or long the high-cash component companies and short the low-cash component companies)

### Importing the necessary modules


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

### Getting the list of stocks from NASDAQ


```python
# getting a list of all the stocks from NASDAQ with Market Cap > 0

df = pd.read_csv('nasdaq_screener.csv')
df = df[df['Market Cap'] > 0]
df.reset_index(inplace=True, drop=True)
stocks = list(df['Symbol'])
len(stocks)
```




    5743



### Functions for web-scraping financial statement tables


```python
# functions for web-scraping financial statement tables from marketwatch.com


def get_table_simple(table,is_table_tag=True):
    '''
    This function will use an html table element and will return 
    a list of lists representing the table
    
    inputs:
    table : an html element
    is_table_tag :  True or False (whether the table is an actual html table 
    element or a simple rows and columns separated by div elements.
    
    output : returns the table in a list of lists form
    '''
    elems = table.find_all('tr') if is_table_tag else get_children(table)
    table_data = list()
    for row in elems:
        row_data = list()
        row_elems = get_children(row)
        for elem in row_elems:
            text = elem.text.strip().replace("\n","")
            text = remove_multiple_spaces(text)
            if len(text)==0:
                continue
            row_data.append(text)
        table_data.append(row_data)
    return table_data

def get_children(html_content):
    return [item for item in html_content.children if type(item)==bs4.element.Tag or len(str(item).replace("\n","").strip())>0]

def remove_multiple_spaces(string):
    if type(string)==str:
        return ' '.join(string.split())
    return string



# function for reading balance sheet data from marketwatch.com using a stock ticker
def get_balance_sheet_data(ticker,yr):
    '''
    The function returns a dictionary of important financial measures for a particular year 
    obtained from the balance sheet of the firm with stock ticker 'ticker'
    
    inputs:
    ticker: stock ticker of the firm
    yr: The year in string notation (e.g. '2020')
    
    output:
    returns a dictionary of the important financial measures 
    (viz. total assets, total current assets, total liabilities, total current liabilities, 
    long-term debt and total common equity)
    '''
    
    urlbalancesheet = 'https://www.marketwatch.com/investing/stock/'+ticker+'/financials/balance-sheet'
    text_soup_balancesheet = BeautifulSoup(requests.get(urlbalancesheet).text,"lxml") #read in
    tables_balancesheet = text_soup_balancesheet.findAll('div', {'class': 'financials'})
    
    bs_assets_table = get_table_simple(tables_balancesheet[0],is_table_tag=True)
    bs_assets_table[0].remove('5-year trend')
    df_assets = pd.DataFrame.from_records(bs_assets_table[1:],columns=bs_assets_table[0])
    df_assets.rename(columns={'ItemItem':'Item'}, inplace=True)
    df_assets['Item'] = df_assets['Item'].astype(str).apply(lambda x: x[:len(x)//2])
    
    bs_liabilities_table = get_table_simple(tables_balancesheet[1],is_table_tag=True)
    bs_liabilities_table[0].remove('5-year trend')
    df_liabilities = pd.DataFrame.from_records(bs_liabilities_table[1:],columns=bs_liabilities_table[0])
    df_liabilities.rename(columns={'ItemItem':'Item'}, inplace=True)
    df_liabilities['Item'] = df_liabilities['Item'].astype(str).apply(lambda x: x[:len(x)//2])
    
    if yr not in df_assets.columns or yr not in df_liabilities.columns:
        return {'CA': None, 'Cash': None, 'CL': None, 'STD': None, 'TP': None, 'TA': None}
    
    TA = text_parse(df_assets[df_assets.Item=='Total Assets'][yr].values[0])
    CA = text_parse(df_assets[df_assets.Item=='Total Current Assets'][yr].values[0])
    Cash = text_parse(df_assets[df_assets.Item=='Cash & Short Term Investments'][yr].values[0])
    
    STD = text_parse(df_liabilities[df_liabilities.Item == 'ST Debt & Current Portion LT Debt'][yr].values[0])
    CL = text_parse(df_liabilities[df_liabilities.Item == 'Total Current Liabilities'][yr].values[0])
    TP = text_parse(df_liabilities[df_liabilities.Item == 'Income Tax Payable'][yr].values[0])
    
    return {'CA': CA, 'Cash': Cash, 'CL': CL, 'STD': STD, 'TP': TP, 'TA': TA}




# function for reading income statement data from marketwatch.com using a stock ticker
def get_income_statement_data(ticker,yr):  
    '''
    The function returns a dictionary of important financial measures for a particular year 
    obtained from the income statement of the firm with stock ticker 'ticker'
    
    inputs:
    ticker: stock ticker of the firm
    yr: The year in string notation (e.g. '2020')
    
    output:
    returns a dictionary of the important financial measures 
    (viz. Net Income, Gross Profit and Total Revenue)
    '''
    
    url_financials = 'https://www.marketwatch.com/investing/stock/'+ticker+'/financials'
    text_soup_financials = BeautifulSoup(requests.get(url_financials).text,"lxml") #read in
    tables_incomestatement = text_soup_financials.findAll('div', {'class': 'financials'})
    
    is_table = get_table_simple(tables_incomestatement[0],is_table_tag=True)
    is_table[0].remove('5-year trend')
    df_is = pd.DataFrame.from_records(is_table[1:],columns=is_table[0])
    df_is.rename(columns={'ItemItem':'Item'}, inplace=True)
    df_is['Item'] = df_is['Item'].astype(str).apply(lambda x: x[:len(x)//2])
    
    if yr not in df_is.columns:
        return {'Income_From_Continuing_Operations': None}
    
    Income_From_Continuing_Operations = text_parse(df_is[df_is['Item']=='Net Income'][yr].values[0])
    
    return {'Income_From_Continuing_Operations': Income_From_Continuing_Operations}




# function for reading cash flow statement data from marketwatch.com using a stock ticker
def get_cash_flow_data(ticker,yr):  
    '''
    The function returns a dictionary of important financial measures for a particular year 
    obtained from the cash flow statement of the firm with stock ticker 'ticker'
    
    inputs:
    ticker: stock ticker of the firm
    yr: The year in string notation (e.g. '2020')
    
    output:
    returns a dictionary of the important financial measures 
    (viz. total cash from operating activities)
    '''
        
    urlcashflow = 'https://www.marketwatch.com/investing/stock/'+ticker+'/financials/cash-flow'
    text_soup_cashflow = BeautifulSoup(requests.get(urlcashflow).text,"lxml") #read in
    tables_cashflow = text_soup_cashflow.findAll('div', {'class': 'financials'})
    cf_table = get_table_simple(tables_cashflow[0],is_table_tag=True)
    
    cf_table[0].remove('5-year trend')
    df_cf = pd.DataFrame.from_records(cf_table[1:],columns=cf_table[0])
    df_cf.rename(columns={'ItemItem':'Item'}, inplace=True)
    df_cf['Item'] = df_cf['Item'].astype(str).apply(lambda x: x[:len(x)//2])
    
    if yr not in df_cf.columns:
        return {'Dep': None}

    Dep = text_parse(df_cf[df_cf['Item']=='Depreciation, Depletion & Amortization'][yr].values[0])
    
    return {'Dep': Dep}

                      
                      



def text_parse(text):
    '''
    This function to convert the string outputs of the 
    financial statements to float values  
    '''
    if len(text) == 2:
        text = text[0]
    text = text.strip('(').strip(')').strip('$').strip(' ')
    if text == '-' or text == '':
        return 0    
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

### Calculating Accrual and cash components of income for all the stocks of the sample


```python
columns = ['symbol', 'TA_cur_year', 'TA_last_year', 'Avg_TA', 'CA_cur_year', 'CA_last_year', 'Delta_CA', 'Cash_cur_year', 'Cash_last_year', 'Delta_Cash', 'CL_cur_year' , 'CL_last_year', 'Delta_CL', 'STD_cur_year', 'STD_last_year', 'Delta_STD', 'TP_cur_year', 'TP_last_year', 'Delta_TP', 'Dep', 'Accruals', 'Income_From_Continuing_Operations', 'Income', 'Accrual_Component','Cash_Component']
Accrual_Cash_Component_df = pd.DataFrame(columns=columns)

for symbol in stocks[4015:]:
    print(symbol)
    
    try:
        # Balance Sheet Data
        bs_data_cur_year = get_balance_sheet_data(symbol,'2020')
        bs_data_last_year = get_balance_sheet_data(symbol,'2019')

        # Income Statement Data
        is_data_cur_year = get_income_statement_data(symbol,'2020')
        is_data_last_year =  get_income_statement_data(symbol,'2019')

        # Cash Flow Data
        cf_data_cur_year = get_cash_flow_data(symbol,'2020')
        cf_data_last_year = get_cash_flow_data(symbol,'2019')
        

        TA_cur_year = bs_data_cur_year['TA']
        TA_last_year = bs_data_last_year['TA']

        CA_cur_year = bs_data_cur_year['CA']
        CA_last_year = bs_data_last_year['CA']
        
        Cash_cur_year = bs_data_cur_year['Cash']
        Cash_last_year = bs_data_last_year['Cash']

        CL_cur_year = bs_data_cur_year['CL']
        CL_last_year = bs_data_last_year['CL']
        
        STD_cur_year = bs_data_cur_year['STD']
        STD_last_year = bs_data_last_year['STD']

        TP_cur_year = bs_data_cur_year['TP']
        TP_last_year = bs_data_last_year['TP']

        Dep = cf_data_cur_year['Dep']
        
        Income_From_Continuing_Operations = is_data_cur_year['Income_From_Continuing_Operations']
        
        financials = [TA_cur_year, TA_last_year, CA_cur_year, CA_last_year, Cash_cur_year, Cash_last_year, CL_cur_year, CL_last_year, STD_cur_year, STD_last_year, TP_cur_year, TP_last_year, Dep, Income_From_Continuing_Operations]

        if None in financials:
            continue

        else:
            Avg_TA = (TA_cur_year + TA_last_year)/2
            Delta_CA = TA_cur_year - TA_last_year
            Delta_Cash = Cash_cur_year - Cash_last_year
            Delta_CL = CL_cur_year - CL_last_year
            Delta_STD = STD_cur_year - STD_last_year
            Delta_TP = TP_cur_year - TP_last_year

            Accruals = (Delta_CA - Delta_Cash) -(Delta_CL - Delta_STD - Delta_TP) - Dep
            Income = Income_From_Continuing_Operations / Avg_TA
            Accrual_Component = Accruals / Avg_TA
            Cash_Component = Income - Accrual_Component


            Accrual_Cash_Component_df = Accrual_Cash_Component_df.append(pd.DataFrame([[symbol, TA_cur_year, TA_last_year, Avg_TA, CA_cur_year, CA_last_year, Delta_CA, Cash_cur_year, Cash_last_year, Delta_Cash, CL_cur_year, CL_last_year, Delta_CL, STD_cur_year, STD_last_year, Delta_STD, TP_cur_year, TP_last_year, Delta_TP, Dep, Accruals, Income_From_Continuing_Operations, Income, Accrual_Component, Cash_Component]],columns=columns),ignore_index=True)
    except IndexError:
        pass
```

### Results


```python
Accrual_Cash_Component_df
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
      <th>TA_cur_year</th>
      <th>TA_last_year</th>
      <th>Avg_TA</th>
      <th>CA_cur_year</th>
      <th>CA_last_year</th>
      <th>Delta_CA</th>
      <th>Cash_cur_year</th>
      <th>Cash_last_year</th>
      <th>Delta_Cash</th>
      <th>...</th>
      <th>Delta_STD</th>
      <th>TP_cur_year</th>
      <th>TP_last_year</th>
      <th>Delta_TP</th>
      <th>Dep</th>
      <th>Accruals</th>
      <th>Income_From_Continuing_Operations</th>
      <th>Income</th>
      <th>Accrual_Component</th>
      <th>Cash_Component</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>A</td>
      <td>9.630000e+09</td>
      <td>9.450000e+09</td>
      <td>9.540000e+09</td>
      <td>3.420000e+09</td>
      <td>3.190000e+09</td>
      <td>1.800000e+08</td>
      <td>1.440000e+09</td>
      <td>1.380000e+09</td>
      <td>6.000000e+07</td>
      <td>...</td>
      <td>-490000000.0</td>
      <td>63000000.0</td>
      <td>292000000.0</td>
      <td>-229000000.0</td>
      <td>3.080000e+08</td>
      <td>-2.970000e+08</td>
      <td>7.190000e+08</td>
      <td>0.075367</td>
      <td>-0.031132</td>
      <td>0.106499</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AA</td>
      <td>1.486000e+10</td>
      <td>1.463000e+10</td>
      <td>1.474500e+10</td>
      <td>4.520000e+09</td>
      <td>3.530000e+09</td>
      <td>2.300000e+08</td>
      <td>1.630000e+09</td>
      <td>9.420000e+08</td>
      <td>6.880000e+08</td>
      <td>...</td>
      <td>0.0</td>
      <td>91000000.0</td>
      <td>104000000.0</td>
      <td>-13000000.0</td>
      <td>6.530000e+08</td>
      <td>-1.324000e+09</td>
      <td>1.700000e+08</td>
      <td>0.011529</td>
      <td>-0.089793</td>
      <td>0.101322</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AAL</td>
      <td>6.201000e+10</td>
      <td>6.000000e+10</td>
      <td>6.100500e+10</td>
      <td>1.110000e+10</td>
      <td>8.210000e+09</td>
      <td>2.010000e+09</td>
      <td>7.470000e+09</td>
      <td>3.980000e+09</td>
      <td>3.490000e+09</td>
      <td>...</td>
      <td>-120000000.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2.370000e+09</td>
      <td>-2.230000e+09</td>
      <td>8.890000e+09</td>
      <td>0.145726</td>
      <td>-0.036554</td>
      <td>0.182280</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AAOI</td>
      <td>4.808100e+08</td>
      <td>4.668300e+08</td>
      <td>4.738200e+08</td>
      <td>2.091700e+08</td>
      <td>1.928000e+08</td>
      <td>1.398000e+07</td>
      <td>5.011000e+07</td>
      <td>6.703000e+07</td>
      <td>-1.692000e+07</td>
      <td>...</td>
      <td>14500000.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2.473000e+07</td>
      <td>8.880000e+06</td>
      <td>5.845000e+07</td>
      <td>0.123359</td>
      <td>0.018741</td>
      <td>0.104618</td>
    </tr>
    <tr>
      <th>4</th>
      <td>AAON</td>
      <td>4.614400e+08</td>
      <td>3.839400e+08</td>
      <td>4.226900e+08</td>
      <td>2.202500e+08</td>
      <td>1.875500e+08</td>
      <td>7.750000e+07</td>
      <td>8.229000e+07</td>
      <td>4.437000e+07</td>
      <td>3.792000e+07</td>
      <td>...</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>2.563000e+07</td>
      <td>1.095000e+07</td>
      <td>7.901000e+07</td>
      <td>0.186922</td>
      <td>0.025906</td>
      <td>0.161016</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>2551</th>
      <td>ZUMZ</td>
      <td>9.142600e+08</td>
      <td>5.341900e+08</td>
      <td>7.242250e+08</td>
      <td>4.125900e+08</td>
      <td>3.271800e+08</td>
      <td>3.800700e+08</td>
      <td>2.512000e+08</td>
      <td>1.653300e+08</td>
      <td>8.587000e+07</td>
      <td>...</td>
      <td>61800000.0</td>
      <td>4690000.0</td>
      <td>5820000.0</td>
      <td>-1130000.0</td>
      <td>8.367000e+07</td>
      <td>2.046000e+08</td>
      <td>6.688000e+07</td>
      <td>0.092347</td>
      <td>0.282509</td>
      <td>-0.190162</td>
    </tr>
    <tr>
      <th>2552</th>
      <td>ZUO</td>
      <td>4.022300e+08</td>
      <td>3.260500e+08</td>
      <td>3.641400e+08</td>
      <td>2.572000e+08</td>
      <td>2.491400e+08</td>
      <td>7.618000e+07</td>
      <td>1.719400e+08</td>
      <td>1.762500e+08</td>
      <td>-4.310000e+06</td>
      <td>...</td>
      <td>7230000.0</td>
      <td>432000.0</td>
      <td>1650000.0</td>
      <td>-1218000.0</td>
      <td>2.045000e+07</td>
      <td>2.850200e+07</td>
      <td>8.339000e+07</td>
      <td>0.229005</td>
      <td>0.078272</td>
      <td>0.150733</td>
    </tr>
    <tr>
      <th>2553</th>
      <td>ZVO</td>
      <td>1.613100e+08</td>
      <td>2.501400e+08</td>
      <td>2.057250e+08</td>
      <td>7.683000e+07</td>
      <td>1.505100e+08</td>
      <td>-8.883000e+07</td>
      <td>5.701000e+07</td>
      <td>9.504000e+07</td>
      <td>-3.803000e+07</td>
      <td>...</td>
      <td>-950000.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.140000e+07</td>
      <td>-1.049000e+07</td>
      <td>4.895000e+07</td>
      <td>0.237939</td>
      <td>-0.050990</td>
      <td>0.288929</td>
    </tr>
    <tr>
      <th>2554</th>
      <td>ZYME</td>
      <td>5.383800e+08</td>
      <td>3.682100e+08</td>
      <td>4.532950e+08</td>
      <td>4.550800e+08</td>
      <td>3.118300e+08</td>
      <td>1.701700e+08</td>
      <td>4.263500e+08</td>
      <td>2.989000e+08</td>
      <td>1.274500e+08</td>
      <td>...</td>
      <td>1440000.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.028000e+07</td>
      <td>3.076000e+07</td>
      <td>1.805500e+08</td>
      <td>0.398306</td>
      <td>0.067859</td>
      <td>0.330447</td>
    </tr>
    <tr>
      <th>2555</th>
      <td>ZYXI</td>
      <td>7.218000e+07</td>
      <td>2.828000e+07</td>
      <td>5.023000e+07</td>
      <td>6.302000e+07</td>
      <td>2.257000e+07</td>
      <td>4.390000e+07</td>
      <td>3.917000e+07</td>
      <td>1.404000e+07</td>
      <td>2.513000e+07</td>
      <td>...</td>
      <td>870000.0</td>
      <td>280000.0</td>
      <td>52000.0</td>
      <td>228000.0</td>
      <td>1.570000e+06</td>
      <td>1.337800e+07</td>
      <td>9.070000e+06</td>
      <td>0.180569</td>
      <td>0.266335</td>
      <td>-0.085765</td>
    </tr>
  </tbody>
</table>
<p>2556 rows × 25 columns</p>
</div>



So we get accrual and cash components for 2566 firms. We will now divide the firms into deciles based on the Accrual component values. One can also use the cash component here, taking into account the fact that the potential winners will be the ones with the lowest accrual components (highest cash components) and potential losers will be the ones with the highest accrual components (lowest cash components).

### Distribution of Accrual components


```python
fig, ax = plt.subplots(figsize=(12,8))
Accrual_Cash_Component_df['Accrual_Component'].plot.hist(bins=50,log=True,ax=ax)
ax.set_xlabel('Accrual Componets',fontsize=20)
ax.set_ylabel('Frequency',fontsize=20)
```




    Text(0, 0.5, 'Frequency')




    
![png](Accrual_Anomaly_Trading_Strategy_Implementation_files/Accrual_Anomaly_Trading_Strategy_Implementation_14_1.png)
    


### Dividing the firms into deciles based on their Accrual Components


```python
deciles = [np.quantile(Accrual_Cash_Component_df['Accrual_Component'].values,x) for x in np.arange(0,11)/10]
```


```python
Accrual_Cash_Component_df['Accrual_Component_decile'] = 0
for i in range(len(deciles)-1):
    Accrual_Cash_Component_df['Accrual_Component_decile'][Accrual_Cash_Component_df['Accrual_Component'].apply(lambda x: x>=deciles[i] and x<deciles[i+1]).values] = i
```


```python
Accrual_Cash_Component_df['Accrual_Component_decile']
```


```python
fig, ax = plt.subplots(figsize=(12,8))
sns.countplot(ax=ax,x='Accrual_Component_decile',data=Accrual_Cash_Component_df)
ax.set_xlabel('Accrual Component Deciles',fontsize=20)
ax.set_ylabel('Count',fontsize=20)
```




    Text(0, 0.5, 'Count')




    
![png](Accrual_Anomaly_Trading_Strategy_Implementation_files/Accrual_Anomaly_Trading_Strategy_Implementation_19_1.png)
    


### Low-Accrual stocks (Potential Winners)
These are the stocks that belong to the bottommost decile of Accrual components.


```python
Accrual_Cash_Component_df['symbol'][Accrual_Cash_Component_df['Accrual_Component_decile'] == 0]
```




    22       ACB
    25      ACER
    26      ACHC
    60       AEY
    70      AGYS
            ... 
    2523    YELL
    2524    YELP
    2525    YETI
    2528    YNDX
    2536    ZDGE
    Name: symbol, Length: 257, dtype: object



### High-Accrual stocks (Potential Losers)
These are the stocks that belong to the topmost decile of Accrual components.


```python
Accrual_Cash_Component_df['symbol'][Accrual_Cash_Component_df['Accrual_Component_decile'] == 9]
```




    9       ABBV
    11      ABCM
    13       ABG
    30      ACMR
    46      ADUS
            ... 
    2534       Z
    2539      ZG
    2547    ZNGA
    2551    ZUMZ
    2555    ZYXI
    Name: symbol, Length: 255, dtype: object




```python

```
