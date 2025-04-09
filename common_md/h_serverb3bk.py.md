indi+tradeLog+resample = [h_serverb3bk1.py](h_serverb3bk1.py.md)

````python
from flask import Flask, render_template_string
from flask_sock import Sock
import json
from datetime import datetime, timedelta
from collections import deque
import pandas as pd
import threading
import os
import logging
import time
from pytz import timezone

###################################################### Flask Setup #######################################################
app = Flask(__name__)
sock = Sock(app)

###################################################### Global Data ########################################################
# Global deque to store tick data for processing
DEQUE_MAXLEN = 50
tick_data = deque(maxlen=DEQUE_MAXLEN)
candles_data = deque(maxlen=50000)  # Increased size to store more historical candles

# Global variables for managing the Fyers WebSocket thread and graceful shutdown.
ws_client = None  # Will hold the FyersDataSocket instance
ws_thread = None
# Global variable to store the active WebSocket connection
active_ws = None

###################################################### Logging Configuration ###############################################
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logging.info(f"Initial tick_data: {list(tick_data)}")


data_dir = '/var/lib/data'

# check if data_dir exists
if not os.path.exists(data_dir):
    print(f"Data directory {data_dir} does not exist.")
else:
    print(f"Data directory {data_dir} exists.")


def update_candle_with_tick(tick):
    """
    Processes an incoming tick to update the 5-second candle.
    If the tick falls within the current 5-second bucket, the candle is updated;
    otherwise, a new candle is created.
    After updating, sends the updated candle to the active WebSocket if connected.
    """
    global candles_data, active_ws
    
    # Convert tick's exchange feed time to an integer timestamp
    tick_dt = datetime.utcfromtimestamp(tick["exch_feed_time"]).replace(microsecond=0)
    tick_timestamp = int(tick_dt.timestamp())
    # Determine the 5-second bucket
    candle_time = tick_timestamp - (tick_timestamp % 5)
    
    if candles_data and candles_data[-1]['time'] == candle_time:
        # Update the existing candle
        last_candle = candles_data[-1]
        last_candle['high'] = max(last_candle['high'], tick["ltp"])
        last_candle['low'] = min(last_candle['low'], tick["ltp"])
        last_candle['close'] = tick["ltp"]
        updated_candle = last_candle
    else:
        # Create a new candle for a new 5-second bucket
        new_candle = {
            'time': candle_time,
            'open': tick["ltp"],
            'high': tick["ltp"],
            'low': tick["ltp"],
            'close': tick["ltp"]
        }
        candles_data.append(new_candle)
        updated_candle = new_candle

    # Send the updated candle to the active WebSocket if connected
    if active_ws:
        try:
            active_ws.send(json.dumps({'candle': updated_candle}, default=str))
        except Exception as e:
            logging.error(f"Error sending to WebSocket: {e}")
            active_ws = None

