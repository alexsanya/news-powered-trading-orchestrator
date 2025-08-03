# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Full System Orchestrator** for a multi-service cryptocurrency monitoring and trading signal generation system. The orchestrator coordinates two specialized microservices using Docker Compose: a Twitter streaming service for real-time tweet collection and an AI-powered sentiment analysis service for cryptocurrency token detection. The system processes social media content through a complete pipeline from data ingestion to actionable trading signals.

## System Architecture

**Multi-Service Event-Driven Architecture** with message-driven coordination:

### Core Services

- **tweets-notifier** (Git Submodule): Real-time Twitter streaming service
  - WebSocket connection to Twitter API for live tweet collection
  - Publishes tweets to `tweet_events` RabbitMQ queue
  - Handles connection monitoring, buffering, and graceful shutdown
  
- **sentiment-analyser** (Git Submodule): AI-powered cryptocurrency token detection
  - Consumes tweets from `tweet_events` queue using threaded message processing
  - Analyzes content using PydanticAI agents (text, image, web scraping)
  - Publishes trading signals to `actions_to_take` queue when tokens are detected

### Infrastructure Services

- **RabbitMQ**: Message broker for inter-service communication
  - Management UI available at http://localhost:15672
  - Handles `tweet_events` and `actions_to_take` queues
  - Persistent storage with health checks

- **Firecrawl**: Web scraping service for link analysis
  - MCP server for crawling external websites mentioned in tweets
  - Supports sentiment analysis agent web content extraction

## Technology Stack

- **Orchestration**: Docker Compose with custom bridge networking
- **Message Broker**: RabbitMQ 3.13 with management plugin
- **Web Scraping**: Firecrawl MCP server (Node.js 18)
- **Services**: Python 3.12 microservices with UV package management
- **AI Integration**: OpenAI GPT-4o via PydanticAI framework
- **Observability**: Structured logging with optional Logfire tracing

## Development Commands

```bash
# Start all services
docker-compose up -d

# View logs for all services
docker-compose logs -f

# View logs for specific service
docker-compose logs -f tweets-notifier
docker-compose logs -f sentiment-analyser
docker-compose logs -f rabbitmq
docker-compose logs -f firecrawl

# Rebuild and restart after code changes
docker-compose up -d --build

# Stop all services
docker-compose down

# Stop services and remove volumes (clean slate)
docker-compose down -v

# Access RabbitMQ Management UI
# http://localhost:15672 (admin/changeme by default)

# Access Firecrawl health endpoint
# http://localhost:3000/sse

# Work with submodules
git submodule update --init --recursive  # Initialize submodules
git submodule update --remote            # Update to latest commits
git submodule foreach git pull origin main  # Pull latest for all submodules

# Individual service development (run from submodule directories)
cd tweets-notifier && python main.py     # Run tweets notifier locally
cd sentiment-analyser && python main.py  # Run sentiment analyser locally
```

## Configuration

### Environment Variables

Create a `.env` file in the root directory with the following configuration:

```bash
# Twitter API Configuration (Required)
TWITTERAPI_KEY=your_twitter_api_key_here

# OpenAI Configuration (Required for sentiment analysis)
OPENAI_API_KEY=your_openai_api_key_here

# Firecrawl Configuration (Optional)
FIRECRAWL_API_KEY=your_firecrawl_api_key_here

# Application Environment
ENVIRONMENT=development  # or 'production' for JSON logs

# RabbitMQ Configuration
RABBITMQ_HOST=rabbitmq  # Auto-configured for Docker
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=admin
RABBITMQ_PASSWORD=changeme
RABBITMQ_QUEUE=tweet_events
RABBITMQ_CONSUME_QUEUE=tweet_events
ACTIONS_QUEUE_NAME=actions_to_take

# RabbitMQ Connection Monitoring
RABBITMQ_MONITOR_ENABLED=true
RABBITMQ_MONITOR_INTERVAL=30
RABBITMQ_MAX_RETRY_ATTEMPTS=3
RABBITMQ_RETRY_DELAY=5

# Message Buffer Configuration
MESSAGE_BUFFER_ENABLED=true
MESSAGE_BUFFER_SIZE=10

# Sentiment Analysis Configuration
SENTIMENT_MODEL_NAME=openai:gpt-4o
MAX_CONCURRENT_ANALYSIS=5
AGENT_RETRIES=4

# Logfire Observability (Optional)
LOGFIRE_ENABLED=true
LOGFIRE_TOKEN=your_logfire_token_here
LOGFIRE_SERVICE_NAME=full-system
LOGFIRE_ENVIRONMENT=development
SERVICE_VERSION=0.1.0
```

