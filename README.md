# Full System Orchestrator

A multi-service cryptocurrency monitoring and trading signal generation system that processes real-time Twitter streams through AI-powered sentiment analysis to detect token announcements and generate trading signals.

## Architecture

This orchestrator coordinates two specialized microservices using Docker Compose:

- **tweets-notifier**: Real-time Twitter streaming service with WebSocket connections
- **sentiment-analyser**: AI-powered cryptocurrency token detection using PydanticAI agents
- **RabbitMQ**: Message broker for inter-service communication
- **Firecrawl**: Web scraping service for link analysis

## Quick Start

1. **Clone with submodules**:
   ```bash
   git clone --recursive <repository-url>
   cd full-system
   ```

2. **Configure environment**:
   ```bash
   cp .env.example .env
   # Edit .env with your API keys (Twitter, OpenAI, Firecrawl)
   ```

3. **Start all services**:
   ```bash
   docker-compose up -d
   ```

4. **Monitor logs**:
   ```bash
   docker-compose logs -f
   ```

## Required Environment Variables

```bash
# Required
TWITTERAPI_KEY=your_twitter_api_key_here
OPENAI_API_KEY=your_openai_api_key_here

# Optional
FIRECRAWL_API_KEY=your_firecrawl_api_key_here
LOGFIRE_TOKEN=your_logfire_token_here
```

## Services

- **RabbitMQ Management**: http://localhost:15672 (admin/changeme)
- **Firecrawl Health**: http://localhost:3000/sse

## Data Flow

```
Twitter API → tweets-notifier → RabbitMQ → sentiment-analyser → Actions Queue
```

1. **Tweet Collection**: Real-time Twitter stream processing
2. **AI Analysis**: Text, image, and web content analysis for token detection
3. **Signal Generation**: Trading signals published when tokens are detected

## Development Commands

```bash
# Rebuild and restart after changes
docker-compose up -d --build

# View specific service logs
docker-compose logs -f tweets-notifier
docker-compose logs -f sentiment-analyser

# Stop all services
docker-compose down

# Clean restart (removes volumes)
docker-compose down -v

# Update submodules
git submodule update --remote
```

## Submodules

- `tweets-notifier/`: Twitter streaming service (separate repository)
- `sentiment-analyser/`: AI sentiment analysis service (separate repository)

Each submodule has its own documentation and can be developed independently.

## Monitoring

- **Service Health**: `docker-compose ps`
- **Queue Status**: RabbitMQ Management UI
- **Logs**: Structured JSON logging with rotation
- **Observability**: Optional Logfire integration

## Message Schemas

**Tweet Events**:
```json
{
  "data_source": {"name": "Twitter", "author_name": "username"},
  "text": "Tweet content...",
  "media": ["https://..."],
  "links": ["https://..."]
}
```

**Trading Signals**:
```json
{
  "action": "snipe",
  "params": {
    "chain_id": 1,
    "token_address": "0x742d35Cc..."
  }
}
```

## Contributing

1. Work on individual services in their submodule directories
2. Test changes with `docker-compose up -d --build`
3. Update submodule references when committing changes
4. See individual service documentation for detailed development guides