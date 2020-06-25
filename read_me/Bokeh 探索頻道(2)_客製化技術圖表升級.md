# Bokeh 探索頻道(2)~客製化技術圖表升級

## 改造動機

上次介紹了Bokeh厲害的地方，那可不可以應用到投資圖表的繪製呢？
在Hahow課程：『Python 理財：打造小資族選股策略』的單元20中，韓老師用Talib和matplotlib套件示範了如何提取資料和客製化技術圖表繪圖，點出應用方向，然而課程重點是放在選股，不是在視覺化，圖表功能比較簡單，當時Bokeh也沒那麼厲害。
![](https://i.imgur.com/Q8hEt30.png)

有些同學希望能有更精美的圖表可使用，搭配課程精彩的選股程式技巧，不就更完美了？這個願望今天就可以實現，後面有銜接課程的程式碼直接提供使用。
現在有Bokeh的幫忙，用Python也能打造不錯的互動圖表效果，接下來就要綜合小資族課程、Bokeh、網路上各路技巧，來打造適用於Hahow課程的新圖表。沒參與課程的可以用github 連結裡的demo_json檔來試試，只要資料格式對應，都可使用。

#### github:https://github.com/benbilly3/bokeh_explore
使用的程式檔為technical_chart。


## 繪圖技巧說明

寫Python就要利用其他魔法師的咒語，省時有效率，先來Bokeh gallery看看沒有範例。
結果有現成的Ｋ線圖範例，太好了！

![](https://i.imgur.com/wjTomBS.png)

雖然仍是陽春的靜態圖，但至少有了改造藍圖，從官網的程式碼：
https://docs.bokeh.org/en/latest/docs/gallery/candlestick.html

可以發現bokeh輕鬆地用pandas分類紅黑棒資料，再用segment畫引線和用vbar畫長棒圖。眼尖的人可以發現股價碰到假日的時候會有空值，這造成閱讀上有些礙眼，能否有讓時間序列日期資料的解決辦法呢？

在還不熟bokeh的時候，stackoverflow也是我們好朋友．．．處理關鍵在讓日期轉為文字，不用會自動補假日日期的datetime。

```
fig.xaxis.major_label_overrides = {
            i: date.strftime('%Y/%m/%d') for i, date in enumerate(pd.to_datetime(df["date"]))
            # pd.date_range(start='3/1/2000', end='1/08/2018')
        }
```

bokeh是物件導向繪圖庫，都封裝相當好，基本上沒啥程式技巧，就像玩樂高積木一樣，搜尋工具來堆，不難，比較繁瑣，內容頗多。
接著主要會實踐下列功能到圖表，有興趣學的可以看連結。

#### 1. figure圖紙設定，bokeh各種models應用
https://docs.bokeh.org/en/latest/docs/reference/plotting.html

#### 2. 處理假日日期造成的資料不連續問題，x_range overwrite技巧
https://stackoverflow.com/questions/23585545/how-do-i-make-bokeh-omit-missing-dates-when-using-datetime-as-x-axis

#### 3. hover互動資料顯示
https://docs.bokeh.org/en/latest/docs/user_guide/tools.html

#### 4. legend物件控制，從label控制線圖開關。將legend移到圖表外，讓版面清晰。
https://stackoverflow.com/questions/26254619/position-of-the-legend-in-a-bokeh-plot
https://docs.bokeh.org/en/latest/docs/user_guide/interaction/legends.html

#### 5. 位移、縮放、十字線、重置、存檔工具

https://docs.bokeh.org/en/latest/docs/user_guide/tools.html

#### 6. second y_ranges繪製(使用雙軸)
https://docs.bokeh.org/en/latest/docs/user_guide/plotting.html#userguide-plotting-twin-axes

#### 7. 利用vbar和segment快速繪製k線,並將均線帶入。
https://docs.bokeh.org/en/latest/docs/gallery/candlestick.html

#### 8. 建立技術線子圖組
https://docs.bokeh.org/en/latest/docs/user_guide/layout.html#userguide-layout

#### 經過神奇的魔法就有hahow課程的進化版，包含以上的功能，實踐了不錯的效果！加入output_file('檔名')就可產生html檔在瀏覽器使用，只要再寫一個傳導查詢變數的API，就能串接做一個服務出來，之後會講用FastApi寫後端來串Bokeh。

![](https://i.imgur.com/4BY8lBt.png)


## 課程相容資料提取工具

#### 從python小資族的sqlite提取資料
若DB為pickle檔，須將pd.read_sql那行修改為read_pickle，並要注意格式。

如果有上過韓老師python小資族，可將course11.ipynb上頭用read_sql取資料的程式改為下部份取資料，輸入股號和資料開始日期，產出的data丟入後面的繪圖程式，即可產生互動式圖表。
```
import pandas as pd
import sqlite3
import os
import json
from talib import abstract


def get_price_data(stock_id,date):
    # connect to sql
    conn = sqlite3.connect(os.path.join('data', "data.db"))
    # read data from sql
    df = pd.read_sql(f"select stock_id,證券名稱, date, 開盤價, 收盤價, 最高價, 最低價, 成交股數 from price where stock_id='{stock_id}' and date > '{date}'", conn,
        index_col=['date'])
    # rename the columns of dataframe
    df.index=df.index.astype(str).str.split(" ").str[0]
    df.rename(columns={'收盤價':'close','證券名稱':'name', '開盤價':'open', '最高價':'high', '最低價':'low', '成交股數':'volume'}, inplace=True)
    df['MA5']=df['close'].rolling(5).mean()
    df['MA10']=df['close'].rolling(10).mean()
    df['MA20']=df['close'].rolling(20).mean()
    df['MA60']=df['close'].rolling(60).mean()
    df['MA120']=df['close'].rolling(120).mean()
    df['volume']=df['volume']/1000

    
    RSI = pd.DataFrame(abstract.RSI(df, timeperiod=12),columns=['RSI_12'])
    RSI['RSI_36']=abstract.RSI(df, timeperiod=36)
    RSI=RSI.to_dict()
    STOCH = abstract.STOCH(df).to_dict()
    MACD=abstract.MACD(df).to_dict()
    basic=df.iloc[-1,:2].to_dict()
    df=df.drop(columns=['stock_id','name']).to_dict()
    data={'basic':basic,'price_df':df,'RSI':RSI,'STOCH' :STOCH,'MACD':MACD }
    
    return data

```
## 程式碼下載

使用的繪圖程式檔為technical_chart，將get_price_data(stock_id,date)帶入technical_chart(json_df)即可繪圖，可到gitlab下載bokeh_explore。
#### github:https://github.com/benbilly3/bokeh_explore

{%gist benbilly3/ac18625f06545cff7ab46256ceaccbbc %}

###### tags: `bokeh`