### Service Dependencies

The orchestrator manages service startup order:

1. **RabbitMQ** starts first with health checks
2. **Firecrawl** starts in parallel with RabbitMQ
3. **tweets-notifier** starts after RabbitMQ is healthy
4. **sentiment-analyser** starts after both RabbitMQ and Firecrawl are healthy

## Message Flow Architecture

**Complete Data Pipeline**:

```
Twitter API → tweets-notifier → RabbitMQ → sentiment-analyser → Actions Queue
     ↓              ↓              ↓              ↓                ↓
   WebSocket     Transforms    tweet_events   AI Analysis    actions_to_take
  Connection      & Buffers      Queue        (3 Agents)        Queue
```

### Message Processing Flow

1. **Tweet Collection**: tweets-notifier connects to Twitter streaming API
2. **Message Publishing**: Tweets are transformed and published to `tweet_events` queue
3. **Consumption**: sentiment-analyser consumes tweets using threaded processing
4. **AI Analysis**: PydanticAI agents analyze text, images, and linked websites
5. **Token Detection**: When cryptocurrency tokens are detected, details are extracted
6. **Action Publishing**: Detected tokens trigger snipe actions published to `actions_to_take` queue

### AI Agent Workflow

The sentiment analysis service uses three specialized agents:

- **TextSearchAgent**: Analyzes tweet text for blockchain addresses and announcements
- **ImageSearchAgent**: Extracts text from images and analyzes for token information  
- **FirecrawlAgent**: Scrapes linked websites for additional token announcement details

## Network Architecture

**Docker Bridge Network**: `trading_network`

- **Internal Communication**: Services communicate via service names (rabbitmq, firecrawl)
- **External Access**: 
  - RabbitMQ Management: `localhost:15672`
  - Firecrawl MCP: `localhost:3000`
- **Service Discovery**: Automatic DNS resolution between containers
- **Network Isolation**: All services isolated on custom bridge network

## File Structure

```
├── docker-compose.yml        # Multi-service orchestration configuration
├── .env                      # Environment configuration (create from template)
├── tweets-notifier/          # Git submodule - Twitter streaming service
│   ├── main.py              # WebSocket Twitter API service
│   ├── src/                 # WebSocket management, MQ publishing, handlers
│   ├── Dockerfile           # Container build for tweets-notifier
│   ├── docker-compose.yml   # Individual service compose (for development)
│   └── CLAUDE.md            # Service-specific documentation
├── sentiment-analyser/       # Git submodule - AI sentiment analysis service  
│   ├── main.py              # RabbitMQ consumer and AI agent orchestration
│   ├── src/                 # MQ consumption, sentiment analysis, AI agents
│   ├── Dockerfile           # Container build for sentiment-analyser
│   ├── docker-compose.yml   # Individual service compose (for development)
│   └── CLAUDE.md            # Service-specific documentation
├── logs/                     # Aggregated log files (auto-created)
│   ├── tweets-notifier/     # tweets-notifier service logs
│   └── sentiment-analyser/  # sentiment-analyser service logs
└── README.md                # System overview and setup instructions
```

## Key Features

### Fault Tolerance & Reliability

- **Health Checks**: All services include comprehensive health monitoring
- **Restart Policies**: Automatic restart on failures with `unless-stopped`
- **Message Buffering**: Temporary message storage during RabbitMQ outages
- **Connection Monitoring**: Automatic reconnection with configurable retry logic
- **Graceful Shutdown**: Signal handling for clean service termination

### Scalability & Performance

- **Threaded Processing**: sentiment-analyser uses thread-per-message for concurrent AI analysis
- **Message Queues**: Asynchronous processing prevents bottlenecks
- **Agent Concurrency**: Configurable concurrent AI agent execution limits
- **Resource Isolation**: Each service runs in isolated Docker containers

### Observability & Monitoring

- **Structured Logging**: JSON logs with rotation and level separation
- **Service Health**: Health check endpoints for all services
- **Message Tracing**: Comprehensive logging throughout the message pipeline
- **AI Agent Monitoring**: Logfire integration for PydanticAI execution tracing
- **Real-time Status**: RabbitMQ management UI for queue monitoring

## Development Workflow

### Initial Setup

```bash
# Clone repository with submodules
git clone --recursive <repository-url>
cd full-system

# Create environment configuration
cp .env.example .env  # Edit with your API keys

# Start all services
docker-compose up -d

# Monitor startup logs
docker-compose logs -f
```

