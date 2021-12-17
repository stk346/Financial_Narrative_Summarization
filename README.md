# Financial_Narrative_Summarization

아이펠(Aiffel) 2차 해커톤 영어 재무재표요약(Financial Narrative Summarization) 해커톤입니다.

영국 Lancaster 대학에서 매 년 개최되는 conference에서 데이터를 가져왔습니다.


## 목차
  

* 데이터의 출처
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

하지만 이대로 모델링을 진행할 수는 없었습니다. 저희가 초기 baseline model로 삼았던 T5를 비롯한 bert 기반 모델의 경우 최대 인풋 길이가 512라는 점이었습니다. 반면 저희가 가진 데이터의 75%에 해당하는 길이는 4,000 token이었습니다.  
label 또한 train data와 짝이 맞지 않거나 내용이 다른 것을 발견했습니다. 따라서 방향을 수정하게 되었습니다.

1. 추출요약, 생성요약을 사용해 label을 새로 만든다.
2. input sequence가 4,000자 정도 되는 모델을 이용해 summarization을 한다. (모델: longformer, 추출요약으로 label 생성)
3. T5 모델의 최대 input 길이인 512자로 document를 분리해 각각 summarization을 한다. (모델: pegasus, 생성요약으로 label 생성)


## Modeling

### 1. Longformer
longformer는 추출요약(extractive summarization)을 사용했습니다. 
