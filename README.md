# Python-Developer-Task

Project overview
- Place market and limit orders (USDT-M Futures) on Binance Testnet.
- Support buy and sell.
- Command-line interface with input validation.
- Structured logging of requests, responses, and errors.
- Clean, reusable class-based design.
- Optional: Stop-market orders as a third type.


- Run the bot:
- Market order: python bot.py --symbol BTCUSDT --side BUY --type MARKET --quantity 0.001
- Limit order: python bot.py --symbol BTCUSDT --side SELL --type LIMIT --quantity 0.001 --price 70000 --timeInForce GTC
- Stop-market: python bot.py --symbol BTCUSDT --side SELL --type STOP_MARKET --quantity 0.001 --stopPrice 68000

File structure
- bot.py
- config.py
- utils.py
- requirements.txt
- README.md
- logs/
- bot.log


Code...

# config.py
import os

TESTNET_BASE_URL = "https://testnet.binancefuture.com"
LOG_DIR = os.getenv("LOG_DIR", "logs")
DEFAULT_TIME_IN_FORCE = "GTC"
REQUEST_TIMEOUT = 10  # seconds

# utils.py
import logging
import os
from logging.handlers import RotatingFileHandler

def setup_logger(log_dir: str, name: str = "bot") -> logging.Logger:
    os.makedirs(log_dir, exist_ok=True)
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)

    # File handler
    file_handler = RotatingFileHandler(
        os.path.join(log_dir, f"{name}.log"),
        maxBytes=2_000_000,
        backupCount=3,
        encoding="utf-8",
    )
    file_handler.setLevel(logging.INFO)
    file_formatter = logging.Formatter(
        "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
    )
    file_handler.setFormatter(file_formatter)

    # Console handler
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_formatter = logging.Formatter("%(levelname)s: %(message)s")
    console_handler.setFormatter(console_formatter)

    # Avoid duplicate handlers if setup_logger is called twice
    if not logger.handlers:
        logger.addHandler(file_handler)
        logger.addHandler(console_handler)

    logger.propagate = False
    return logger

    # bot.py
import argparse
import sys
import time
from typing import Optional, Dict, Any

from binance.um_futures import UMFutures
from binance.error import ClientError, ServerError

from config import TESTNET_BASE_URL, LOG_DIR, DEFAULT_TIME_IN_FORCE, REQUEST_TIMEOUT
from utils import setup_logger


