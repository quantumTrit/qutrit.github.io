---
layout: post
title:  "Stock Price Prediction with LSTM based on the hist data only"
date:   2021-05-30 00:00:00 +0300
categories: jekyll update
---

**The problem set:** In this example I was trying to test if the stock price can be predicted only out of its historical data. The market I considered is MICEX, the stock is Gazprom (blue chip). The task is to:

1. predict the price for the end of the next day (close price)

2. predict upside / downside movement for the end of the next day

To check the time of the cells execution in auto mode I use *ipython-autotime*.

<pre><code>
!pip install ipython-autotime
%load_ext autotime
</code></pre>

<pre><code>
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

from datetime import datetime, timedelta

from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import mean_squared_error

from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense, Dropout, BatchNormalization, Conv1D, Flatten
from tensorflow.keras.layers import SpatialDropout1D, MaxPooling1D, GlobalAveragePooling1D, GlobalMaxPooling1D
from tensorflow.keras.layers import LSTM, Bidirectional

inputFolder = 'd/My Drive/01_03_nnMicex/input/'

!ls 'd/My Drive/01_03_nnMicex/input'
</code></pre>

<pre><code>
GAZP_130501_190614.csv	GAZP_190617_191023.csv
time: 140 ms (started: 2021-05-30 18:01:57 +00:00)
</code></pre>

## Data preparation

As input we have train and validation csv files with 6 years of historical data (min, max, open, close, volume) and 4 months to test the model. The step is one day.

As the first steps we convert the date to *datetime* stamp and drop non-informative columns:

<pre><code>
# converting to datetime
dfTrain['dtime'] = pd.to_datetime(dfTrain['&lt;DATE>'], format='%Y%m%d')
dfTest['dtime'] = pd.to_datetime(dfTest['&lt;DATE>'], format='%Y%m%d')
# setting dime index
dfTrain['dtime'] = pd.to_datetime(dfTrain['&lt;DATE>'], format='%Y%m%d')
dfTest['dtime'] = pd.to_datetime(dfTest['&lt;DATE>'], format='%Y%m%d')
# dropping non-feature columns
dfTrain = dfTrain.drop(columns=['&lt;TICKER>', '&lt;PER>', '&lt;DATE>', '&lt;TIME>'], axis=1)
dfTest = dfTest.drop(columns=['&lt;TICKER>', '&lt;PER>', '&lt;DATE>', '&lt;TIME>'], axis=1)
# setting index to datetime
dfTrain = dfTrain.set_index('dtime')
dfTest = dfTest.set_index('dtime')

def consistCheck(df):
    print(f'nulls: {sum([df[col].isnull().sum() for col in df.columns])}')
    print(f'unique dates vs all values: {df.index.unique().shape[0]} / {df.shape[0]}')

print('train sample:')
consistCheck(dfTrain)
print('\ntest sample:')
consistCheck(dfTest)
</code></pre>

<pre><code>
train sample:
nulls: 0
unique dates vs all values: 1540 / 1540

test sample:
nulls: 0
unique dates vs all values: 93 / 93
time: 8.78 ms (started: 2021-05-30 18:01:58 +00:00)
</code></pre>

Having confirmed that the input is correct let's plot it:

<pre><code>
def plotPrice(df, beg, end):
    '''plotting eithor the whole or
    the defined period'''
    for col in df.columns[:-1]:
        df[col][beg:end].plot(
            style='-',
            figsize=(13,7),
            title=f'{beg} : {end}'
        )

periods = [dfTrain.index[0], '2014-05-02', '2015-05-02', '2016-05-02',
           '2017-05-02', '2018-05-02', dfTrain.index[-1]]

for i in range(len(periods)-1):
    plotPrice(dfTrain, periods[i], periods[i+1])
    plt.show()
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-1.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-2.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-3.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-4.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-5.png">

Now let's extract the extra features (year, day of a week, month, etc) from the date itself:

<pre><code>
def dateFeat(df):
    df['year'] = df.index.year
    df['quarter'] = df.index.quarter
    df['month'] = df.index.month
    df['dayOfYear'] = df.index.dayofyear
    df['dayOfMonth'] = df.index.day
    df['weekOfYear'] = df.index.isocalendar().week
    df['dayOfWeek'] = df.index.dayofweek

    return df

dfTrain = dateFeat(dfTrain)
dfTest = dateFeat(dfTest)
</code></pre>

