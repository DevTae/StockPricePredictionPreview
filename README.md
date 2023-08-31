### 주식 가격 예측 모델 개발 일지

<br/>

### 2023.7.24 - 2023.8.3

#### 진행사항
  - `주가 데이터 수집`(보조지표 포함)과 metadata 형식으로 `전처리 과정` 진행
  - `주가 분석 모델 툴킷 제작` 및 `모델 설계` (CNN + RNN 기반) 진행
  - `하이퍼 파라미터 설정` 및 학습 진행

<br/>

### 2023.8.6

#### 현재 문제점
  - 70~100 epochs 를 최종 목표로 하고 있는 도중, 10, 20, 30 epochs 일 때의 Training 과 Validation 에 대한 MSE 와 Loss 를 보면서 모델 품질에 대하여 판단하는 중이다.
  - 그 결과, Training 에 대해서는 MSE 가 대폭 줄어들고 있지만, Validation 에 대해서는 MSE 가 줄고 있지 않는 `과적합 상태`를 발견하였다.

#### 예상되는 원인
  - 현재 metadata.txt 에서 시장과 종목 코드를 기준으로 정렬하게 해두었는데, 이로 인하여 Validation 데이터셋 쪽에 변동폭이 큰 종목들이 대거 들어가게 되어 그렇게 된 것 같다.

#### 데이터에 대한 고찰 (23.8.6)

- 가격 움직임에 대하여 변동성이 큰 종목과 변동성이 작은 종목에 대한 비율이 비슷한가?
  - 변동성에 대한 지표를 바탕으로 변동성이 큰 종목과 변동성이 작은 종목에 대한 비율을 조사한다. 이것도 히스토그램으로 얼마나 분포되어 있는지 확인할 수 있어야 함.
  
- 수익률이 기존 가격에 근접한 종목과 큰 수익을 달성하는 종목의 비율이 어떻게 구성되어 있나?
  - 히스토그램을 바탕으로 수익에 있어서 10% 단위에 대하여 데이터의 분산을 확인한다. 
  - 만약, 비율이 다르다면 어떻게 진행하는 것이 좋을 것 같나?
  - 일단, train 과 validation 데이터를 골고루 분산시키는 것이 좋겠다. metadata.txt 에 있는 데이터들을 기반으로 각 데이터들에 대하여 sort 를 진행하고 만약 train 과 validation 사이의 관계가 10:1 이라면 (전체데이터개수)/11 (=x) 을 계산하여 10x 를 train 데이터에 넣고, x 를 validation 데이터에 집어넣는 방식으로 진행.
  - 비교적 변동성이 큰 종목의 데이터 개수가 모자랄 것이라는 생각이 든다. 이에 대하여 데이터 개수가 부족한 부분에 대하여 데이터 증강을 통하여 접근할 수 있도록하면 좋겠다는 생각을 하였다.
  
- 데이터 증강의 경우, 정확하게 어떤 방식을 선택할 것인가?
  - 현재, Time Domain 에 있어서 기본적으로 제안하는 증강 기법은 Window cropping or slicing, Window warping, Flipping, Noise injection, Label expansion 정도가 있다. 나에게 적합한 방식은 Noise Injection 가 될 것 같다. 노이즈로는 gaussian noise, spike, step-like trend, and slope-like trend 정도가 있다고 한다. 최종적으로 gaussian noise 를 적용할 예정이다.
  - 당장에는 시계열 데이터의 증강을 적용하므로써 생길 데이터의 변질이 우려되어 적용하고 있지 않는 중이다.

- **결론**
  - `random.shuffle` 함수를 바탕으로 **변동성이 큰 종목이 몰려 있는 validation 데이터셋**과 **training 데이터셋**을 섞는 것을 목표로 하였음.
  - 위 방법을 통하여 Training Set 과 Validation Set 의 MSE 가 엄청나게 많이 차이나는 현상을 해결할 수 있었음.

<br/>

### 2023.8.10

