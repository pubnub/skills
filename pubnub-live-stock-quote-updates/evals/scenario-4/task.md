# Real-Time Stock Chart with Market Session Awareness

## Problem/Feature Description

A fintech company is building an interactive stock charting feature for their web trading platform. The chart should display live candlestick bars that update in real time as price data streams in via PubNub. Since the platform serves users across different time zones, the chart needs to be aware of US market trading sessions -- users should clearly see what session they are viewing data from and whether the market is currently open.

Incoming tick data arrives at varying frequencies and quality. Some ticks may have invalid prices (zero or negative), impossible bid/ask spreads, or very old timestamps from provider glitches. The charting module must validate data before using it. Additionally, during fast-moving markets the incoming data rate can be extremely high, so the chart rendering must be efficient and not overwhelm the browser.

Build a client-side JavaScript module that subscribes to a stock's quote channel, validates incoming data, aggregates ticks into OHLCV candlestick bars, tracks market session state, and exposes the data in a format ready for a charting library to render.

## Output Specification

Produce the following files:

- `stock-chart.js` -- A JavaScript module that:
  - Subscribes to PubNub for a given stock symbol and processes incoming quote/tick data
  - Validates incoming data before using it for charting
  - Aggregates incoming price ticks into time-based candlestick (OHLCV) bars
  - Determines the current US market trading session and whether trading is active
  - Manages rendering efficiency for high-frequency data
  - Tracks sparkline data points for mini price visualization

- `package.json` -- Dependencies needed

No additional files are required.