class BasicBot:
    def __init__(self, api_key: str, api_secret: str, testnet: bool = True):
        self.logger = setup_logger(LOG_DIR, name="bot")
        base_url = TESTNET_BASE_URL if testnet else None
        self.client = UMFutures(
            api_key=api_key,
            api_secret=api_secret,
            base_url=base_url,
            timeout=REQUEST_TIMEOUT,
        )
        self.logger.info(f"Initialized BasicBot (testnet={testnet}) with base_url={base_url}")

    def get_exchange_info(self) -> Dict[str, Any]:
        try:
            info = self.client.exchange_info()
            self.logger.info("Fetched exchange info")
            return info
        except (ClientError, ServerError) as e:
            self.logger.error(f"Error fetching exchange info: {e}")
            raise

    def get_price(self, symbol: str) -> float:
        try:
            ticker = self.client.ticker_price(symbol=symbol)
            price = float(ticker["price"])
            self.logger.info(f"Fetched price for {symbol}: {price}")
            return price
        except (ClientError, ServerError, KeyError, ValueError) as e:
            self.logger.error(f"Failed fetching price for {symbol}: {e}")
            raise

    def _validate_inputs(
        self,
        symbol: str,
        side: str,
        order_type: str,
        quantity: float,
        price: Optional[float],
        time_in_force: Optional[str],
        stop_price: Optional[float],
    ):
        side = side.upper()
        order_type = order_type.upper()
        if side not in {"BUY", "SELL"}:
            raise ValueError("side must be BUY or SELL")
        if order_type not in {"MARKET", "LIMIT", "STOP_MARKET"}:
            raise ValueError("type must be MARKET, LIMIT, or STOP_MARKET")
        if quantity <= 0:
            raise ValueError("quantity must be positive")
        if order_type == "LIMIT":
            if price is None or price <= 0:
                raise ValueError("price must be provided and positive for LIMIT orders")
            if not time_in_force:
                raise ValueError("timeInForce is required for LIMIT orders (e.g., GTC)")
        if order_type == "STOP_MARKET":
            if stop_price is None or stop_price <= 0:
                raise ValueError("stopPrice must be provided and positive for STOP_MARKET orders")
        return side, order_type

    def place_order(
        self,
        symbol: str,
        side: str,
        order_type: str,
        quantity: float,
        price: Optional[float] = None,
        time_in_force: Optional[str] = DEFAULT_TIME_IN_FORCE,
        stop_price: Optional[float] = None,
        reduce_only: bool = False,
    ) -> Dict[str, Any]:
        """
        Places an order on USDT-M Futures (Testnet).
        Supported: MARKET, LIMIT, STOP_MARKET
        """
        side, order_type = self._validate_inputs(symbol, side, order_type, quantity, price, time_in_force, stop_price)

        params: Dict[str, Any] = {
            "symbol": symbol.upper(),
            "side": side,
            "type": order_type,
            "quantity": quantity,
            "reduceOnly": reduce_only,
            "newOrderRespType": "RESULT",  # get execution details if possible
        }

        if order_type == "LIMIT":
            params["price"] = price
            params["timeInForce"] = time_in_force
        if order_type == "STOP_MARKET":
            params["stopPrice"] = stop_price

        self.logger.info(f"Placing order: {params}")

        try:
            resp = self.client.new_order(**params)
            self.logger.info(f"Order response: {resp}")
            return resp
        except ClientError as e:
            self.logger.error(f"ClientError placing order: status={e.status_code} code={e.error_code} msg={e.error_message}")
            raise
        except ServerError as e:
            self.logger.error(f"ServerError placing order: {e}")
            raise
        except Exception as e:
            self.logger.error(f"Unexpected error placing order: {e}")
            raise

    def get_order_status(self, symbol: str, order_id: Optional[int] = None, orig_client_order_id: Optional[str] = None):
        try:
            resp = self.client.get_order(symbol=symbol.upper(), orderId=order_id, origClientOrderId=orig_client_order_id)
            self.logger.info(f"Fetched order status: {resp}")
            return resp
        except (ClientError, ServerError) as e:
            self.logger.error(f"Error fetching order status: {e}")
            raise

    def cancel_order(self, symbol: str, order_id: Optional[int] = None, orig_client_order_id: Optional[str] = None):
        try:
            resp = self.client.cancel_order(symbol=symbol.upper(), orderId=order_id, origClientOrderId=orig_client_order_id)
            self.logger.info(f"Canceled order: {resp}")
            return resp
        except (ClientError, ServerError) as e:
            self.logger.error(f"Error canceling order: {e}")
            raise


def parse_args():
    parser = argparse.ArgumentParser(description="Simplified Binance USDT-M Futures Testnet Trading Bot")
    parser.add_argument("--apiKey", required=True, help="Binance API key")
    parser.add_argument("--apiSecret", required=True, help="Binance API secret")
    parser.add_argument("--symbol", required=True, help="Trading symbol, e.g., BTCUSDT")
    parser.add_argument("--side", required=True, choices=["BUY", "SELL"], help="Order side")
    parser.add_argument("--type", required=True, choices=["MARKET", "LIMIT", "STOP_MARKET"], help="Order type")
    parser.add_argument("--quantity", required=True, type=float, help="Order quantity")
    parser.add_argument("--price", type=float, help="Limit order price")
    parser.add_argument("--timeInForce", default=DEFAULT_TIME_IN_FORCE, choices=["GTC", "IOC", "FOK"], help="Time in force (LIMIT only)")
    parser.add_argument("--stopPrice", type=float, help="Stop price (STOP_MARKET)")
    parser.add_argument("--reduceOnly", action="store_true", help="Set reduceOnly=true to only reduce positions")
    parser.add_argument("--checkStatus", action="store_true", help="Poll for order status after placement")
    parser.add_argument("--pollSeconds", type=int, default=2, help="Polling interval for status")
    parser.add_argument("--pollCount", type=int, default=5, help="Number of polls for status")
    parser.add_argument("--testnet", action="store_true", default=True, help="Use Testnet (default True)")
    return parser.parse_args()


