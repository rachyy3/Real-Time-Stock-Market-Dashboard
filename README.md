# Real-Time-Stock-Market-Dashboard
A dashboard that tracks and visualizes live stock market data.

import streamlit as st
import pandas as pd
import plotly.graph_objects as go
import requests
from datetime import datetime, timedelta
import time

# Configuration
API_KEY = "YOUR_ALPHA_VANTAGE_API_KEY"  # Get free key from https://www.alphavantage.co
DEFAULT_SYMBOLS = ["AAPL", "MSFT", "GOOGL", "AMZN", "META"]
REFRESH_INTERVAL = 60  # seconds

# Set up Streamlit page
st.set_page_config(
    page_title="Real-Time Stock Dashboard",
    page_icon="📈",
    layout="wide"
)

# Custom CSS for better styling
st.markdown("""
<style>
    .main {
        background-color: #f8f9fa;
    }
    .stock-card {
        border-radius: 10px;
        padding: 15px;
        margin-bottom: 15px;
        background-color: white;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
    .positive {
        color: #28a745;
    }
    .negative {
        color: #dc3545;
    }
    .header {
        color: #2c3e50;
    }
</style>
""", unsafe_allow_html=True)

def get_stock_data(symbol, function="TIME_SERIES_INTRADAY", interval="5min"):
    """Fetch stock data from Alpha Vantage API"""
    try:
        if function == "TIME_SERIES_INTRADAY":
            url = f"https://www.alphavantage.co/query?function={function}&symbol={symbol}&interval={interval}&apikey={API_KEY}"
        else:
            url = f"https://www.alphavantage.co/query?function={function}&symbol={symbol}&apikey={API_KEY}"
        
        response = requests.get(url)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        st.error(f"Error fetching data for {symbol}: {str(e)}")
        return None

def process_intraday_data(data):
    """Process intraday time series data"""
    time_series = data.get("Time Series (5min)", {})
    df = pd.DataFrame.from_dict(time_series, orient="index")
    df.index = pd.to_datetime(df.index)
    df.columns = [col.split(" ")[1] for col in df.columns]
    df = df.astype(float)
    return df.sort_index()

def process_overview_data(data):
    """Process company overview data"""
    return {
        "Name": data.get("Name"),
        "Sector": data.get("Sector"),
        "Industry": data.get("Industry"),
        "MarketCap": data.get("MarketCapitalization"),
        "PERatio": data.get("PERatio"),
        "DividendYield": data.get("DividendYield"),
        "52WeekHigh": data.get("52WeekHigh"),
        "52WeekLow": data.get("52WeekLow")
    }

def format_currency(value):
    """Format numeric values as currency"""
    if pd.isna(value):
        return "N/A"
    return f"${float(value):,.2f}"

def format_percent(value):
    """Format numeric values as percentage"""
    if pd.isna(value):
        return "N/A"
    return f"{float(value):.2f}%"

def create_candlestick_chart(df, symbol):
    """Create interactive candlestick chart"""
    fig = go.Figure(data=[
        go.Candlestick(
            x=df.index,
            open=df['open'],
            high=df['high'],
            low=df['low'],
            close=df['close'],
            name='Price'
        )
    ])
    
    fig.update_layout(
        title=f"{symbol} Intraday Price (5min intervals)",
        xaxis_title="Time",
        yaxis_title="Price (USD)",
        xaxis_rangeslider_visible=False,
        height=500,
        template="plotly_white"
    )
    
    return fig

def display_stock_card(symbol, overview_data, price_data):
    """Display a stock information card"""
    latest = price_data.iloc[-1]
    prev_close = price_data.iloc[-2]['close'] if len(price_data) > 1 else latest['close']
    change = latest['close'] - prev_close
    percent_change = (change / prev_close) * 100
    
    change_class = "positive" if change >= 0 else "negative"
    change_icon = "↑" if change >= 0 else "↓"
    
    with st.container():
        st.markdown(f"""
        <div class="stock-card">
            <div style="display: flex; justify-content: space-between; align-items: center;">
                <h2 class="header">{symbol} - {overview_data.get('Name', 'N/A')}</h2>
                <div style="text-align: right;">
                    <h2>{format_currency(latest['close'])}</h2>
                    <span class="{change_class}">
                        {change_icon} {format_currency(abs(change))} ({format_percent(abs(percent_change))})
                    </span>
                </div>
            </div>
            <div style="display: flex; justify-content: space-between; margin-top: 10px;">
                <div>
                    <p><strong>Sector:</strong> {overview_data.get('Sector', 'N/A')}</p>
                    <p><strong>Industry:</strong> {overview_data.get('Industry', 'N/A')}</p>
                </div>
                <div>
                    <p><strong>Market Cap:</strong> {format_currency(overview_data.get('MarketCap'))}</p>
                    <p><strong>P/E Ratio:</strong> {overview_data.get('PERatio', 'N/A')}</p>
                </div>
                <div>
                    <p><strong>52W High:</strong> {format_currency(overview_data.get('52WeekHigh'))}</p>
                    <p><strong>52W Low:</strong> {format_currency(overview_data.get('52WeekLow'))}</p>
                </div>
            </div>
        </div>
        """, unsafe_allow_html=True)
        
        # Display the chart
        st.plotly_chart(create_candlestick_chart(price_data, symbol), use_container_width=True)

def main():
    st.title("📈 Real-Time Stock Market Dashboard")
    st.markdown("Track live stock prices and key financial metrics")
    
    # Sidebar controls
    with st.sidebar:
        st.header("Dashboard Controls")
        symbols = st.text_input(
            "Stock Symbols (comma separated)",
            value=", ".join(DEFAULT_SYMBOLS)
        ).upper().split(", ")
        
        auto_refresh = st.checkbox("Auto-refresh", value=True)
        if auto_refresh:
            refresh_rate = st.slider("Refresh rate (seconds)", 10, 300, REFRESH_INTERVAL)
        
        st.markdown("---")
        st.markdown("**Data Source:** [Alpha Vantage](https://www.alphavantage.co)")
        st.markdown("**Note:** Free API tier has 5 requests/minute and 500 requests/day limits")
    
    # Placeholder for data refresh indicator
    last_refresh = st.empty()
    
    # Main dashboard area
    while True:
        # Get current time for refresh indicator
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        last_refresh.markdown(f"Last updated: {current_time}")
        
        # Fetch and display data for each symbol
        for symbol in symbols:
            symbol = symbol.strip()
            if not symbol:
                continue
                
            try:
                # Get intraday price data
                price_data = get_stock_data(symbol)
                if not price_data or "Time Series (5min)" not in price_data:
                    st.warning(f"No intraday data available for {symbol}")
                    continue
                
                processed_price_data = process_intraday_data(price_data)
                
                # Get company overview data
                overview_data = get_stock_data(symbol, function="OVERVIEW")
                processed_overview_data = process_overview_data(overview_data) if overview_data else {}
                
                # Display the stock card
                display_stock_card(symbol, processed_overview_data, processed_price_data)
                
            except Exception as e:
                st.error(f"Error processing {symbol}: {str(e)}")
        
        # Break the loop if auto-refresh is disabled
        if not auto_refresh:
            break
            
        # Wait for the specified refresh interval
        time.sleep(refresh_rate if auto_refresh else 1)
        
        # Clear the previous output for refresh
        st.experimental_rerun()

if __name__ == "__main__":
    main()