#### EDA 접근
- 결과층 데이터가 `특정 구간에 몰려 있는 것`을 확인 / 이러한 부분이 `데이터 불균형`을 뜻하고 `학습 평가 검증`이 다소 어려워질 수 있음
- 따라서, `결과값 확대 및 축소 함수(log 및 sqrt 공식)`를 적용하여 다음과 같이 `정규 분포 형태의 데이터 분포`로 구성할 수 있었다.
  
  ![화면 캡처 2023-08-13 031115](https://github.com/DevTae/StockDatabasePreview/assets/55177359/b9340afa-88bd-4cf8-b92b-9c0c3dbb580b)
- 데이터 분포의 문제를 해결할 수 있어 당장에는 Data Augmentation 을 적용하지 않고 학습을 진행하고자 한다.
  - 데이터 수는 약 160만 개로 충분히 커버가 가능할 듯 보인다.

<br/>

### 2023.08.19

#### 모델 구조 변경
  - `CNN + RNN` 결합한 모델을 바탕으로 학습을 돌려본 결과, 생각보다 좋은 성능이 나오질 않았다.
    - `평균 약 5~6% 내외`의 오차율을 보임 / `MSE 1000`, `MAE 28`
      - 평가지표의 경우, 결과층에 의도적으로 Scaling 을 진행하지 않아 이런 결과가 나오게 되었음. (뒤에 모두 비슷한 스케일의 검증 결과가 나올 예정임)
  - 이후, `LSTM` 을 바탕으로 `Layer Normalization 기법`까지 적용하여 모델을 구성하였고 학습을 시작할 수 있었다.

<br/>

### 2023.08.20

#### 하이퍼파라미터 튜닝 및 전처리 스케일링 방식 변경
  - 모델 구조를 바꿨음에도 `MSE 가 1000 부근`으로 나옴으로 `하이퍼파라미터 설정`과 `스케일링 방식`을 변경하기로 하였다.
  - 학습률, Epoch, RNN 레이어 개수, Hidden Dimension 에 대한 하이퍼파라미터 튜닝을 진행하였다.
  - 마지막 종가를 바탕으로 모든 수치를 계산하지 않고 `전날 종가를 기준으로 수치를 스케일링`하여 변환하였다.
  
<br/>

### 2023.08.21 ~ 2023.08.22

#### 현재 학습 결과
  - 새롭게 **결과층에 변환된 값을 해제**하는 공식을 적용하여 `PriceMeanSquaredError` 평가지표를 바탕으로 `PMSE 가 39.19` 정도가 나왔다.
    - 20 Epochs 에 `PMSE 가 39.19` (이전 MSE 와 기준이 다름) 정도가 나오며 평균적으로 `약 6.2% 의 오차`를 가지고 있다는 것을 알 수 있다.
      - (23.8.25) 위의 평균 6.2% 의 오차라는 것은 많이 `편향(bias)`된 검증 결과이다. 따라서, `데이터 분포 변환의 필요성`을 느끼게 됨.
  - 이전의 CNN + RNN 결합한 모델과 그렇게 큰 차이를 보이진 않는 것으로 보여, `GRU 모델`을 바탕으로 진행하고자 한다.
  
#### 이후 방향성
  - 스케일링 방식을 바탕으로 Feature 들에 대하여 `Non-Stationary 데이터`가 되도록 변경할 것이다.
  - `하이퍼 파라미터 튜닝`을 계속하여 진행할 것이다.

<br/>

### 2023.08.23 ~ 2023.08.25

#### 현재 진행 상황
  - 스케일링 방식을 변형하여 모든 데이터들에 대하여 `Non-Stationary 데이터`로 변환하였음.
  - 이전 데이터셋은 모양만 정규분포와 유사하였고, `Shapiro-Wilk 검정`을 진행한 결과, 정규분포가 아님을 확인하여 이에 대한 데이터 선정 알고리즘을 적용하고자 함.
    - 추가적으로, 정규분포보다 `균등분포 (Uniform Distribution)`에 더 가깝도록 데이터를 설계하고자 하였고 `일련의 보정`을 진행하였음.
      - 결과층의 반환값이 편향된 분포를 가지면 안 되는 이유로 데이터를 선정하였고 고르게 분포하도록 했기 때문에 이전 데이터셋의 검증 결과와 비교할 수 없음.
    - `QQ Plot` 을 진행한 결과, 다음과 같이 균등하게 분포하도록 개선할 수 있었음.
      
      ![image](https://github.com/DevTae/StockDatabasePreview/assets/55177359/a66c4bc9-f106-4cb1-b83a-c44932a73f2d)
  - 전처리 방식 변경
    - 출력층 기준 변경 (look-ahead period)
    - 데이터 선정 알고리즘 이외에도 `변동성 차원에서의 지표 조건을 추가`하여 데이터 선정 (도메인 지식 적용)

<br/>

### 2023.8.26

#### 현재 진행 상황
  - 하이퍼 파라미터 설정 진행 중에 `warmup steps` 에 관련된 실험을 진행 중이다.
    - `warmup steps` 를 전체 step size (모든 epochs) 의 **약 10% 가량**으로 설정한 것과 그것보다 훨씬 적게 **약 1000 단계**로 설정하는 것과의 차이를 비교해보았다.
    - 그 결과, `후자의 경우`가 MSE 에 있어서 비교적 빠르게 수렴하는 것을 확인할 수 있었다.
    - 예상되는 이유로는 `warmup steps` 가 많을 경우, 초반에 빠르게 학습하지 못하여 이후 전체 학습이 느려지는 이유인 것 같고, 추가적으로 초반에 `local minima` 가 발생할 수 있다는 점 때문이다.
    - 따라서, 빠른 수렴 이후 안정성까지 어느 정도 조화를 이루는 적절한 `warmup steps` 수치를 정하는 것이 좋겠다.
    - 결과적으로는 **1 epoch 에 대한 step size 의 약 5% 가량**으로 설정 후 진행하고자 한다.

<br/>

### 2023.8.29

#### 현재 진행 상황
  - 데이터 수치가 `lower boundary` 와 `upper boundary` 를 벗어나는 경우, 제외하지 않고 `clipping` 하는 방식으로 데이터를 보존하였다.
    - 균등 분포를 만드는 과정에서 데이터를 제외하는 이유로 전체 데이터 개수가 부족해지는 현상이 생기기 때문에 위 방식을 적용하게 되었다.
    - 이상치 데이터 개수가 적기 때문에 약간의 데이터를 살리는 것만 해도 상당한 비율의 데이터를 구하게 되는 것이다.
    ![image](https://github.com/DevTae/StockPricePredictionPreview/assets/55177359/0c6ffa7c-b1a0-415c-bccc-05941c134409)
  - 출력층 기준 변경 (look-back period, look-ahead period)
  - 출력층 산정 방식 변경 (수식을 바탕으로 상승 확률로 표현)
  - `Deadlock` 을 해결하기 위하여 `해시 함수`를 활용하여 `Disk I/O` 를 줄이고자 한다.
    - 현재, 데이터 추출 방식을 동일한 raw data 에서 반복적으로 읽는 방법으로 진행하고 있는데, 이러한 구조 때문인지 학습 도중 1 step 이 이전의 1000 배의 시간이 걸리는 현상이 발생하었다.
    - `num_workers` 가 4 로 설정되어 있었고, 디버깅해보았을 때 `CUDA Out of Memory` 등의 오류가 발생하지 않았다는 점에서 동일한 파일에 대한 여러 번의 접근으로 인한 `Deadlock` 발생에 대한 의심을 하는 중이다.
    - 이러한 상황에서 가장 먼저 해볼 목표는 `해시 함수`를 활용하여 `Disk I/O` 를 줄이는 것이다.

      ![image](https://github.com/DevTae/StockPricePredictionPreview/assets/55177359/598338d9-e678-49ef-945d-8c0a9a79b39a)
    - 그 결과, 성공적으로 다음과 같이 초반에 데이터를 읽어들인 이후에는 메모리에서 가져오기 때문에 성공적으로 `Disk I/O` 를 줄일 수 있었고, 그에 따라 평균 소요 시간도 `약 75% 가량 절감`할 수 있었다.
  - 위와 같이 `Disk I/O` 를 줄였음에도 학습 진행 중 똑같이 `Deadlock` 현상이 발생하였고, 추가적으로 `Memory Leak` 현상이 의심되어 매 반복문에서 할당되는 `torch`, `cpu` 변수를 해제하도록 하였다.
  - 앞으로는 하이퍼파라미터 튜닝을 진행하며 라이브러리인 `optuna` 또한 학습해볼 예정이다.
  - 현재 부정부터 긍정까지 `0 ~ 100` 사이에서 표현(*서비스 상에서*)하도록 하였고, `평균 오차 목표`를 `전체의 10%` 정도로 잡았으며, 이에 따라 `PMSE` 가 `25` 정도가 되는 것이 목표이다.
    - (23.8.31) 현재, `-1 ~ 1` 사이의 출력 목표 기준으로 `0.2` 정도의 오차를 허용하며 `최종 목표 MSE` 는 `0.04` 로 재설정하였다.

<br/>

### 2023.8.31

#### 현재 진행 상황
  - `torch` 의 `to` 메소드에 대하여, 원래의 `torch` 값이 소실되는 우려를 바탕으로 `clone` 으로 `cpu` 에 복제 후 평가지표 계산하는 데에 쓰이도록 변경하였다.
  - 학습 중 Loss 가 줄어들지 않고 있는 상황임을 발견하였다.
    - 모델 구조 변경 시도 (`Transpose`, `Dropout`, `Linear`, `Xavier Uniform`, `RReLU` 등 적용)
    - Scaling 기준 변경 (입력값 및 출력값 모두 `-1 ~ 1` 사이로 스케일링 진행)
      - 따라서, `목표 MSE` 는 출력층 데이터에서의 0.2 정도 오차로 `0.04` 로 재설정

<br/>
