# TradingView Data API with WebSocket Support

A real-time market data API that provides both REST endpoints and WebSocket connections for streaming market data from TradingView.

## Table of Contents
- [Backend Deployment (VPS)](#backend-deployment-vps)
- [Frontend Integration (React)](#frontend-integration-react)
- [API Documentation](#api-documentation)
- [WebSocket Documentation](#websocket-documentation)
- [Troubleshooting](#troubleshooting)

## Backend Deployment (VPS)

### Prerequisites
- VPS with Ubuntu/Debian
- Python 3.8 or higher
- Nginx
- Domain name (optional but recommended)

### Step 1: Initial Server Setup
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required system packages
sudo apt install python3 python3-pip nginx supervisor -y

# Create project directory
mkdir -p ~/tvdata_api
cd ~/tvdata_api
```

### Step 2: Project Setup
```bash
# Copy project files to VPS
# From your local machine:
scp scrpt.py requirements.txt your_username@your_vps_ip:~/tvdata_api/

# Install Python dependencies
pip3 install -r requirements.txt

# Test the application
python3 scrpt.py
```

### Step 3: Configure Supervisor
Create supervisor config:
```bash
sudo nano /etc/supervisor/conf.d/tvdata_api.conf
```

Add the following configuration:
```ini
[program:tvdata_api]
directory=/home/your_username/tvdata_api
command=gunicorn scrpt:app -w 4 -k uvicorn.workers.UvicornWorker -b 127.0.0.1:8080
autostart=true
autorestart=true
stderr_logfile=/var/log/tvdata_api.err.log
stdout_logfile=/var/log/tvdata_api.out.log
user=your_username
```

Start the service:
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start tvdata_api
```

### Step 4: Configure Nginx
Create Nginx config:
```bash
sudo nano /etc/nginx/sites-available/tvdata_api
```

Add the following configuration:
```nginx
server {
    listen 80;
    server_name your_domain.com;  # Replace with your domain or IP

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/tvdata_api /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Frontend Integration (React)

### Step 1: Create React Components

1. Install required dependencies:
```bash
npm install axios websocket
```

2. Create API service (`src/services/tvDataService.js`):
```javascript
import axios from 'axios';

const API_BASE_URL = 'http://your_vps_ip_or_domain';

export class TvDataService {
    // REST API calls
    static async fetchMarketData(symbol, exchange, interval = 'in_1_minute', n_bars = 100) {
        try {
            const response = await axios.get(`${API_BASE_URL}/fetch_data`, {
                params: { symbol, exchange, interval, n_bars }
            });
            return response.data;
        } catch (error) {
            console.error('Error fetching market data:', error);
            throw error;
        }
    }

    // WebSocket connection
    static createWebSocket(onMessage, onError) {
        const ws = new WebSocket(`ws://${API_BASE_URL}/ws`);
        
        ws.onopen = () => {
            console.log('WebSocket Connected');
        };

        ws.onmessage = onMessage;
        ws.onerror = onError;
        ws.onclose = () => {
            console.log('WebSocket Disconnected');
            // Implement reconnection logic
            setTimeout(() => this.createWebSocket(onMessage, onError), 5000);
        };

        return ws;
    }

    static subscribeToSymbol(ws, symbol, exchange, interval = 'in_1_minute', updateInterval = 60) {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({
                action: 'subscribe',
                symbol,
                exchange,
                interval,
                update_interval: updateInterval
            }));
        }
    }

    static unsubscribeFromSymbol(ws, symbol, exchange) {
        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({
                action: 'unsubscribe',
                symbol,
                exchange
            }));
        }
    }
}
```

3. Create Market Data Component (`src/components/MarketData.js`):
```javascript
import React, { useEffect, useState, useRef } from 'react';
import { TvDataService } from '../services/tvDataService';

export const MarketData = ({ symbol, exchange, interval }) => {
    const [marketData, setMarketData] = useState(null);
    const [error, setError] = useState(null);
    const wsRef = useRef(null);

    useEffect(() => {
        // Handle incoming WebSocket messages
        const handleMessage = (event) => {
            try {
                const data = JSON.parse(event.data);
                if (data.error) {
                    setError(data.error);
                } else {
                    setMarketData(data);
                }
            } catch (e) {
                setError('Failed to parse market data');
            }
        };

        // Handle WebSocket errors
        const handleError = (error) => {
            setError('WebSocket error: ' + error.message);
        };

        // Create WebSocket connection
        wsRef.current = TvDataService.createWebSocket(handleMessage, handleError);

        // Subscribe to symbol
        TvDataService.subscribeToSymbol(wsRef.current, symbol, exchange, interval);

        // Cleanup on unmount
        return () => {
            if (wsRef.current) {
                TvDataService.unsubscribeFromSymbol(wsRef.current, symbol, exchange);
                wsRef.current.close();
            }
        };
    }, [symbol, exchange, interval]);

    if (error) return <div>Error: {error}</div>;
    if (!marketData) return <div>Loading...</div>;

    return (
        <div className="market-data">
            <h2>{symbol} - {exchange}</h2>
            <div className="price-info">
                <p>Open: {marketData.Open}</p>
                <p>High: {marketData.High}</p>
                <p>Low: {marketData.Low}</p>
                <p>Close: {marketData.Close}</p>
                <p>Volume: {marketData.Volume}</p>
                <p>Last Updated: {new Date(marketData.datetime).toLocaleString()}</p>
            </div>
        </div>
    );
};
```

4. Use the Component:
```javascript
import { MarketData } from './components/MarketData';

function App() {
    return (
        <div className="App">
            <MarketData 
                symbol="AAPL" 
                exchange="NASDAQ" 
                interval="in_1_minute" 
            />
            <MarketData 
                symbol="GOOGL" 
                exchange="NASDAQ" 
                interval="in_5_minute" 
            />
        </div>
    );
}
```

### Step 2: Deploy React App to Shared Hosting

1. Build the React app:
```bash
npm run build
```

2. Upload the contents of the `build` folder to your shared hosting via FTP/SFTP

3. Configure your shared hosting:
   - Point your domain/subdomain to the uploaded directory
   - Enable CORS if needed
   - Set up SSL if available

## API Documentation

### REST Endpoints

#### GET /fetch_data
Fetches historical market data.

Parameters:
- `symbol` (required): Trading symbol
- `exchange` (required): Exchange name
- `interval` (optional): Time interval (default: "in_daily")
- `n_bars` (optional): Number of bars to fetch (default: 5000)
- `fut_contract` (optional): Futures contract ID

### WebSocket API

Connect to `/ws` endpoint for real-time data.

Message Format:
```javascript
// Subscribe
{
    "action": "subscribe",
    "symbol": "AAPL",
    "exchange": "NASDAQ",
    "interval": "in_1_minute",
    "update_interval": 60
}

// Unsubscribe
{
    "action": "unsubscribe",
    "symbol": "AAPL",
    "exchange": "NASDAQ"
}
```

## Troubleshooting

### Common Issues

1. WebSocket Connection Failed
   - Check if the VPS firewall allows WebSocket connections
   - Verify Nginx configuration
   - Check if the API is running (`sudo supervisorctl status tvdata_api`)

2. CORS Errors
   - Verify CORS settings in the API
   - Check if your domain is properly configured

3. Data Not Updating
   - Check the API logs: `sudo tail -f /var/log/tvdata_api.err.log`
   - Verify TradingView connectivity

For additional support or issues, please create a GitHub issue or contact support.

## Exchange and Symbol Examples

### Common Exchanges and Symbols

1. **US Stock Market (NASDAQ)**
```javascript
// Major Tech Stocks
const techStocks = [
    { symbol: "AAPL", exchange: "NASDAQ", name: "Apple Inc." },
    { symbol: "MSFT", exchange: "NASDAQ", name: "Microsoft Corporation" },
    { symbol: "GOOGL", exchange: "NASDAQ", name: "Alphabet Inc." },
    { symbol: "META", exchange: "NASDAQ", name: "Meta Platforms Inc." }
];

// Example usage with 1-minute interval
<MarketData symbol="AAPL" exchange="NASDAQ" interval="in_1_minute" />
```

2. **Cryptocurrency (BINANCE)**
```javascript
// Major Crypto Pairs
const cryptoPairs = [
    { symbol: "BTCUSDT", exchange: "BINANCE", name: "Bitcoin/USDT" },
    { symbol: "ETHUSDT", exchange: "BINANCE", name: "Ethereum/USDT" },
    { symbol: "BNBUSDT", exchange: "BINANCE", name: "Binance Coin/USDT" }
];

// Example usage with 5-minute interval
<MarketData symbol="BTCUSDT" exchange="BINANCE" interval="in_5_minute" />
```

3. **Indian Stock Market (NSE)**
```javascript
// Major Indian Stocks
const indianStocks = [
    { symbol: "RELIANCE", exchange: "NSE", name: "Reliance Industries" },
    { symbol: "TCS", exchange: "NSE", name: "Tata Consultancy Services" },
    { symbol: "INFY", exchange: "NSE", name: "Infosys Limited" }
];

// Example usage with daily interval
<MarketData symbol="RELIANCE" exchange="NSE" interval="in_daily" />
```

4. **Forex (FX)**
```javascript
// Major Currency Pairs
const forexPairs = [
    { symbol: "EURUSD", exchange: "FX", name: "Euro/US Dollar" },
    { symbol: "GBPUSD", exchange: "FX", name: "British Pound/US Dollar" },
    { symbol: "USDJPY", exchange: "FX", name: "US Dollar/Japanese Yen" }
];

// Example usage with 15-minute interval
<MarketData symbol="EURUSD" exchange="FX" interval="in_15_minute" />
```

5. **Commodities (MCX - Multi Commodity Exchange)**
```javascript
// Common Commodities
const commodities = [
    { symbol: "GOLD", exchange: "MCX", name: "Gold Futures" },
    { symbol: "SILVER", exchange: "MCX", name: "Silver Futures" },
    { symbol: "CRUDE", exchange: "MCX", name: "Crude Oil Futures" }
];

// Example usage with hourly interval
<MarketData symbol="GOLD" exchange="MCX" interval="in_1_hour" />
```

### Multiple Symbols Dashboard Example
```javascript
import React from 'react';
import { MarketData } from './components/MarketData';

const Dashboard = () => {
    return (
        <div className="dashboard">
            {/* US Stocks Section */}
            <div className="market-section">
                <h2>US Stocks</h2>
                <div className="grid">
                    <MarketData symbol="AAPL" exchange="NASDAQ" interval="in_1_minute" />
                    <MarketData symbol="MSFT" exchange="NASDAQ" interval="in_1_minute" />
                </div>
            </div>

            {/* Crypto Section */}
            <div className="market-section">
                <h2>Cryptocurrency</h2>
                <div className="grid">
                    <MarketData symbol="BTCUSDT" exchange="BINANCE" interval="in_5_minute" />
                    <MarketData symbol="ETHUSDT" exchange="BINANCE" interval="in_5_minute" />
                </div>
            </div>

            {/* Forex Section */}
            <div className="market-section">
                <h2>Forex</h2>
                <div className="grid">
                    <MarketData symbol="EURUSD" exchange="FX" interval="in_15_minute" />
                    <MarketData symbol="GBPUSD" exchange="FX" interval="in_15_minute" />
                </div>
            </div>
        </div>
    );
};

export default Dashboard;
```

## Deployment on Hostinger VPS

### Step 1: Connect to Hostinger VPS

1. Get your VPS credentials from Hostinger Control Panel
2. Connect via SSH:
```bash
ssh root@your_vps_ip
```

### Step 2: Initial Server Setup (Hostinger Specific)
```bash
# Update system
apt update && apt upgrade -y

# Install required packages
apt install python3 python3-pip nginx supervisor git ufw -y

# Configure firewall (Hostinger specific)
ufw allow 22    # SSH
ufw allow 80    # HTTP
ufw allow 443   # HTTPS
ufw allow 8080  # API port
ufw enable

# Create non-root user (recommended)
adduser tvdata_user
usermod -aG sudo tvdata_user
```

### Step 3: Setup Python Environment
```bash
# Switch to application user
su - tvdata_user

# Create project directory
mkdir ~/tvdata_api
cd ~/tvdata_api

# Create Python virtual environment
python3 -m venv venv
source venv/bin/activate

# Clone your project or copy files
# Option 1: Copy files
scp scrpt.py requirements.txt your_username@your_vps_ip:~/tvdata_api/

# Option 2: Clone from git (if using git)
# git clone your_repository_url .

# Install dependencies
pip install -r requirements.txt
```

### Step 4: Configure Supervisor (Hostinger Specific)
```bash
# Create supervisor config
sudo nano /etc/supervisor/conf.d/tvdata_api.conf
```

Add the following configuration:
```ini
[program:tvdata_api]
directory=/home/tvdata_user/tvdata_api
command=/home/tvdata_user/tvdata_api/venv/bin/gunicorn scrpt:app -w 4 -k uvicorn.workers.UvicornWorker -b 127.0.0.1:8080
user=tvdata_user
autostart=true
autorestart=true
stderr_logfile=/var/log/tvdata_api.err.log
stdout_logfile=/var/log/tvdata_api.out.log
environment=PATH="/home/tvdata_user/tvdata_api/venv/bin"
```

### Step 5: Configure Nginx on Hostinger
```bash
# Create Nginx config
sudo nano /etc/nginx/sites-available/tvdata_api
```

Add the following configuration:
```nginx
server {
    listen 80;
    server_name your_domain.com;  # Your Hostinger domain

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }

    # Enable SSL if you have it
    # listen 443 ssl;
    # ssl_certificate /path/to/cert.pem;
    # ssl_certificate_key /path/to/key.pem;
}
```

### Step 6: Enable and Start Services
```bash
# Enable Nginx config
sudo ln -s /etc/nginx/sites-available/tvdata_api /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# Start supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start tvdata_api
```

### Step 7: SSL Setup with Hostinger
1. Go to Hostinger Control Panel
2. Navigate to SSL/TLS section
3. Install Let's Encrypt certificate
4. Update Nginx configuration to use SSL

### Step 8: Monitoring on Hostinger
```bash
# Check API status
sudo supervisorctl status tvdata_api

# View API logs
sudo tail -f /var/log/tvdata_api.err.log

# Monitor Nginx access
sudo tail -f /var/log/nginx/access.log

# Check system resources
htop
```
