import dash
from dash import dcc, html
from dash.dependencies import Input, Output
import pandas as pd
import numpy as np
import plotly.graph_objs as go

# Sample historical data (replace with real data source if available)
data = pd.DataFrame({
    'Date': pd.date_range(start='2023-01-01', periods=100),
    'Close': np.random.normal(100, 5, 100).cumsum()
})
data.set_index('Date', inplace=True)

# Define Trading Strategies
def moving_average_strategy(data, short_window=40, long_window=100):
    data['Short_MA'] = data['Close'].rolling(window=short_window, min_periods=1).mean()
    data['Long_MA'] = data['Close'].rolling(window=long_window, min_periods=1).mean()
    data['Signal'] = 0
    data['Signal'][short_window:] = np.where(data['Short_MA'][short_window:] > data['Long_MA'][short_window:], 1, 0)
    data['Positions'] = data['Signal'].diff()
    return data

def rsi_strategy(data, window=14, rsi_buy=30, rsi_sell=70):
    delta = data['Close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=window).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=window).mean()
    rs = gain / loss
    data['RSI'] = 100 - (100 / (1 + rs))
    data['Signal'] = 0
    data['Signal'] = np.where(data['RSI'] < rsi_buy, 1, np.where(data['RSI'] > rsi_sell, -1, 0))
    data['Positions'] = data['Signal'].diff()
    return data

# Initialize the Dash app
app = dash.Dash(__name__)

# App Layout
app.layout = html.Div([
    html.H1("Trading Algorithm Simulator"),
    html.Label("Select Strategy:"),
    dcc.Dropdown(id='strategy-dropdown', options=[
        {'label': 'Moving Average Crossover', 'value': 'ma'},
        {'label': 'RSI Strategy', 'value': 'rsi'}
    ], value='ma'),

    html.Label("Parameter:"),
    dcc.Slider(id='param-slider', min=10, max=100, step=1, value=20),

    dcc.Graph(id='price-graph'),
    dcc.Graph(id='performance-graph')
])

# Update Graphs based on User Input
@app.callback(
    [Output('price-graph', 'figure'),
     Output('performance-graph', 'figure')],
    [Input('strategy-dropdown', 'value'),
     Input('param-slider', 'value')]
)
def update_graph(strategy, param):
    df = data.copy()
    if strategy == 'ma':
        df = moving_average_strategy(df, short_window=param, long_window=param*2)
    elif strategy == 'rsi':
        df = rsi_strategy(df, window=param)

    # Plot Price with Buy/Sell Signals
    price_fig = go.Figure()
    price_fig.add_trace(go.Scatter(x=df.index, y=df['Close'], mode='lines', name='Close Price'))
    
    if strategy == 'ma':
        price_fig.add_trace(go.Scatter(x=df.index, y=df['Short_MA'], mode='lines', name=f'Short MA ({param})'))
        price_fig.add_trace(go.Scatter(x=df.index, y=df['Long_MA'], mode='lines', name=f'Long MA ({param*2})'))
    elif strategy == 'rsi':
        price_fig.add_trace(go.Scatter(x=df.index, y=df['RSI'], mode='lines', name=f'RSI ({param})'))
    
    buy_signals = df[df['Positions'] == 1]
    sell_signals = df[df['Positions'] == -1]
    
    price_fig.add_trace(go.Scatter(x=buy_signals.index, y=buy_signals['Close'], mode='markers', marker=dict(color='green', size=10), name='Buy Signal'))
    price_fig.add_trace(go.Scatter(x=sell_signals.index, y=sell_signals['Close'], mode='markers', marker=dict(color='red', size=10), name='Sell Signal'))

    price_fig.update_layout(title="Price and Signals", xaxis_title="Date", yaxis_title="Price")

    # Performance Metrics Graph
    df['Return'] = df['Close'].pct_change()
    df['Cumulative Return'] = (1 + df['Return']).cumprod() - 1
    performance_fig = go.Figure()
    performance_fig.add_trace(go.Scatter(x=df.index, y=df['Cumulative Return'], mode='lines', name='Cumulative Return'))
    performance_fig.update_layout(title="Performance Metrics", xaxis_title="Date", yaxis_title="Cumulative Return")

    return price_fig, performance_fig

# Run the App
if __name__ == '__main__':
    app.run_server(debug=True)
