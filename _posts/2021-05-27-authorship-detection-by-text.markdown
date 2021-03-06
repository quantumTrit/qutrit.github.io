---
layout: post
title:  "Authorship Detection By Text" 
date:   2021-05-23 16:05:36 +0300
categories: jekyll update
---

**Probem set:** In this example we have the database of the public appeals (RUS) devided by categories. The task is to classify the upcoming appeals based on the text within.

<pre><code>
import numpy as np
import pandas as pd
from sklearn.preprocessing import LabelEncoder
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt

from tensorflow.keras.preprocessing.text import Tokenizer
from tensorflow.keras import utils
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense, Dropout, Embedding, Flatten, BatchNormalization
from tensorflow.keras.preprocessing.sequence import pad_sequences
</code></pre>

As the initial source we have the zip with the txt files already divided into the train and test split. Each file contains the the works of famous authors (Bradberry, Henry, Bulgakov...). Let's extract the zip into a single directory:

<pre><code>
!unzip *drive/My Drive/nn/_4/Тексты-писателей-20210525T170220Z-001.zip*
</code></pre>

The txt files' names containt the reference - either it is the train or the test. So let's collect them into the separate lists: one for the all the texts labeled as train, the other one - as test.

<pre><code>
def readText(fileName): 
    '''
    collect all the texts
    '''
    f = open(fileName, 'r')
    text = f.read()
    text = text.replace('\n', ' ')

    return text

# define the labels
labels = ['О. Генри', 'Стругацкие', 'Булгаков', 'Саймак', 'Фрай', 'Брэдберри']
nLabels = len(labels)

inputFolder = 'Тексты писателей/'

def collectTexts(inputFolder=inputFolder):
    '''
    collect all texts into the train and test smaples
    '''
    trainText, testText = [], []
    for label in labels:
        for fileName in os.listdir(inputFolder):
            if label in fileName:
                if 'Обучающая' in fileName:
                    trainText.append(readText(inputFolder + fileName))
                    print(f'{fileName} added to the train sample')
                if 'Тестовая' in fileName:
                    testText.append(readText(inputFolder+fileName))
                    print(f'{fileName} added to the test sample')

    return trainText, testText

trainText, testText = collectTexts()
</code></pre>

<pre><code>
(О. Генри) Тестовая_20 вместе.txt added to the test sample
(О. Генри) Обучающая_50 вместе.txt added to the train sample
(Стругацкие) Тестовая_2 вместе.txt added to the test sample
(Стругацкие) Обучающая_5 вместе.txt added to the train sample
(Булгаков) Тестовая_2 вместе.txt added to the test sample
(Булгаков) Обучающая_5 вместе.txt added to the train sample
(Клиффорд_Саймак) Тестовая_2 вместе.txt added to the test sample
(Клиффорд_Саймак) Обучающая_5 вместе.txt added to the train sample
(Макс Фрай) Тестовая_2 вместе.txt added to the test sample
(Макс Фрай) Обучающая_5 вместе.txt added to the train sample
(Рэй Брэдберри) Тестовая_8 вместе.txt added to the test sample
(Рэй Брэдберри) Обучающая_22 вместе.txt added to the train sample
</code></pre>

as the result we have the texts of 6 authors both in train and test samples

<pre><code>
print(f'train: {len(trainText)}, test: {len(testText)}')
</code></pre>

To speed up the model fit process and reduce RAM usage we set the up limit of the words to use from the sample:

<pre><code>
maxWordsCount = 20000
</code></pre>

*The* important step to pass the text to the model is to correctly convert it to the numerics. That can be performed in several steps:

**STEP 1:** word tokenization

We need to split our texts into the single words array and fit the tokenizer based on it.

<pre><code>
# all the symbols to exclude from the processed text
symbolsToFilter = *'!"#$%&()*+,-–—./…:;<=>?@[\\]^_`{|}~«»\t\n\xa0\ufeff'*

# tensorflow.keras.preprocessing.text
tokenizer = Tokenizer(
    num_words = maxWordsCount, # max words to be processed by the model
    filters = symbolsToFilter,
    lower = True, # enforce the lower register
    split = ' ', # split by space
    oov_token = 'unknown', # replaces all out-of-vocabulary words
    char_level = False # if True, every charcter is used as token
)