We try to generate some more features by shifting close date for **h** days and taking first difference with the original series.

<pre><code>
def shiftAdd(df, col, h):
    '''
    shifting series of a column
    to h periods ahead & taking 1st diff
    '''
    colDf = pd.DataFrame(df[clm], index=df.index)
    for i in range(1, h+1):
        colDf[f't{i}'] = 0.0 # init cols for the shifted values
        colDf.iloc[i:, i] = colDf.iloc[:-i, 0].values # shift values

    # adding diff-s
    colDf['diff1'] = 0.0
    for i in range(colDf.shape[0]-1):
        colDf['diff1'][i] = colDf.iloc[:-1,0][i] - colDf.iloc[1:,0][i]

    # merge all the new features
    newDf = pd.merge(
        colDf.reset_index(),
        df.reset_index(),
        how='left'
    ).set_index(colDf.index)
    newDf = newDf.drop(columns=['dtime'], axis=1)
    
    print(newDf.shape)

    return newDf

dfTrain = shiftAdd(dfTrain, '&lt;CLOSE>', 5)
dfTest = shiftAdd(dfTest, '&lt;CLOSE>', 5)
</code></pre>

Now we are ready to split our train data for the train and validation samples. Let's take one last year of the train data for the validation sample.

<pre><code>
def splitDf(df, split_date):
    return df.loc[df.index <= split_date].copy(), \
           df.loc[df.index >  split_date].copy()

trainSamp, valSamp = splitDf(dfTrain, '2018-06-14')
</code></pre>

We can also plot the samples for a visual check.

<pre><code>
clmsToVis = ['&lt;CLOSE>', '&lt;OPEN>', '&lt;HIGH>', '&lt;LOW>', '&lt;VOL>']
for clm in clmsToVis:
    plt.figure(figsize=(15,7))
    plt.plot(trainSamp.index, trainSamp[clm], label='Train')
    plt.plot(valSamp.index, valSamp[clm], label='Validation')
    plt.title(clm, color='r')
    plt.show()
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-6.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-7.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-8.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-9.png">
<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-10.png">

After that we are ready to state the target (close price) and scale the sample values that are passed to the neural net.

<pre><code>
# extract target
yTrain = trainSamp['&lt;CLOSE>'].values
xTrain = trainSamp.drop(columns='&lt;CLOSE>')
yVal = valSamp['&lt;CLOSE>'].values
xVal = valSamp.drop(columns='&lt;CLOSE>')

# scale
scaler = MinMaxScaler(feature_range=(0,1))
xTrain = scaler.fit_transform(xTrain)
xVal = scaler.fit_transform(xVal)

# prep the samples fo the input to the recurrent nn 
xTrainLstm = xTrain.reshape(xTrain.shape[0], 1, xTrain.shape[1])
xValLstm = xVal.reshape(xVal.shape[0], 1, xVal.shape[1])
</code></pre>

## Fit the model

To predict the price we can use different models structures. Let's try and validate the score.