###################################################### WebSocket Client Setup (Fyers) #######################################
def ws_client_connect():
    """
    Connects to Fyers WebSocket and populates the tick_data deque.
    This function starts the connection and then runs indefinitely until externally closed.
    """
    append_counter = 0  # Local counter

    import requests, time, base64, struct, hmac
    from fyers_apiv3 import fyersModel
    from urllib.parse import urlparse, parse_qs 
    pin = '8894'
    
    class FyesApp:
        def __init__(self) -> None:
            self.__username = 'XP12325'
            self.__totp_key = 'Q2HC7F57FHMHPRT2VRLPRWA4ORWPK34E'
            self.__pin = '8894'
            self.__client_id = "1BE74FZNXA-100"
            self.__secret_key = "P485IP4X2O"
            self.__redirect_uri = 'http://127.0.0.1:8081'
            self.__access_token = None

        def enable_app(self):
            appSession = fyersModel.SessionModel(
                client_id=self.__client_id,
                redirect_uri=self.__redirect_uri,
                response_type='code',
                state='state',
                secret_key=self.__secret_key,
                grant_type='authorization_code'
            )
            return appSession.generate_authcode()

        def __totp(self, key, time_step=30, digits=6, digest="sha1"):
            key = base64.b32decode(key.upper() + "=" * ((8 - len(key)) % 8))
            counter = struct.pack(">Q", int(time.time() / time_step))
            mac = hmac.new(key, counter, digest).digest()
            offset = mac[-1] & 0x0F
            binary = struct.unpack(">L", mac[offset: offset + 4])[0] & 0x7FFFFFFF
            return str(binary)[-digits:].zfill(digits)

        def get_token(self, refresh=False):
            try:
                if self.__access_token is None and refresh:
                    logging.error("Access token is None and refresh is True")
                    return

                headers = {
                    "Accept": "application/json",
                    "Accept-Language": "en-US,en;q=0.9",
                    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36",
                }
                s = requests.Session()
                s.headers.update(headers)

                # Step 1: Send login OTP
                data1 = f'{{"fy_id":"{base64.b64encode(f"{self.__username}".encode()).decode()}","app_id":"2"}}'
                r1 = s.post("https://api-t2.fyers.in/vagator/v2/send_login_otp_v2", data=data1)
                logging.info(f"Step 1 Response: {r1.status_code} - {r1.text}")
                if r1.status_code != 200:
                    raise Exception(f"Failed to send OTP: {r1.text}")

                # Step 2: Verify OTP
                request_key = r1.json()["request_key"]
                totp_code = self.__totp(self.__totp_key)
                data2 = f'{{"request_key":"{request_key}","otp":{totp_code}}}'
                logging.info(f"TOTP Generated: {totp_code}")
                r2 = s.post("https://api-t2.fyers.in/vagator/v2/verify_otp", data=data2)
                logging.info(f"Step 2 Response: {r2.status_code} - {r2.text}")
                if r2.status_code != 200:
                    raise Exception(f"Failed to verify OTP: {r2.text}")

                request_key = r2.json()["request_key"]
                data3 = f'{{"request_key":"{request_key}","identity_type":"pin","identifier":"{base64.b64encode(f"{pin}".encode()).decode()}"}}'
                r3 = s.post("https://api-t2.fyers.in/vagator/v2/verify_pin_v2", data=data3)
                assert r3.status_code == 200, f"Error in r3:\n {r3.json()}"

                headers = {"authorization": f"Bearer {r3.json()['data']['access_token']}",
                           "content-type": "application/json; charset=UTF-8"}
                data4 = f'{{"fyers_id":"{self.__username}","app_id":"{self.__client_id[:-4]}","redirect_uri":"{self.__redirect_uri}","appType":"100","code_challenge":"","state":"abcdefg","scope":"","nonce":"","response_type":"code","create_cookie":true}}'
                r4 = s.post("https://api.fyers.in/api/v2/token", headers=headers, data=data4)
                assert r4.status_code == 308, f"Error in r4:\n {r4.json()}"

                parsed = urlparse(r4.json()["Url"])
                auth_code = parse_qs(parsed.query)["auth_code"][0]

                session = fyersModel.SessionModel(
                    client_id=self.__client_id,
                    secret_key=self.__secret_key,
                    redirect_uri=self.__redirect_uri,
                    response_type="code",
                    grant_type="authorization_code"
                )
                session.set_token(auth_code)
                response = session.generate_token()
                self.__access_token = response["access_token"]
                return self.__access_token
            except Exception as e:
                logging.error(f"Error in get_token: {str(e)}")
                raise

    # Get access token
    app_obj = FyesApp()
    access_token = app_obj.get_token()
    print(f'AcessTOKEN: {access_token}')

    client_id = "1BE74FZNXA-100"
    
    def get_hist():
        """
        Fetches historical candle data from Fyers API and populates the candles_data deque.
        """
        global candles_data
        fyers = fyersModel.FyersModel(client_id=client_id, is_async=False, token=access_token, log_path="./")
        
        current_date = datetime.now().strftime("%Y-%m-%d")
        # Calculate date range
        range_from = (datetime.now() - timedelta(days=2)).strftime("%Y-%m-%d")
        range_to = (datetime.now() - timedelta(days=2)).strftime("%Y-%m-%d")
        # range_to = datetime.now().strftime("%Y-%m-%d")

        data = {
            "symbol": "NSE:NIFTY50-INDEX",
            "resolution": "5S",
            "date_format": "1",
            "range_from": range_from,
            "range_to": range_to,
            "cont_flag": "1"
        }

        res = fyers.history(data=data)

        if "candles" in res:
            # Each candle is a list of 6 elements (time, open, high, low, close, volume)
            # We only want the first five, so we slice each candle accordingly.
            candles_sliced = [candle[:5] for candle in res['candles']]
            df = pd.DataFrame(candles_sliced, columns=['time', 'open', 'high', 'low', 'close'])

            # Convert time to integer timestamps if needed.
            df["time"] = df["time"].astype(int)

            # Convert historical timestamps from IST to UTC by subtracting 5.5 hours (19800 seconds)
            df["time"] = df["time"] - 19800

            # Append to candles_data deque
            for candle in df.to_dict(orient="records"):
                candles_data.append(candle)

            logging.info(f"Historical candles appended. Total count: {len(candles_data)}")
        else:
            logging.error("Failed to fetch historical data.")
    
    # Fetch historical data before starting real-time updates
    get_hist()
    
    def replay_ws(interval=1.0, date=None):
        """
        Replays tick data from a CSV file at the specified interval.
        
        Args:
            interval: Time between ticks in seconds (default: 1.0)
            date: Date to replay in format 'mmdd' (e.g., 'apr3'). If None, uses current date.
        """
        global candles_data
        
        # Determine which date to use
        ist = timezone('Asia/Kolkata')
        if date is None:
            date = datetime.now(ist).strftime('%b%d').lower()
        
        csv_filename = f'{date}.csv'
        csv_path = os.path.join(data_dir, csv_filename)
        
        if not os.path.exists(csv_path):
            logging.error(f"CSV file not found: {csv_path}")
            return
        
        try:
            # Load the CSV file
            df = pd.read_csv(csv_path, names=['timestamp', 'price'])
            
            # Convert timestamp strings to datetime objects
            df['timestamp'] = pd.to_datetime(df['timestamp'])
            
            # Ensure price is numeric
            df['price'] = pd.to_numeric(df['price'], errors='coerce')
            
            # Drop rows with NaN values
            df = df.dropna()
            
            logging.info(f"Loaded {len(df)} ticks from {csv_path}")
            
            # Process each tick
            for index, row in df.iterrows():
                # Create a tick object similar to what would come from Fyers WebSocket
                # The CSV has datetime strings, so we need to convert to Unix timestamps
                tick = {
                    "exch_feed_time": int(row['timestamp'].timestamp()),
                    "ltp": float(row['price'])
                }
                
                # Update candle with this tick
                update_candle_with_tick(tick)
                
                # Sleep for the specified interval
                time.sleep(interval)
                
                # Log progress occasionally
                if index % 100 == 0:
                    logging.info(f"Processed {index} ticks")
                
        except Exception as e:
            logging.error(f"Error replaying CSV: {e}")
    
    # Start the replay with a specific date (e.g., 'apr3') or use None for today
    replay_ws(interval=0.1, date='apr03')  # Replay at 10x speed