tokenizer.fit_on_texts(trainText)
items = list(tokenizer.word_index.items())
</code></pre>

Let's check the size of the resulting vocabulary top 10 frequent words in it:

<pre><code>
print(f'top 10 frequent words:\n{items[-10:]}')
print()
print(f'vocabulary size: {len(items)}')
</code></pre>

<pre><code>
top 10 frequent words:
[('поджарьте', 133061), ('заполните', 133062), ('мучающие', 133063), ('погремушкой', 133064), ('свистком', 133065), ('потерян', 133066), ('расплывающиеся', 133067), ('миллионе', 133068), ('зияющая', 133069), ('ничтонавстречу', 133070)]

vocabulary size: 133070
</code></pre>

This is how you can check the index of any word:

<pre><code>
def checkIndex(word=input()):
    try:
        return f'index is {tokenizer.word_index[word]} out of {len(items)}'
    except KeyError:
        return 'no such word in vocabulary'

checkIndex()
</code></pre>

**STEP 2:** convert tokens into the numeric sequence

We call the *fit_on_sequence* method for our tokenizer:

<pre><code>
trainWordIndices = tokenizer.texts_to_sequences(trainText)
testWordIndices = tokenizer.texts_to_sequences(testText)
</code></pre>

Let's check the origin and numeric sequence:

<pre><code>
print(f'original sequence\n{trainText[2][:80]}')
print(f'numeric sequence\n{trainWordIndices[2][:20]}')
</code></pre>

Now we can extract some statitics based on the results:

<pre><code>
myCount = {
    'symbolsInTrain' : 0,
    'wordsInTrain' : 0,
    'symbolsInTest' : 0,
    'wordsInTest' : 0,
}

print('-------------------')
print('TRAIN STATS:')
print('-------------------')
for i in range(nLabels):
    print(
        labels[i], 
        ' ' * (10 - len(labels[i])), 
        len(trainText[i]), 'symbols, ',
        len(trainWordIndices[i]), 'words'
    )
    myCount['symbolsInTrain'] += len(trainText[i])
    myCount['wordsInTrain'] += len(trainWordIndices[i])

print(
    'in total:', 
    ' ' * 1, 
    myCount['symbolsInTrain'], 'symbols, ',
    myCount['wordsInTrain'], 'words'
)
print('-------------------')

print('TEST STATS:')
print('-------------------')
for i in range(nLabels):
    print(
        labels[i], 
        ' ' * (10 - len(labels[i])), 
        len(testText[i]), 'symbols, ',
        len(testWordIndices[i]), 'words'
    )
    myCount['symbolsInTest'] += len(testText[i])
    myCount['wordsInTest'] += len(testWordIndices[i])

print(
    'in total:', 
    ' ' * 1, 
    myCount['symbolsInTest'], 'symbols, ',
    myCount['wordsInTest'], 'words'
)
print('-------------------')
</code></pre>

<pre><code>
-------------------
TRAIN STATS:
-------------------
О. Генри    1049517 symbols,  160607 words
Стругацкие  2042469 symbols,  313012 words
Булгаков    1765648 symbols,  261465 words
Саймак      1609507 symbols,  251502 words
Фрай        3700010 symbols,  568533 words
Брэдберри   1386454 symbols,  214454 words
in total:   11553605 symbols,  1769573 words
-------------------
TEST STATS:
-------------------
О. Генри    349662 symbols,  53238 words
Стругацкие  704846 symbols,  108621 words
Булгаков    875042 symbols,  132730 words
Саймак      318811 symbols,  50360 words
Фрай        1278191 symbols,  196731 words
Брэдберри   868673 symbols,  132524 words
in total:   4395225 symbols,  674204 words
-------------------
</code></pre>

**STEP 3:** Prepare the train and test inputs for the model.

This step implies the following sequence:

1. Get the array of the word indices (*wordIndices*)
2. Cut the sample with the given length (*xLen*)
3. Dump it to the container
4. Take a step from the begining of the previous sample (*step*)

<pre><code>
def getSetFromIndices(wordIndices, xLen, step):
    '''
    slice indices of words into the saples of the
    given length (xLen) by the given step
    '''
    xSample = []
    wordsLen = len(wordIndices)
    index = 0

    while (index + xLen <= wordsLen):
        xSample.append(wordIndices[index:index + xLen])
        index += step

    return xSample

