# Using the Web Interface

Cracktrader provides a modern web interface for monitoring and controlling your trading strategies. The web interface offers real-time data visualization, strategy management, and performance analytics.

## Starting the Web Interface

To launch the web interface:

```python
from cracktrader.web import start_web_server

# Start the web server
start_web_server(host="0.0.0.0", port=8080)
```

Or use the command line:

```bash
cracktrader web --host 0.0.0.0 --port 8080
```

## Features

### Dashboard Overview
- Real-time portfolio performance
- Active strategy status
- Exchange connection health
- System resource monitoring

### Strategy Management
- Start/stop strategies
- View strategy parameters
- Monitor strategy performance
- Real-time position tracking

### Data Visualization
- Interactive price charts
- Performance analytics
- Trade history
- Risk metrics

### System Monitoring
- Exchange connectivity status
- Data feed health
- System resource usage
- Error logs and alerts

## Configuration

The web interface can be configured through the main configuration file or environment variables:

```json
{
  "web": {
    "host": "0.0.0.0",
    "port": 8080,
    "debug": false,
    "cors_origins": ["http://localhost:3000"]
  }
}
```

## Security Considerations

- The web interface should only be accessible from trusted networks
- Use HTTPS in production environments
- Configure proper authentication if exposing publicly
- Monitor access logs regularly

## Next Steps

- [Configuration](configuration.md) - Learn about web interface configuration options
- [Web API Reference](../reference/web_api.md) - Explore the REST API endpoints