### Development Mode

```bash
# Work on individual services
cd tweets-notifier
# Make changes to tweets-notifier code

cd ../sentiment-analyser  
# Make changes to sentiment-analyser code

# Rebuild and restart modified services
docker-compose up -d --build tweets-notifier
docker-compose up -d --build sentiment-analyser
```

### Submodule Management

```bash
# Update submodules to latest commits
git submodule update --remote

# Commit submodule updates
git add tweets-notifier sentiment-analyser
git commit -m "Update submodules to latest versions"

# Work with individual submodule repositories
cd tweets-notifier
git checkout -b feature-branch
# Make changes and commit
git push origin feature-branch

# Update orchestrator to use new submodule commit
cd ..
git add tweets-notifier
git commit -m "Update tweets-notifier to include new feature"
```

## Monitoring & Troubleshooting

### Service Health Monitoring

- **RabbitMQ Management**: http://localhost:15672 - Monitor queues and connections
- **Container Health**: `docker-compose ps` - Check service health status
- **Service Logs**: `docker-compose logs <service-name>` - View service-specific logs

### Common Issues & Solutions

**RabbitMQ Connection Issues**:
```bash
# Check RabbitMQ health
docker-compose exec rabbitmq rabbitmq-diagnostics check_running
# Restart RabbitMQ
docker-compose restart rabbitmq
```

**AI Agent Failures**:
```bash
# Check OpenAI API key configuration
docker-compose exec sentiment-analyser env | grep OPENAI
# View detailed agent logs
docker-compose logs sentiment-analyser | grep -i agent
```

**Service Startup Issues**:
```bash
# Check service dependencies
docker-compose config --services
# Verify health check status
docker-compose ps
```

## Message Schemas

### Tweet Events (tweet_events queue)

```json
{
  "data_source": {
    "name": "Twitter",
    "author_name": "username", 
    "author_id": "12345"
  },
  "createdAt": 1721765647,
  "text": "Tweet content...",
  "media": ["https://..."],
  "links": ["https://..."],
  "sentiment_analysis": {
    "chain_id": 1,
    "chain_name": "Ethereum",
    "is_release": true,
    "token_address": "0x742d35Cc6765C0532575f5A2c0a078Df8a2D4e5e"
  }
}
```

### Snipe Actions (actions_to_take queue)

```json
{
  "action": "snipe",
  "params": {
    "chain_id": 1,
    "chain_name": "Ethereum", 
    "token_address": "0x742d35Cc6765C0532575f5A2c0a078Df8a2D4e5e"
  }
}
```

## Security Considerations

- **API Keys**: Store sensitive keys in `.env` file (not committed to Git)
- **Network Isolation**: Services communicate only through controlled bridge network
- **Access Control**: RabbitMQ authentication required for all connections
- **Container Security**: Services run with non-root users where possible
- **Log Security**: Avoid logging sensitive information (API keys, tokens)

## Production Deployment

### Docker Compose Production

```bash
# Set production environment
ENVIRONMENT=production docker-compose up -d

# Use production-grade settings
# - JSON structured logging
# - Resource limits and reservations
# - Health check monitoring
# - Log rotation policies
```

### Scaling Considerations

- **RabbitMQ Clustering**: For high availability, consider RabbitMQ cluster setup
- **Service Replicas**: Use Docker Swarm or Kubernetes for service scaling
- **Load Balancing**: Implement load balancing for multiple sentiment-analyser instances
- **Database Integration**: Add persistent storage for processed results and analytics

## Development Notes

### Orchestrator Design Principles

- **Service Autonomy**: Each service is independently deployable and testable
- **Loose Coupling**: Services communicate only through message queues
- **Infrastructure as Code**: Complete system defined in docker-compose.yml
- **Environment Parity**: Development and production use identical container setup
- **Submodule Strategy**: Service code maintained in separate repositories for independent development

### Message-Driven Architecture Benefits

- **Scalability**: Services can be scaled independently based on load
- **Resilience**: Service failures don't cascade due to queue buffering
- **Flexibility**: New services can be added by consuming existing message queues
- **Debugging**: Message queues provide visibility into system data flow
- **Testing**: Services can be tested independently with mock message producers/consumers

### Integration Points

- **Queue Naming**: Standardized queue names across services (`tweet_events`, `actions_to_take`)
- **Message Schemas**: Consistent Pydantic schemas for type safety
- **Error Handling**: Comprehensive error handling with message dead letter queues
- **Monitoring Integration**: Unified logging and observability across all services