def createSetsMultiClasses(wordIndices, xLen, step):
    '''
    prepare both the train and test input for the model 
    '''
    nLabels = len(wordIndices) # get the number of word indices
    classesXSamples = [] 
    for wIdx in wordIndices:
        classesXSamples.append(getSetFromIndices(wIdx, xLen, step))
    # classesXSamples shape:
    # labels count x steps count x steps length (xLen)

    xSamples, ySamples = [], []
    for l in range(nLabels): # for every label
        xL = classesXSamples[l] # collect all the arrays of the respective sample
        for i in range(len(xL)):
            xSamples.append(xL[i]) # dump each array into the features list
            ySamples.append(utils.to_categorical(l, nLabels)) # collect the label as a binary matrix

    xSamples, ySamples = np.array(xSamples), np.array(ySamples)

    return (xSamples, ySamples)

xLen = 1000
step = 100

xTrain, yTrain = createSetsMultiClasses(trainWordIndices, xLen, step)
xTest, yTest = createSetsMultiClasses(testWordIndices, xLen, step)
print(
    f'the shapes:\nxTrain: {xTrain.shape}, \nxTest: {xTest.shape}',
    f'\nyTest: {yTrain.shape}, \nyTest: {yTest.shape}'
)
</code></pre>

Finally we convert the resulting indices into the boolean matrix alike Bag of Words format. This can be performed with method *sequences_to_matrix*:

<pre><code>
xTrainB = tokenizer.sequences_to_matrix(xTrain.tolist())
xTestB = tokenizer.sequences_to_matrix(xTest.tolist())
</code></pre>

Example of what we get as the output:

<pre><code>
print(f'the shape: {xTrainB.shape}')
print('first 100 elements:')
print(xTrainB[0][:100])

the shape: (17640, 20000)
first 100 elements:
[0. 1. 1. 1. 1. 0. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 1. 0. 1. 1.
 0. 0. 1. 1. 0. 1. 1. 1. 1. 0. 0. 0. 1. 1. 1. 1. 1. 1. 1. 1. 0. 1. 1. 0.
 0. 1. 1. 1. 0. 0. 1. 1. 0. 0. 0. 1. 0. 0. 0. 1. 1. 0. 1. 1. 0. 0. 0. 0.
 1. 1. 0. 0. 1. 0. 1. 0. 1. 0. 0. 0. 0. 1. 1. 1. 1. 0. 0. 1. 0. 0. 1. 1.
 0. 0. 1. 1.]
</code></pre>

Following the preprocessing steps we can dive in the NN model build-up. The model architecture may vary, let's try dense layers with dropout and batchnormalization.

Dropout layer randomly sets input units to 0 with a frequency of rate at each step during training time, which helps prevent overfitting. 

We will apply *ReLU* as the activation function to the dense layer. The function will output the input directly if it is positive, otherwise, it will output zero. The math formula:

$$ y = max(0, x) $$

As the last activation function we apply *softmax* which take input of N numbers and normalizes it (by applying the standard exponential function) into N probabilites proportional to the exponentials if the input numbers.
Tha math formula:

$$ \sigma(z)_i = \frac{e^{z_i}}{\Sigma^K_{j=1} e^{z_i}} $$

The loss function applied is the cross-entropy loss, or log loss, whcik measures the performance of a classification model whose output is a probability value between 0 and 1. Cross-entropy loss increases as the predicted probability diverges from the actual label. The math formula:

$$ CE = - \Sigma^C_i t_i x log(s_i) $$  

where:

$ t_i $ is the groundtruth

$ s_i $ is the model prediction

$ i $ is a label

$ C $ is the number of labels

<pre><code>
modelD = Sequential()
modelD.add(Dense(
    200, # number of neurons
    input_dim = maxWordsCount, # the input shape must be passed to the 1st stated layer 
    activation='relu' 
))
modelD.add(Dropout(0.25))
modelD.add(BatchNormalization())
modelD.add(Dense(nLabels, activation='softmax'))

modelD.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

history = modelD.fit(
    xTrainB,
    yTrain,
    epochs=20,
    batch_size=128,
    validation_data = (xTestB, yTest)
)

