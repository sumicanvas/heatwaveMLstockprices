# HeatWave autoML - predict the next day's stock price
Predict the next day's stock price using HeatWave AutoML


### 1.	학습 데이터 및 테스트 데이터 로딩
-  **파이썬 프로그램 실행전 반드시 DB(stockdb)를 생성**해줘야 합니다. Python 프로그램으로 Yahoo Finance api 를 이용해서 10년간의 주식 데이터를 가지고 옵니다. <br>

```
print(f"'{TICKER_SYMBOL}' 주식 데이터 다운로드를 시작합니다...")
df = yf.download(TICKER_SYMBOL, start="2015-09-24", end="2025-09-29")

.. skip..

try:
    connection_str = f"mysql+mysqlconnector://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}"
    engine = create_engine(connection_str)

    with engine.connect() as connection:
        create_table_query = text(f"""
        CREATE TABLE IF NOT EXISTS {TABLE_NAME} (
            Ticker VARCHAR(10),
            Date DATE,
            Open DECIMAL(10, 2),
            High DECIMAL(10, 2),
            Low DECIMAL(10, 2),
            Close DECIMAL(10, 2),
            Volume BIGINT,
            PRIMARY KEY (Ticker, Date)
        );
        """)
        connection.execute(create_table_query)

```

### 2.	MySQL heatwave에 접속해서 테이블과 데이터 확인
-  테이블 및 컬럼 확인 <br>
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/ab79946e-46de-4baf-97dd-f3157db01099" />
<br>
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/39f0a8d4-57cf-48f6-ad2b-0126f63e37eb" />
<br>
<br>

- next_day_close 컬럼 추가 (실제 다음날값으로 설정하고 예측값과 이후 비교용)

```
ALTER TABLE stock_prices ADD COLUMN next_day_close DECIMAL(10, 2);
``` 
<br>


- next_day_close 값을 넣어줘야 함

```
-- CTE(Common Table Expression)를 사용하여 먼저 다음 날 종가를 계산합니다.
WITH next_day_data AS (
    SELECT
        Ticker,
        Date,
        -- LEAD() 함수로 다음 거래일의 종가를 가져옵니다.
        -- PARTITION BY Ticker: 종목별로 그룹을 나눕니다.
        -- ORDER BY Date: 날짜순으로 정렬하여 '다음'의 기준을 정합니다.
        LEAD(Close, 1) OVER (PARTITION BY Ticker ORDER BY Date) AS next_close
    FROM
        stock_prices
)
-- 위에서 계산된 결과를 원래 테이블에 JOIN하여 UPDATE합니다.
UPDATE stock_prices s
JOIN next_day_data n ON s.Ticker = n.Ticker AND s.Date = n.Date
SET s.next_day_close = n.next_close;

```
<br>
- 마지막날 next_day_close 값 만들어서 추가해줘야 하는데 간단하게 당일 종가와 동일한 값을 입력하기로 이테스트에서는 정함.

```
-- NULL 인 값을 찾음(최종날짜)
SELECT COUNT(*) FROM stock_prices WHERE next_day_close IS NULL;

-- 최종일 종가와 동일한 값을 입력
UPDATE stock_prices s1
SET s1.next_day_close = s1.Close
WHERE s1.Date = (
    -- s1과 동일한 티커를 가진 데이터 중에서 가장 마지막 날짜를 찾음
    SELECT MAX(s2.Date)
    FROM (SELECT * FROM stock_prices) s2
    WHERE s2.Ticker = s1.Ticker
);
```
<br>

- 훈련 데이터와 테스트 데이터 준비 (각가 별개 테이블로 생성)
```
-- 1. 훈련 데이터 테이블 생성 (예: 2024년 이전 데이터)
CREATE TABLE stock_train_data AS
SELECT * FROM stock_prices
WHERE Date < '2024-01-01';

-- 2. 테스트 데이터 테이블 생성 (예: 2024년 이후 데이터)
CREATE TABLE stock_test_data AS
SELECT * FROM stock_prices
WHERE Date >= '2024-01-01';

-- 3. 생성된 훈련 데이터와 테스트 데이터 확인
select * from stock_train_data order by Date desc limit 10;

select * from stock_test_data order by Date desc limit 10;

```
<br>

### 3.	MySQL heatwave 노드에 LOAD

```
CALL sys.heatwave_load(JSON_ARRAY("stockdb"),NULL);
```

<img width="700" height="776" alt="image" src="https://github.com/user-attachments/assets/b1c9b31e-db54-40ac-8c6b-39c2c1c31555" /><br>

### 4.	MySQL heatwave 훈련 후 모델생성
```
-- HeatWave 머신러닝 훈련 프로시저 호출
CALL sys.ML_TRAIN(
    'stockdb.stock_train_data', -- 모델을 학습시킬 데이터가 담긴 테이블
    'next_day_close',   --  목표 컬럼 (Target): 모델이 예측해야 할 정답 컬럼
    JSON_OBJECT(
        'task', 'regression',    -- 머신러닝 문제의 유형을 '회귀(regression)'로 지정
        -- 'optimization_metric': 모델 성능 최적화의 기준이 되는 평가지표입니다.
        -- 'neg_mean_squared_error'는 평균 제곱 오차(MSE)를 최소화하겠다는 의미입니다.
        'optimization_metric', 'neg_mean_squared_error',
        'exclude_column_list', JSON_ARRAY('Ticker', 'Date')    -- 'exclude_column_list': 학습에 사용하지 않을 컬럼 목록
    ),
    @stock_model
);

```
- 모델확인

<img width="700" height="244" alt="image" src="https://github.com/user-attachments/assets/f91de446-7c93-41f6-a953-0936c3a238ac" /><br>

```
select @stock_model;

SELECT model_id, model_handle, train_table_name FROM ML_SCHEMA_admin.MODEL_CATALOG\G
```
<br>

### 5.	MySQL heatwave 모델 로드
```
CALL sys.ML_MODEL_LOAD(@stock_model, NULL);
```

### 6.	MySQL heatwave 예측
```
CALL sys.ML_PREDICT_TABLE('stockdb.stock_test_data', @stock_model, 'stockdb.test_predictions', NULL);
```
<img width="1754" height="126" alt="image" src="https://github.com/user-attachments/assets/75a6f694-c86f-41e1-a3c5-92fbc73290d9" /><br>

- 예측 결과 확인 <br>
```
  SELECT Date, next_day_close, prediction FROM test_predictions ORDER BY Date;
```
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/a88228b0-74c4-4a0d-9604-80a3bdfa9d2b" /><br>


### 7.	MySQL heatwave 예측 모델 평가
- neg_mean_squared_error (음의 평균 제곱 오차)<br>
    - 이 값은 평균 제곱 오차(MSE, Mean Squared Error)에 음수(-)를 붙인 것입니다.<br>
```
CALL sys.ML_SCORE('stockdb.stock_test_data', 'next_day_close',@stock_model, 'neg_mean_squared_error', @stock_score, NULL);
```
- 
<img width="700" height="288" alt="image" src="https://github.com/user-attachments/assets/b4d43270-7cf7-4f0c-b602-1d4d5c5b0b41" />

* 회귀(Regression) 문제용:

    * mean_squared_error (MSE): 평균 제곱 오차 (오차)

    * mean_absolute_error (MAE): 평균 절대 오차 (오차)

    * r2: R2 Score (설명력)

* 분류(Classification) 문제용:

    * accuracy: 정확도

    * f1: F1 Score

    * precision: 정밀도

    * recall: 재현율