###################################################### Flask WebSocket Server #######################################################
@sock.route("/ws")
def ws_endpoint(ws):
    """
    Sets this connection as the active WebSocket and keeps it open.
    The active connection will receive candle updates directly from update_candle_with_tick.
    """
    global active_ws
    active_ws = ws
    try:
        # Keep the connection open
        while True:
            ws.receive()  # This call blocks until a message is received (or connection is closed)
    except Exception as e:
        logging.error(f"WebSocket error: {e}")
    finally:
        if active_ws == ws:
            active_ws = None


###################################################### Flask Routes #######################################################
@app.route("/historic")
def get_historic_candles():
    """
    Returns all historical candles stored in memory.
    This endpoint is called when the chart is first loaded or refreshed.
    """
    return json.dumps(list(candles_data), default=str)

@app.route("/")
def index():
    """Serves the frontend chart."""
    html = """
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Live NIFTY Tick Chart (IST)</title>
        <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
    </head>
    <body>
        <h1>Live NIFTY Tick Chart (IST)</h1>
        <div id="chart"></div>
<script>
   const chart = LightweightCharts.createChart(document.getElementById('chart'), {
    width: window.innerWidth,
    height: window.innerHeight,
    priceScale: { borderColor: '#cccccc' },
    timeScale: { 
        borderColor: '#cccccc', 
        timeVisible: true, 
        secondsVisible: true,
        tickMarkFormatter: (time) => {
            const utcDate = new Date(time * 1000); // Convert UNIX time to Date object (UTC)
            const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); // Convert to IST
            return istDate.toLocaleTimeString('en-IN');
        }
    },
    localization: {
        timeFormatter: (time) => {
            const utcDate = new Date(time * 1000);
            const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); 
            return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');
        }
    }
});

   const candleSeries = chart.addCandlestickSeries();
   
   // First, fetch historical data
   fetch('/historic')
     .then(response => response.json())
     .then(candles => {
       // Set the initial data
       candleSeries.setData(candles);
       
       // Then connect to WebSocket for real-time updates
       const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");
       
       ws.onmessage = function(event) {
           const data = JSON.parse(event.data);
           if (data.candle) {
               // We're receiving a candle directly from the server
               candleSeries.update(data.candle);
           }
       };
     })
     .catch(error => {
       console.error('Error fetching historical data:', error);
     });
</script>
    </body>
    </html>
    """
    return render_template_string(html)


    
###################################################### Main Flow #######################################################
def main():
    """Starts the WebSocket client thread."""
    # create_table_if_not_exists()
    # Start ws_client_connect in a separate thread.
    global ws_thread
    ws_thread = threading.Thread(target=ws_client_connect, daemon=True)
    ws_thread.start()
    logging.info("Fyers WebSocket thread started.")



main()



port = int(os.getenv('PORT', 80))
print('Listening on port %s' % (port))
app.run(debug=False, host="0.0.0.0", port=port)
````