plt.plot(history.history['accuracy'], label='correct answers @train')
plt.plot(history.history['val_accuracy'], label='correct answers @test')
plt.xlabel('epoch')
plt.ylabel('correct prediction share')
plt.legend()
plt.show()
</code></pre>

<pre><code>
Epoch 1/20
138/138 [==============================] - 11s 75ms/step - loss: 0.0505 - accuracy: 0.9850 - val_loss: 0.3764 - val_accuracy: 0.9037
Epoch 2/20
138/138 [==============================] - 10s 72ms/step - loss: 3.6209e-04 - accuracy: 1.0000 - val_loss: 0.3164 - val_accuracy: 0.9068
Epoch 3/20
138/138 [==============================] - 10s 72ms/step - loss: 1.7111e-04 - accuracy: 1.0000 - val_loss: 0.3161 - val_accuracy: 0.9014
Epoch 4/20
138/138 [==============================] - 10s 75ms/step - loss: 1.0510e-04 - accuracy: 1.0000 - val_loss: 0.3092 - val_accuracy: 0.9014
Epoch 5/20
138/138 [==============================] - 10s 72ms/step - loss: 8.1488e-05 - accuracy: 1.0000 - val_loss: 0.3041 - val_accuracy: 0.9023
Epoch 6/20
138/138 [==============================] - 10s 71ms/step - loss: 6.0976e-05 - accuracy: 1.0000 - val_loss: 0.3013 - val_accuracy: 0.9035
Epoch 7/20
138/138 [==============================] - 10s 72ms/step - loss: 4.6316e-05 - accuracy: 1.0000 - val_loss: 0.2974 - val_accuracy: 0.9046
Epoch 8/20
138/138 [==============================] - 10s 72ms/step - loss: 3.8414e-05 - accuracy: 1.0000 - val_loss: 0.3013 - val_accuracy: 0.9035
Epoch 9/20
138/138 [==============================] - 10s 73ms/step - loss: 3.3897e-05 - accuracy: 1.0000 - val_loss: 0.3007 - val_accuracy: 0.9026
Epoch 10/20
138/138 [==============================] - 10s 74ms/step - loss: 2.9758e-05 - accuracy: 1.0000 - val_loss: 0.2971 - val_accuracy: 0.9043
Epoch 11/20
138/138 [==============================] - 10s 73ms/step - loss: 2.4322e-05 - accuracy: 1.0000 - val_loss: 0.2957 - val_accuracy: 0.9047
Epoch 12/20
138/138 [==============================] - 10s 72ms/step - loss: 2.0997e-05 - accuracy: 1.0000 - val_loss: 0.2963 - val_accuracy: 0.9040
Epoch 13/20
138/138 [==============================] - 10s 72ms/step - loss: 1.8370e-05 - accuracy: 1.0000 - val_loss: 0.2963 - val_accuracy: 0.9040
Epoch 14/20
138/138 [==============================] - 10s 72ms/step - loss: 1.8250e-05 - accuracy: 1.0000 - val_loss: 0.2963 - val_accuracy: 0.9044
Epoch 15/20
138/138 [==============================] - 10s 71ms/step - loss: 1.6745e-05 - accuracy: 1.0000 - val_loss: 0.2952 - val_accuracy: 0.9029
Epoch 16/20
138/138 [==============================] - 10s 71ms/step - loss: 1.4096e-05 - accuracy: 1.0000 - val_loss: 0.2960 - val_accuracy: 0.9025
Epoch 17/20
138/138 [==============================] - 10s 72ms/step - loss: 1.2515e-05 - accuracy: 1.0000 - val_loss: 0.2949 - val_accuracy: 0.9025
Epoch 18/20
138/138 [==============================] - 10s 72ms/step - loss: 1.2019e-05 - accuracy: 1.0000 - val_loss: 0.2923 - val_accuracy: 0.9034
Epoch 19/20
138/138 [==============================] - 10s 71ms/step - loss: 1.0969e-05 - accuracy: 1.0000 - val_loss: 0.2951 - val_accuracy: 0.9032
Epoch 20/20
138/138 [==============================] - 10s 71ms/step - loss: 9.8800e-06 - accuracy: 1.0000 - val_loss: 0.2933 - val_accuracy: 0.9031
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-27-authorship-detection-by-text-1.png">

Upon model fit we can pass the test samples to validate the accuracy. We can do that in 2 steps:

1. collect the binary matrices for the test sample (similiar to what we have done with the train sample)

2. Pass the matrices into the model for the prediction & check the score

<pre><code>
def createTestMultiClasses(wordIndices, xLen, step):
    nLabels = len(wordIndices)
    xTestLabels, xTestLabelsB = [], []
    for wIdx in wordIndices:
        sample = (getSetFromIndices(wIdx, xLen, step))
        xTestLabels.append(sample)
        xTestLabelsB.append(tokenizer.sequences_to_matrix(sample))

    xTestLabels, xTestLabelsB = np.array(xTestLabels, dtype=object), np.array(xTestLabelsB, dtype=object)

    return xTestLabels, xTestLabelsB

def recognizeMultiLabels(model, xTest, modelName):
    print(f'model: {modelName}')

    totalSumRec = 0 # sum up the correct predictions

    for i in range(nLabels):
        currPred = model.predict(xTest[i]) # predict
        currOut = np.argmax(currPred, axis=1) # get the index of the max element

        evVal = []
        for j in range(nLabels): # for every label
            evVal.append(len(currOut[currOut==j]) / len(xTest[i]))
        totalSumRec += len(currOut[currOut==i])
        recognizedLabel = np.argmax(evVal)

        isRecognized = 'false prediction'
        if (recognizedLabel == i):
            isRecognized = 'correct prediction'

        tempStr = 'Label: ' + labels[i] + ' ' * (11 - len(labels[i])) +\
        str(int(100*evVal[i])) + '% the model recognized as ' + labels[recognizedLabel]
        print(tempStr, ' ' * (55 - len(tempStr)), isRecognized, sep ='')

    print()
    sumCount = 0
    for i in range(nLabels):
        sumCount += len(xTest[i])
    print(f'average prediction rate: {int(100*totalSumRec/sumCount)} %')

    return totalSumRec / sumCount

xTestLabels, xTestLabelsB = createTestMultiClasses(testWordIndices, xLen, step)
pred = recognizeMultiLabels(modelD, xTestLabelsB, 'Dense')
</code></pre>

output:
<pre><code>
model: Dense
Label: О. Генри   93% the model recognized as О. Генри correct prediction
Label: Стругацкие 89% the model recognized as Стругацкиеcorrect prediction
Label: Булгаков   83% the model recognized as Булгаков correct prediction
Label: Саймак     87% the model recognized as Саймак   correct prediction
Label: Фрай       89% the model recognized as Фрай     correct prediction
Label: Брэдберри  98% the model recognized as Брэдберриcorrect prediction

average prediction rate: 90 %
</code></pre>

Let's check what happens if we use not the matrix but the array of the indeces itself. W'll apply the *Embedding* layer which turns positive integers into dense vectors of the fixed size. 

<pre><code>
modelE = Sequential()
modelE.add(Embedding(maxWordsCount, 20, input_length=xLen))
modelE.add(Dropout(0.2))
modelE.add(Flatten())
modelE.add(BatchNormalization())
modelE.add(Dense(200, activation='relu'))
modelE.add(Dropout(0.2))
modelE.add(BatchNormalization())
modelE.add(Dense(6, activation='sigmoid'))

modelE.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

history = modelE.fit(
    xTrain, yTrain,
    epochs = 20,
    batch_size = 128,
    validation_data = (xTest, yTest)
)

plt.plot(history.history['accuracy'], label='correct answers @train')
plt.plot(history.history['val_accuracy'], label='correct answers @test')
plt.xlabel('epoch')
plt.ylabel('correct prediction share')
plt.legend()
plt.show()
</code></pre>

