import streamlit as st
import pandas as pd
import yfinance as yf
from datetime import datetime, timedelta
import plotly.graph_objects as go

st.set_page_config(page_title="Educational Stock Holding Analyzer", layout="wide")
st.title("Educational Tax-Aware Stock Holding Analyzer")

st.warning(
    "⚠️ THIS IS **NOT** FINANCIAL, INVESTMENT, OR TAX ADVICE. "
    "Experimental educational tool only. General information and hypothetical scenarios. "
    "Always consult a licensed financial advisor, tax professional, or attorney. "
    "No personalized recommendations are made. Past performance ≠ future results."
)

# ─── Cached Data Fetch ───────────────────────────────────────────────────────
@st.cache_data(ttl=300)
def get_stock_data(ticker: str, period: str = "2y") -> tuple:
    ticker = ticker.upper()
    try:
        stock = yf.Ticker(ticker)
        current_price = stock.history(period="1d")["Close"].iloc[-1] if not stock.history(period="1d").empty else 0.0
        hist = stock.history(period=period)
        return current_price, hist
    except Exception as e:
        st.error(f"Error fetching data for {ticker}: {e}")
        return 0.0, pd.DataFrame()

# ─── Tax Context (2026 general illustration) ────────────────────────────────
def get_tax_context(days_held: int) -> str:
    years_held = days_held / 365.25
    holding_type = "Long-term" if days_held > 365 else "Short-term"
    return (
        f"**2026 Capital Gains Rules (single filer example – illustrative only):**\n\n"
        f"**Long-term** (> 1 year):\n"
        "• 0% up to ~$49,450 taxable income\n"
        "• 15% ~$49,451 – $545,500\n"
        "• 20% above $545,500\n\n"
        f"**Short-term** (≤ 1 year): taxed as ordinary income (up to 37%).\n\n"
        f"Your position: ~{years_held:.1f} years held ({holding_type}).\n"
        "Also consider: wash-sale rule, 3.8% NIIT (high earners), state taxes, etc."
    )

def tax_aware_advisor(ticker: str, buy_price: float, buy_date: datetime) -> dict:
    current_price, _ = get_stock_data(ticker, "1d")
    if current_price == 0:
        return {"price": None, "tax_context": "Data unavailable", "info": "No market data."}

    days_held = (datetime.today() - buy_date).days
    gain_pct = ((current_price - buy_price) / buy_price) * 100 if buy_price > 0 else 0

    return {
        "price": current_price,
        "tax_context": get_tax_context(days_held),
        "info": (
            f"**{ticker}** (current ≈ ${current_price:,.2f})\n"
            f"• Bought at ${buy_price:,.2f} → unrealized {gain_pct:+.1f}%\n"
            "Educational context only. No transaction recommendation."
        ),
        "days_held": days_held,
        "holding_type": "Long-term" if days_held > 365 else "Short-term"
    }

def compliance_judge(text: str) -> dict:
    forbidden = ["buy", "sell", "hold", "recommend", "should", "must", "target", "will rise", "will fall"]
    if any(word in text.lower() for word in forbidden):
        return {"status": "FLAGGED", "reason": "Possible recommendation-like wording detected."}
    return {"status": "EDUCATIONAL ONLY", "reason": "General info — no buy/sell/hold advice."}

# ─── SIDEBAR ────────────────────────────────────────────────────────────────
with st.sidebar:
    st.header("Holding Details")
    ticker = st.text_input("Ticker", value="AAPL").strip().upper()
    buy_price = st.number_input("Purchase Price ($)", min_value=0.01, value=150.00, step=0.01)
    default_date = datetime.today() - timedelta(days=730)
    buy_date = st.date_input("Purchase Date", value=default_date, max_value=datetime.today())
    buy_datetime = datetime.combine(buy_date, datetime.min.time())

    st.subheader("Chart Settings")
    period_options = {
        "1 Month": "1mo", "3 Months": "3mo", "6 Months": "6mo",
        "1 Year": "1y", "2 Years": "2y", "5 Years": "5y"
    }
    period_label = st.select_slider("Lookback Period", options=list(period_options), value="2 Years")
    period = period_options[period_label]

    chart_type = st.radio("Chart Type", ["Line", "Candlestick"], index=1)
    show_ma = st.checkbox("Show 20-day Moving Average", value=True)

    analyze = st.button("Analyze & Show Chart", type="primary")

# ─── MAIN LOGIC ─────────────────────────────────────────────────────────────
if analyze and ticker:
    with st.spinner("Loading market data..."):
        current_price, hist = get_stock_data(ticker, period)
        if hist.empty:
            st.error("No data retrieved. Check ticker / internet.")
        else:
            advice = tax_aware_advisor(ticker, buy_price, buy_datetime)
            comp = compliance_judge(advice["info"])

            st.session_state.update({
                "advice": advice,
                "compliance": comp,
                "history": hist,
                "period_label": period_label,
                "chart_type": chart_type,
                "show_ma": show_ma
            })

# ─── DISPLAY RESULTS ────────────────────────────────────────────────────────
if "advice" in st.session_state and st.session_state["advice"]["price"] is not None:
    adv = st.session_state["advice"]
    comp = st.session_state["compliance"]
    hist = st.session_state["history"]
    period_label = st.session_state["period_label"]
    chart_type = st.session_state["chart_type"]
    show_ma = st.session_state["show_ma"]

    # Metrics
    c1, c2, c3 = st.columns(3)
    c1.metric("Current Price", f"${adv['price']:,.2f}")
    c2.metric("Days Held", adv["days_held"], delta=adv["holding_type"])
    c3.markdown(
        f"**Compliance**  \n"
        f"<span style='color:{'green' if comp['status']=='EDUCATIONAL ONLY' else 'orange'};'>"
        f"{comp['status']}</span>", unsafe_allow_html=True
    )
    st.caption(comp["reason"])

    st.divider()

    # ─── CHART SECTION ──────────────────────────────────────────────────────
    st.subheader(f"{ticker} Price History – Last {period_label}")

    if chart_type == "Candlestick":
        fig = go.Figure()

        # Candlestick
        fig.add_trace(go.Candlestick(
            x=hist.index,
            open=hist['Open'], high=hist['High'], low=hist['Low'], close=hist['Close'],
            name='OHLC'
        ))

        # Moving Average (if enabled)
        if show_ma and len(hist) >= 20:
            ma20 = hist['Close'].rolling(window=20).mean()
            fig.add_trace(go.Scatter(
                x=hist.index, y=ma20, line=dict(color='orange', width=2),
                name='20-day MA'
            ))

        fig.update_layout(
            xaxis_title="Date",
            yaxis_title="Price ($)",
            xaxis_rangeslider_visible=True,
            template="plotly_dark" if st.get_option("theme") == "dark" else "plotly_white",
            height=500
        )
        st.plotly_chart(fig, use_container_width=True)

    else:  # Line chart fallback
        chart_data = hist[["Close"]].copy()
        if show_ma and len(hist) >= 20:
            chart_data["20-day MA"] = hist["Close"].rolling(20).mean()
        st.line_chart(chart_data, use_container_width=True)

    st.divider()

    # Tax & Info columns
    left, right = st.columns([2, 1])
    with left:
        st.subheader("Tax Context (2026 – General)")
        st.info(adv["tax_context"])
    with right:
        st.subheader("Educational Overview")
        st.write(adv["info"])

    st.caption("Data: yfinance (Yahoo Finance) • Tax brackets simplified/illustrative • Verify with professionals")
