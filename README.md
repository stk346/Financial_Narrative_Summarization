# Financial_Narrative_Summarization

아이펠(Aiffel) 2차 해커톤 영어 재무재표요약(Financial Narrative Summarization) 해커톤입니다.

영국 Lancaster 대학에서 매 년 개최되는 conference에서 데이터를 가져왔습니다.


## 목차
  

* [데이터의 출처](#데이터의-출처)
* Preprocessing
* Modeling
* Evaluation


## 데이터의 출처
> * **input data:** 전체 재무제표 중에서 chief executive officer's review에 해당하는 section2를 사용했습니다.  
> * **Gold summary:** 영국의 재무제표는 PDF 형식으로 발행됩니다. 컨퍼런스를 개최하는 교수는 이를 문자로 변환한 이후 여러 snorkel과 같은 알고리즘을 적용해 전처리를 하여 Gold summary를 만들었습니다. 저희는 이 중 section2에 해당하는 부분을 label로 사용헸습니다.  

데이터 출처: http://multiling.iit.demokritos.gr/pages/view/1648/task-financial-narrative-summarization

초기 data의 shape:
| Data type | Train/Validation | Test |
|:----------|:----------------:|-----:|
| Full Text | 3,000 | 363 |
| Gold Summaries | 3,000 | 363 |

## Preprocessing

Data_Preprocessing/Dataset_preprocessing.ipynb에 해당하는 부분입니다.

우선 Train, Test의 문서 길이 분포를 측정한 뒤 적절한 부분을 잘라주었습니다. 이후 정규식을 이용해 저희가 원하지 않는 부분은 공백으로 치환했습니다.

하지만 이대로 모델링을 진행할 수는 없었습니다. 저희가 초기 baseline model로 삼았던 T5를 비롯한 bert 기반 모델의 경우 최대 인풋 길이가 512라는 점 때문이었습니다. 반면 저희가 가진 데이터의 75%에 해당하는 길이는 4,000 token이었습니다.  
label 또한 train data와 짝이 맞지 않거나 내용이 다른 것을 발견했습니다. 따라서 방향을 수정하게 되었습니다.

1. 추출요약, 생성요약을 사용해 label을 새로 만든다.
2. input sequence가 4,000자 정도 되는 모델을 이용해 summarization을 한다. (모델: longformer, 추출요약으로 label 생성)
3. T5 모델의 최대 input 길이인 512자로 document를 분리해 각각 summarization을 한다. (모델: pegasus, 생성요약으로 label 생성)


## Modeling

### 1. Longformer
longformer는 추출요약(extractive summarization) 기법 중 Edmundsonsummerizer와 Textrank 알고리즘을 이용했습니다. Edmundsonsummerizer 알고리즘의 경우 중요한 단어를 따로 지정해줄 수 있기 때문에 재무제표에 등장하는 중요한 문장을 더욱 잘 추출할 수 있다고 생각했기 때문입니다. 해당 기법은 문장의 개수를 지정해 요약을 진행할 수 있는데 저희는 12문장을 지정했습니다. 이 중 500 token이 넘어가는 text에 대해서는 Textrank로 label을 만들었습니다.

일반적인 attention 기법과 달리 longformer는 sparse attention 기법을 이용하여 더욱 긴 sequence를 input으로 넣을 수 있습니다. base의 경우 4096이며 large는 16,384입니다. 하지만 large 모델의 사이즈가 너무 컸기 때문에 저희는 base 모델을 이용했습니다.


### 2. Pegasus
Pegasus는 생성요약에 특화 되었으며 2020년 모델이 발표된 당시 12개 데이터셋에서 SOTA를 달성했습니다. 또한 이전 년도 FNS 컨퍼런스에서 좋은 성적을 냈던 만큼 해당 모델이 summarization task에 적합하다고 판단했습니다.  
label은 pre-trained T5에 train data를 잘라 넣어 각각 150자 정도를 생성했습니다. 예를 들어 문장의 token 개수가 4096개라면 512\*8개로 잘라서 넣어 주었습니다.

## Evaluation

**Rouge score**
|  | Rouge-1 | Rouge-L |
|:----|:-------:|:------:|
| Longformer | 0.54 | 0.39 |
| Pegasus | 0.34 | 0.21 |

저희는 각기 다른 방식으로 label을 추출했기에 rouge-score와 더불어 정성적 평가를 수행했습니다.

| | Text 길이 (input: 1683) |
|:---------|:--------------:|
| longformer | 342 words |
| pegasus | 110 words |

길이는 pegasus가 상대적으로 짧았습니다. 또한 longformer 모델은 생성요약을 사용하는데 마치 추출요약처럼 output이 나왔습니다. 이는 label을 추출요약을 사용해 만들었기 때문으로 보입니다. 롱포머는 문장 요약이 꽤 잘 수행되지만 글의 뒷부분 내용이 포함되지 않는 경우가 많습니다.  
  
 abstractive summarization은 문장을 생성하기 때문에 내용을 왜곡할 수 있는 여지가 있습니다. 모델 내 비슷하게 임베딩 된 단어들로 치환되는 과정에서 뜻이 약간 다르게 해석될 수 있는 부분이 있습니다. 또한 의문문을 써야 할 부분이 아닌데 물음표가 등장하곤 합니다. 하지만 요약이 잘 된 경우 3-4 문장에 해당하는 부분을 깔끔하게 처리했습니다.