output:
<pre><code>
Epoch 1/20
138/138 [==============================] - 25s 178ms/step - loss: 0.7350 - accuracy: 0.7702 - val_loss: 1.8049 - val_accuracy: 0.3365
Epoch 2/20
138/138 [==============================] - 24s 173ms/step - loss: 0.0077 - accuracy: 0.9998 - val_loss: 1.5086 - val_accuracy: 0.4403
Epoch 3/20
138/138 [==============================] - 24s 172ms/step - loss: 0.0035 - accuracy: 1.0000 - val_loss: 1.1990 - val_accuracy: 0.5156
Epoch 4/20
138/138 [==============================] - 24s 172ms/step - loss: 0.0019 - accuracy: 1.0000 - val_loss: 0.9421 - val_accuracy: 0.6415
Epoch 5/20
138/138 [==============================] - 24s 172ms/step - loss: 0.0013 - accuracy: 1.0000 - val_loss: 0.8406 - val_accuracy: 0.6988
Epoch 6/20
138/138 [==============================] - 24s 175ms/step - loss: 8.7682e-04 - accuracy: 1.0000 - val_loss: 0.8153 - val_accuracy: 0.7188
Epoch 7/20
138/138 [==============================] - 24s 172ms/step - loss: 6.7539e-04 - accuracy: 1.0000 - val_loss: 0.8051 - val_accuracy: 0.7278
Epoch 8/20
138/138 [==============================] - 24s 173ms/step - loss: 5.7175e-04 - accuracy: 1.0000 - val_loss: 0.7913 - val_accuracy: 0.7341
Epoch 9/20
138/138 [==============================] - 24s 175ms/step - loss: 4.4960e-04 - accuracy: 1.0000 - val_loss: 0.7881 - val_accuracy: 0.7374
Epoch 10/20
138/138 [==============================] - 24s 173ms/step - loss: 3.5192e-04 - accuracy: 1.0000 - val_loss: 0.7890 - val_accuracy: 0.7375
Epoch 11/20
138/138 [==============================] - 24s 173ms/step - loss: 2.9417e-04 - accuracy: 1.0000 - val_loss: 0.7839 - val_accuracy: 0.7402
Epoch 12/20
138/138 [==============================] - 24s 172ms/step - loss: 2.7420e-04 - accuracy: 1.0000 - val_loss: 0.7804 - val_accuracy: 0.7424
Epoch 13/20
138/138 [==============================] - 24s 174ms/step - loss: 2.2254e-04 - accuracy: 1.0000 - val_loss: 0.7778 - val_accuracy: 0.7439
Epoch 14/20
138/138 [==============================] - 24s 173ms/step - loss: 1.8824e-04 - accuracy: 1.0000 - val_loss: 0.7788 - val_accuracy: 0.7444
Epoch 15/20
138/138 [==============================] - 24s 173ms/step - loss: 1.6237e-04 - accuracy: 1.0000 - val_loss: 0.7760 - val_accuracy: 0.7463
Epoch 16/20
138/138 [==============================] - 24s 173ms/step - loss: 1.4674e-04 - accuracy: 1.0000 - val_loss: 0.7728 - val_accuracy: 0.7475
Epoch 17/20
138/138 [==============================] - 24s 174ms/step - loss: 1.2645e-04 - accuracy: 1.0000 - val_loss: 0.7730 - val_accuracy: 0.7483
Epoch 18/20
138/138 [==============================] - 24s 176ms/step - loss: 1.0677e-04 - accuracy: 1.0000 - val_loss: 0.7640 - val_accuracy: 0.7540
Epoch 19/20
138/138 [==============================] - 24s 176ms/step - loss: 9.9545e-05 - accuracy: 1.0000 - val_loss: 0.7673 - val_accuracy: 0.7519
Epoch 20/20
138/138 [==============================] - 24s 176ms/step - loss: 9.2081e-05 - accuracy: 1.0000 - val_loss: 0.7698 - val_accuracy: 0.7519
</code></pre>

<img src="{{site.baseurl}}/assets/img/2021-05-27-authorship-detection-by-text-2.png">

<pre><code>
xTestLabels, xTestLabelsB = createTestMultiClasses(testWordIndices, xLen, step)
pred = recognizeMultiLabels(modelE, xTestLabels, 'Dense')
</code></pre>

output:
<pre><code>
model: Dense
Label: О. Генри   86% the model recognized as О. Генри correct prediction
Label: Стругацкие 84% the model recognized as Стругацкиеcorrect prediction
Label: Булгаков   58% the model recognized as Булгаков correct prediction
Label: Саймак     67% the model recognized as Саймак   correct prediction
Label: Фрай       83% the model recognized as Фрай     correct prediction
Label: Брэдберри  71% the model recognized as Брэдберриcorrect prediction

average prediction rate: 75 %
</code></pre>

In general, the text processing can be drawn as follows:

<img src="{{site.baseurl}}/assets/img/2021-05-27-authorship-detection-by-text-4.png">