The first one will be [Long Short Term Memory](https://en.wikipedia.org/wiki/Long_short-term_memory) RNN with *mean squarred error* as the loss function and metrics. We have 150 neurons LSTM layer and 1 neuron dense output layer.

<pre><code>
model = Sequential()

model.add(LSTM(150, activation='relu'))
model.add(Dense(1))
model.compile(optimizer='adam', loss='mse', metrics=['mse'])

history = model.fit(
    xTrainLstm, yTrain,
    epochs=10,
    validation_data=(xValLstm, yVal),
    shuffle=False,
)

plt.plot(history.history['mse'], label='error @train')
plt.plot(history.history['val_mse'], label='error @val')
plt.xlabel('epoch')
plt.ylabel('error %')
plt.legend()
plt.show()

print(f"the validation score: {history.history['val_mse'][-1]}")
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-11.png">

Now we add *BatchNormalization* and *Dropout* layers.

Batch normalization applies a transformation that maintains the mean output close to 0 and the output standard deviation close to 1.

Dropout randomly zeros some of the incoming layers.

<pre><code>
model = Sequential()

model.add(LSTM(150, activation='relu'))
model.add(BatchNormalization())
model.add(Dropout(0.2))
model.add(Dense(100, activation='relu'))
model.add(Dense(1))
model.compile(optimizer='adam', loss='mse', metrics=['mse'])

history = model.fit(
    xTrainLstm, yTrain,
    epochs=10,
    validation_data=(xValLstm, yVal),
    shuffle=False,
)

plt.plot(history.history['mse'], label='error @train')
plt.plot(history.history['val_mse'], label='error @val')
plt.xlabel('epoch')
plt.ylabel('error %')
plt.legend()
plt.show()

print(f"the validation score: {history.history['val_mse'][-1]}")
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-12.png">

The result is a  much better score. So let's use this model.

<pre><code>
print(model.summary())
</code></pre>

<pre><code>
Model: "sequential_19"
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
lstm_22 (LSTM)               (None, 150)               100800    
_________________________________________________________________
batch_normalization_13 (Batc (None, 150)               600       
_________________________________________________________________
dropout_13 (Dropout)         (None, 150)               0         
_________________________________________________________________
dense_27 (Dense)             (None, 100)               15100     
_________________________________________________________________
dense_28 (Dense)             (None, 1)                 101       
=================================================================
Total params: 116,601
Trainable params: 116,301
Non-trainable params: 300
</code></pre>

## Test the model
The final step is to test the model for the left-off sample of 4 latter months. To do that we need preprocess the sample for the model input and then predict:

<pre><code>
# extract target
yTest = dfTest['&lt;CLOSE>'].values
xTest = dfTest.drop(columns='&lt;CLOSE>')

# scale
# scaler = MinMaxScaler(feature_range=(0,1))
xTest = scaler.transform(xTest)

# prep the samples fo the input to the recurrent nn 
xTestLstm = xTest.reshape(xTest.shape[0], 1, xTest.shape[1])

# predicting for test 
yPredRaw = model.predict(xTestLstm)

# plotting prediction
plt.plot(yTest, label='true')
plt.plot(yPredRaw, label='pred')
plt.legend()
plt.show()

print(f'mse: {mean_squared_error(y_true=yTest, y_pred=yPredRaw)}')
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-30-stock-price-prediction-only-history-13.png">

<pre><code>
print(f'mse: {mean_squared_error(y_true=yTest, y_pred=yPredRaw)}')
</code></pre>

<pre><code>
# true vs prediction comparison
print(pd.concat(
    (pd.DataFrame(yPredRaw, columns=['pred']), 
     pd.DataFrame(yTest, columns=['true'])
    ), 
    axis=1)[:10])
</code></pre>

Obviously the prediction turns out to be lower than the true value (probably because of the intense upside trend. Let's check can we at least predict the direction of the stock movement if compared today end vs tomorrow end.

<pre><code>
pd.options.mode.chained_assignment = None 

# checking direction 
cmpr = pd.concat(
    (pd.DataFrame(yTest, columns=['true'], index=dfTest.index),
    pd.DataFrame(yPredRaw, columns=['pred'], index=dfTest.index)),
    axis=1
)

cmpr['dir_true'] = np.zeros(cmpr.shape[0])
cmpr['dir_pred'] = np.zeros(cmpr.shape[0])
cmpr['corr_dir'] = np.zeros(cmpr.shape[0])
cmpr['dir_true'][0] = 0
cmpr['dir_pred'][0] = 0

for i in range(1, cmpr.shape[0]):
    cmpr['dir_true'][i] = cmpr['true'][i] - cmpr['true'][i-1]
    cmpr['dir_pred'][i] = cmpr['pred'][i] - cmpr['pred'][i-1]
    if np.sign(cmpr['dir_true'][i]) == np.sign(cmpr['dir_pred'][i]):
        cmpr['corr_dir'][i] = 1
    else: 
        cmpr['corr_dir'][i] = 0
        
print(
    f'================================================\n'
    f'the share of the correct predictions is '
    f'{round(cmpr.corr_dir.sum() / cmpr.corr_dir.shape[0]* 100, 2)} %\n'
    f'================================================\n'
)
</code></pre>

<pre><code>
================================================
the share of the correct predictions is 54.84 %
================================================

time: 44.7 ms (started: 2021-05-30 18:02:09 +00:00)
</code></pre>

In the case the prediction only out of the price history is no better than the coin toss. The further research might be continued with:

* more data re world market indices and other stocks 
* dynamic data about the company fundamential factors (revenue, costs, number of employees, etc)
* text parsing of the the news that might contain triggers for the price direction 	




