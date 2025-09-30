# heatwaveMLstockprices
Predict a next day stock price using HeatWave AutoML

### 1.	학습 데이터 및 테스트 데이터 로딩
-  Python 프로그램을 이용해서 Yahoo Finance api 를 이용해서 주식 데이터를 가지고 옵니다. <br>

'''
# --- Yahoo Finance에서 데이터 다운로드 ---
print(f"'{TICKER_SYMBOL}' 주식 데이터 다운로드를 시작합니다...")
df = yf.download(TICKER_SYMBOL, start="2015-09-24", end="2025-09-29")

.. skip..

 # --- SQLAlchemy를 사용한 데이터베이스 연결 및 적재 ---
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


'''

