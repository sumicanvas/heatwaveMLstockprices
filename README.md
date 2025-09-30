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
<img width="1168" height="524" alt="image" src="https://github.com/user-attachments/assets/ab79946e-46de-4baf-97dd-f3157db01099" />
<br>
<img width="1468" height="726" alt="image" src="https://github.com/user-attachments/assets/39f0a8d4-57cf-48f6-ad2b-0126f63e37eb" />
<br>

- next_day_close 컬럼 추가 (실제 다음날값으로 설정하고 예측값과 이후 비교용)
```
ALTER TABLE stock_prices ADD COLUMN next_day_close DECIMAL(10, 2);
```
<br>
- next_day_close 값 만들어서 추가해줘야 함

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