def main():
    args = parse_args()

    bot = BasicBot(api_key=args.apiKey, api_secret=args.apiSecret, testnet=args.testnet)

    try:
        order_resp = bot.place_order(
            symbol=args.symbol,
            side=args.side,
            order_type=args.type,
            quantity=args.quantity,
            price=args.price,
            time_in_force=args.timeInForce,
            stop_price=args.stopPrice,
            reduce_only=args.reduceOnly,
        )
    except Exception as e:
        print(f"Order failed: {e}")
        sys.exit(1)

    # Output summary to console (also logged)
    summary = {
        "symbol": order_resp.get("symbol"),
        "orderId": order_resp.get("orderId"),
        "clientOrderId": order_resp.get("clientOrderId"),
        "status": order_resp.get("status"),
        "type": order_resp.get("type"),
        "side": order_resp.get("side"),
        "price": order_resp.get("price"),
        "avgPrice": order_resp.get("avgPrice"),
        "origQty": order_resp.get("origQty"),
        "executedQty": order_resp.get("executedQty"),
        "reduceOnly": order_resp.get("reduceOnly"),
        "updateTime": order_resp.get("updateTime"),
    }
    print("Order placed:")
    for k, v in summary.items():
        print(f"  {k}: {v}")

    if args.checkStatus:
        order_id = order_resp.get("orderId")
        if order_id:
            print("Polling order status...")
            for i in range(args.pollCount):
                time.sleep(args.pollSeconds)
                status = bot.get_order_status(symbol=args.symbol, order_id=order_id)
                print(f"  Poll {i+1}: status={status.get('status')} executedQty={status.get('executedQty')} avgPrice={status.get('avgPrice')}")
        else:
            print("No orderId returned to poll status.")

if __name__ == "__main__":
    main()

    
    # requirements.txt
python-binance==1.0.19

# README.md

## Binance USDT-M Futures Testnet Bot

### Prerequisites
- Create a Binance Futures Testnet account and generate API key/secret.
- Confirm trading on USDT-M Futures is enabled for the Testnet account.
- Python 3.10+; `pip install -r requirements.txt`.

### Usage examples
- Market buy:
  python bot.py --apiKey YOUR_KEY --apiSecret YOUR_SECRET --symbol BTCUSDT --side BUY --type MARKET --quantity 0.001 --checkStatus

- Limit sell:
  python bot.py --apiKey YOUR_KEY --apiSecret YOUR_SECRET --symbol BTCUSDT --side SELL --type LIMIT --quantity 0.001 --price 70000 --timeInForce GTC --checkStatus

- Stop-market (optional bonus):
  python bot.py --apiKey YOUR_KEY --apiSecret YOUR_SECRET --symbol BTCUSDT --side SELL --type STOP_MARKET --quantity 0.001 --stopPrice 68000

### Logs
- Logs are written to logs/bot.log and the console.
- Includes request parameters, responses, and error details.

### Notes
- Base URL is set to Binance Futures Testnet: https://testnet.binancefuture.com via UMFutures(base_url=...).
- This bot uses official Binance API via python-binance UMFutures client.
- Extend easily with TWAP/Grid strategies or a lightweight TUI/CLI UI if needed.

