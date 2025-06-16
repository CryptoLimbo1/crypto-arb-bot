import requests
import time
import telebot

TOKEN = '7600143464:AAGfOtuk6xe2eHTuDCHRrzpMPfCB-a6NZNg'
CHAT_ID = '526447510'
SPREAD_LIMIT = 2.0  # %

bot = telebot.TeleBot(TOKEN)

def get_okx_symbols():
    try:
        resp = requests.get("https://www.okx.com/api/v5/market/tickers?instType=SPOT").json()
        return {item['instId'].replace('-USDT', '') for item in resp['data'] if item['instId'].endswith('USDT')}
    except:
        return set()

def get_mexc_symbols():
    try:
        resp = requests.get("https://www.mexc.com/open/api/v2/market/ticker").json()
        return {item['symbol'].replace('_USDT', '') for item in resp['data'] if item['symbol'].endswith('USDT')}
    except:
        return set()

def get_bybit_symbols():
    try:
        resp = requests.get("https://api.bybit.com/v5/market/tickers?category=spot").json()
        return {item['symbol'].replace('USDT', '') for item in resp['result']['list'] if item['symbol'].endswith('USDT')}
    except:
        return set()

def get_htx_symbols():
    try:
        resp = requests.get("https://api.huobi.pro/market/tickers").json()
        return {item['symbol'].replace('usdt', '').upper() for item in resp['data'] if item['symbol'].endswith('usdt')}
    except:
        return set()

def get_price_okx(symbol):
    try:
        resp = requests.get(f"https://www.okx.com/api/v5/market/ticker?instId={symbol}-USDT").json()
        return float(resp['data'][0]['last'])
    except:
        return None

def get_price_mexc(symbol):
    try:
        resp = requests.get(f"https://www.mexc.com/open/api/v2/market/ticker?symbol={symbol}_USDT").json()
        return float(resp['data'][0]['last'])
    except:
        return None

def get_price_bybit(symbol):
    try:
        resp = requests.get(f"https://api.bybit.com/v5/market/ticker/24hr?category=spot&symbol={symbol}USDT").json()
        return float(resp['result']['list'][0]['lastPrice'])
    except:
        return None

def get_price_htx(symbol):
    try:
        resp = requests.get(f"https://api.huobi.pro/market/detail/merged?symbol={symbol.lower()}usdt").json()
        return float(resp['tick']['close'])
    except:
        return None

def get_common_symbols():
    okx = get_okx_symbols()
    mexc = get_mexc_symbols()
    bybit = get_bybit_symbols()
    htx = get_htx_symbols()

    all_symbols = okx | mexc | bybit | htx
    filtered = set()
    for sym in all_symbols:
        count = sum(sym in exch for exch in [okx, mexc, bybit, htx])
        if count >= 2:
            filtered.add(sym)
    return sorted(filtered)

def check_arbitrage(symbols):
    exchanges = {
        "OKX": get_price_okx,
        "MEXC": get_price_mexc,
        "BYBIT": get_price_bybit,
        "HTX": get_price_htx,
    }
    for sym in symbols:
        prices = {}
        for exch, func in exchanges.items():
            price = func(sym)
            if price:
                prices[exch] = price
        if len(prices) < 2:
            continue

        exch_list = list(prices.items())
        for i in range(len(exch_list)):
            for j in range(i+1, len(exch_list)):
                exch1, p1 = exch_list[i]
                exch2, p2 = exch_list[j]
                spread = abs(p1 - p2) / min(p1, p2) * 100
                if spread >= SPREAD_LIMIT:
                    direction = f"{exch1} → {exch2}" if p1 > p2 else f"{exch2} → {exch1}"
                    msg = (
                        f"🚀 Арбитраж: {sym}\n"
                        f"{exch1}: {p1:.6f} USDT\n"
                        f"{exch2}: {p2:.6f} USDT\n"
                        f"📊 Спред: {spread:.2f}%\n"
                        f"➡️ Направление: {direction}"
                    )
                    bot.send_message(CHAT_ID, msg)
                    time.sleep(1)

if __name__ == "__main__":
    while True:
        symbols = get_common_symbols()
        print(f"🔍 Проверяем {len(symbols)} монет...")
        check_arbitrage(symbols)
        time.sleep(60)
