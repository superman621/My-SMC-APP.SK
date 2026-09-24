import streamlit as st
import pandas as pd
import numpy as np
import pyotp
import time
from SmartApi import SmartConnect

st.set_page_config(page_title="SMC Pro Swing App", layout="wide")

st.title("⚡ SMC Pro Swing - Angel One Engine")

# ───────── SIDEBAR: CREDENTIALS & INPUTS ─────────
with st.sidebar:
    st.header("SmartAPI Login")
    api_key = st.text_input("API Key", type="password")
    client_code = st.text_input("Client ID")
    pwd = st.text_input("PIN / Password", type="password")
    totp_secret = st.text_input("TOTP Secret Key", type="password")
    
    st.divider()
    st.header("Strategy Settings")
    symbol = st.text_input("Trading Symbol", value="SBIN-EQ")
    token = st.text_input("Instrument Token", value="3045")
    exchange = st.selectbox("Exchange", ["NSE", "BSE", "NFO"])
    timeframe = st.selectbox("Timeframe", ["ONE_MINUTE", "FIVE_MINUTE", "FIFTEEN_MINUTE", "ONE_HOUR"], index=2)
    
    swing_len = st.number_input("Swing Length", value=5)
    atr_mult = st.number_input("SL ATR Multiplier", value=1.5)
    rr1 = st.number_input("TP1 R:R", value=2.0)
    rr2 = st.number_input("TP2 R:R", value=3.0)

if "bot_running" not in st.session_state:
    st.session_state.bot_running = False
if "trade_log" not in st.session_state:
    st.session_state.trade_log = []

# ───────── NATIVE TECHNICAL INDICATORS ─────────
def calculate_indicators(df):
    # EMA
    df["ema20"] = df["close"].ewm(span=20, adjust=False).mean()
    df["ema50"] = df["close"].ewm(span=50, adjust=False).mean()
    
    # RSI
    delta = df["close"].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
    rs = gain / loss
    df["rsi"] = 100 - (100 / (1 + rs))
    
    # ATR
    high_low = df["high"] - df["low"]
    high_close = (df["high"] - df["close"].shift()).abs()
    low_close = (df["low"] - df["close"].shift()).abs()
    ranges = pd.concat([high_low, high_close, low_close], axis=1)
    true_range = ranges.max(axis=1)
    df["atr"] = true_range.rolling(14).mean()
    
    # Average Volume
    df["avg_vol"] = df["volume"].rolling(window=20).mean()
    return df

# ───────── SMC LOGIC ENGINE ─────────
def calculate_smc(df, swing_len=5, sl_atr=1.5, rr1=2.0, rr2=3.0):
    df = calculate_indicators(df)

    bull_trend = (df["close"] > df["ema20"]) & (df["ema20"] > df["ema50"])
    bear_trend = (df["close"] < df["ema20"]) & (df["ema20"] < df["ema50"])

    # Swings
    df["swing_high"] = df["high"].rolling(window=swing_len*2+1, center=True).apply(lambda x: x[swing_len] if max(x) == x[swing_len] else np.nan)
    df["swing_low"] = df["low"].rolling(window=swing_len*2+1, center=True).apply(lambda x: x[swing_len] if min(x) == x[swing_len] else np.nan)
    df["last_high"] = df["swing_high"].ffill()
    df["last_low"] = df["swing_low"].ffill()

    # Liquidity Sweep & BOS
    bull_sweep = (df["low"] < df["last_low"]) & (df["close"] > df["last_low"])
    bear_sweep = (df["high"] > df["last_high"]) & (df["close"] < df["last_high"])
    bull_bos = df["close"] > df["last_high"]
    bear_bos = df["close"] < df["last_low"]

    bull_choch = bull_sweep | bull_bos
    bear_choch = bear_sweep | bear_bos

    # FVG
    bull_fvg = df["low"] > df["high"].shift(2)
    bear_fvg = df["high"] < df["low"].shift(2)

    # Filters
    vol_ok = df["volume"] > (df["avg_vol"] * 1.2)
    bull_rsi = (df["rsi"] >= 50) & (df["rsi"] <= 75)
    bear_rsi = (df["rsi"] <= 50) & (df["rsi"] >= 25)

    df["buy_signal"] = bull_trend & bull_choch & bull_fvg & bull_rsi & vol_ok
    df["sell_signal"] = bear_trend & bear_choch & bear_fvg & bear_rsi & vol_ok

    return df

# ───────── CONTROLS ─────────
col1, col2 = st.columns(2)
with col1:
    if st.button("🟢 Start SMC Bot", use_container_width=True):
        if not api_key or not client_code or not totp_secret:
            st.error("Kripya pehle saari API details bharein!")
        else:
            st.session_state.bot_running = True
            st.success("Bot Successfully Started!")

with col2:
    if st.button("🔴 Stop SMC Bot", use_container_width=True):
        st.session_state.bot_running = False
        st.warning("Bot Stopped.")

# ───────── LIVE SCANNING ─────────
status_placeholder = st.empty()

if st.session_state.bot_running:
    try:
        totp = pyotp.TOTP(totp_secret).now()
        smartApi = SmartConnect(api_key=api_key)
        login_data = smartApi.generateSession(client_code, pwd, totp)
        
        status_placeholder.info(f"Connected to Angel One | Scanning {symbol}...")

        to_date = pd.Timestamp.now().strftime("%Y-%m-%d %H:%M")
        from_date = (pd.Timestamp.now() - pd.Timedelta(days=5)).strftime("%Y-%m-%d %H:%M")

        historic_param = {
            "exchange": exchange,
            "symboltoken": token,
            "interval": timeframe,
            "fromdate": from_date,
            "todate": to_date
        }
        candle_data = smartApi.getCandleData(historic_param)
        
        if candle_data["status"] and candle_data["data"]:
            df = pd.DataFrame(candle_data["data"], columns=["timestamp", "open", "high", "low", "close", "volume"])
            df = calculate_smc(df, swing_len, atr_mult, rr1, rr2)
            
            latest = df.iloc[-1]
            prev = df.iloc[-2]

            st.metric("Latest Close", latest["close"], delta=f"RSI: {round(latest['rsi'], 2)}")

            if prev["buy_signal"]:
                sl = prev["close"] - (prev["atr"] * atr_mult)
                tp1 = prev["close"] + ((prev["atr"] * atr_mult) * rr1)
                st.session_state.trade_log.append({"Type": "BUY", "Price": prev["close"], "SL": sl, "TP1": tp1, "Time": prev["timestamp"]})
                st.toast(f"🚨 SMC BUY Signal Triggered at {prev['close']}!")

            elif prev["sell_signal"]:
                sl = prev["close"] + (prev["atr"] * atr_mult)
                tp1 = prev["close"] - ((prev["atr"] * atr_mult) * rr1)
                st.session_state.trade_log.append({"Type": "SELL", "Price": prev["close"], "SL": sl, "TP1": tp1, "Time": prev["timestamp"]})
                st.toast(f"🚨 SMC SELL Signal Triggered at {prev['close']}!")

            st.subheader("Live Market Data (Last 5 Candles)")
            st.dataframe(df[["timestamp", "close", "ema20", "ema50", "rsi", "volume"]].tail(5), use_container_width=True)

    except Exception as e:
        status_placeholder.error(f"Error: {e}")

st.divider()
st.subheader("Generated Signals / Trade Logs")
if st.session_state.trade_log:
    st.table(pd.DataFrame(st.session_state.trade_log))
else:
    st.write("Abhi tak koi trade trigger nahi hua hai.")
    
