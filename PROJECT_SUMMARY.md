# Real-Time Voting Data Engineering Project Summary

## Overview
This project implements a real-time election voting system that processes voting data through a distributed data pipeline using Kafka, Spark Streaming, PostgreSQL, and Streamlit. The system simulates an election by generating voter data, processing votes in real-time, aggregating results, and displaying them on a live dashboard.

## Architecture

### Data Flow
1. **Data Generation** (`main.py`) → PostgreSQL + Kafka (`voters_topic`)
2. **Vote Processing** (`voting.py`) → Kafka (`voters_topic`) → PostgreSQL + Kafka (`votes_topic`)
3. **Stream Processing** (`spark-streaming.py`) → Kafka (`votes_topic`) → Kafka (aggregated topics)
4. **Visualization** (`streamlit-app.py`) → Kafka (aggregated topics) + PostgreSQL → Dashboard

### Infrastructure Components
- **Zookeeper**: Coordination service for Kafka
- **Kafka**: Message broker for streaming data
- **PostgreSQL**: Persistent storage for voters, candidates, and votes
- **Spark Streaming**: Real-time data processing and aggregation
- **Streamlit**: Web-based dashboard for visualization

## Core Components

### 1. main.py
**Purpose**: Initializes the system and generates voter/candidate data

**Key Functions**:
- `generate_voter_data()`: Fetches random user data from randomuser.me API and formats it as voter information
- `generate_candidate_data()`: Generates candidate data with party affiliations
- `create_tables()`: Creates PostgreSQL tables for candidates, voters, and votes
- `insert_voters()`: Inserts voter data into PostgreSQL

**Operations**:
- Creates 3 candidates (one per party) if none exist
- Generates 1000 voters and stores them in PostgreSQL
- Publishes voter data to Kafka topic `voters_topic`

**Kafka Topics Used**:
- `voters_topic`: Output topic for voter data

### 2. voting.py
**Purpose**: Simulates the voting process by consuming voter data and generating votes

**Key Functions**:
- `consume_messages()`: Consumes candidate data from Kafka (currently unused in main flow)
- Main loop: Consumes voters from `voters_topic`, randomly assigns candidates, and creates votes

**Operations**:
- Reads candidates from PostgreSQL
- Consumes voter messages from `voters_topic`
- For each voter, randomly selects a candidate
- Inserts vote into PostgreSQL `votes` table (with duplicate prevention)
- Publishes enriched vote data to `votes_topic`

**Kafka Topics Used**:
- `voters_topic`: Input (consumes voter data)
- `votes_topic`: Output (publishes vote data with voter and candidate information)

**Data Enrichment**:
- Merges voter data, candidate data, voting timestamp, and vote count into a single message

### 3. spark-streaming.py
**Purpose**: Real-time stream processing and aggregation of voting data

**Key Features**:
- SparkSession configured with Kafka integration and PostgreSQL driver
- Local execution mode with 2GB memory allocation
- Checkpoint-based fault tolerance for streaming queries

**Operations**:
- Consumes from `votes_topic` Kafka topic
- Applies watermarking (1 minute) for late data handling
- Performs two aggregations:
  1. **Votes per candidate**: Groups by candidate_id, candidate_name, party_affiliation, photo_url and sums votes
  2. **Turnout by location**: Groups by address.state and counts votes
- Publishes aggregated results to Kafka topics:
  - `aggregated_votes_per_candidate`
  - `aggregated_turnout_by_location`

**Kafka Topics Used**:
- `votes_topic`: Input (consumes vote data)
- `aggregated_votes_per_candidate`: Output (publishes candidate vote totals)
- `aggregated_turnout_by_location`: Output (publishes location-based turnout)

**Checkpoint Locations**:
- `checkpoints/checkpoint1`: For votes per candidate aggregation
- `checkpoints/checkpoint2`: For turnout by location aggregation

**Note**: Contains commented-out code for additional aggregations (by age, gender, party, region) and PostgreSQL joins that could be enabled for extended functionality.

### 4. streamlit-app.py
**Purpose**: Real-time dashboard for visualizing election results

**Key Functions**:
- `create_kafka_consumer()`: Creates Kafka consumer for specified topic
- `fetch_voting_stats()`: Retrieves total voters and candidates count from PostgreSQL
- `fetch_data_from_kafka()`: Polls Kafka for new messages
- `plot_colored_bar_chart()`: Creates bar chart visualization
- `plot_donut_chart()`: Creates donut chart visualization
- `plot_pie_chart()`: Creates pie chart visualization
- `paginate_table()`: Implements pagination for large tables
- `update_data()`: Main function that refreshes dashboard data
- `sidebar()`: Configures refresh interval and manual refresh button

**Dashboard Features**:
- **Metrics**: Total voters and candidates count
- **Leading Candidate**: Displays photo, name, party, and vote count
- **Statistics**: 
  - Bar chart showing votes per candidate
  - Donut chart showing vote distribution
  - Table with candidate statistics
- **Location Analysis**: Paginated table showing voter turnout by state
- **Auto-refresh**: Configurable refresh interval (5-60 seconds)

**Kafka Topics Consumed**:
- `aggregated_votes_per_candidate`: For candidate vote statistics
- `aggregated_turnout_by_location`: For location-based turnout data

**Data Sources**:
- PostgreSQL: For voter and candidate counts
- Kafka: For real-time aggregated voting data

### 5. docker-compose.yml
**Purpose**: Infrastructure orchestration

**Services**:
- **zookeeper**: Confluent Zookeeper 7.4.0 on port 2181
- **broker**: Confluent Kafka 7.4.0 on port 9092 (with JMX on 9101)
- **postgres**: PostgreSQL latest on port 5432
  - Database: `voting`
  - User: `postgres`
  - Password: `postgres`

**Note**: Spark services are commented out but could be enabled for distributed processing

## Database Schema

### candidates Table
- `candidate_id` (VARCHAR, PRIMARY KEY)
- `candidate_name` (VARCHAR)
- `party_affiliation` (VARCHAR)
- `biography` (TEXT)
- `campaign_platform` (TEXT)
- `photo_url` (TEXT)

### voters Table
- `voter_id` (VARCHAR, PRIMARY KEY)
- `voter_name` (VARCHAR)
- `date_of_birth` (VARCHAR)
- `gender` (VARCHAR)
- `nationality` (VARCHAR)
- `registration_number` (VARCHAR)
- `address_street`, `address_city`, `address_state`, `address_country`, `address_postcode` (VARCHAR)
- `email`, `phone_number`, `cell_number` (VARCHAR)
- `picture` (TEXT)
- `registered_age` (INTEGER)

### votes Table
- `voter_id` (VARCHAR, UNIQUE)
- `candidate_id` (VARCHAR)
- `voting_time` (TIMESTAMP)
- `vote` (INT, DEFAULT 1)
- PRIMARY KEY: (voter_id, candidate_id)

## Kafka Topics

1. **voters_topic**: Voter registration data
2. **candidates_topic**: Candidate information (currently generated in main.py, not actively used in voting.py)
3. **votes_topic**: Complete vote records (voter + candidate + timestamp)
4. **aggregated_votes_per_candidate**: Real-time vote totals per candidate
5. **aggregated_turnout_by_location**: Real-time voter turnout by state

## Technology Stack

### Python Libraries
- **confluent-kafka**: Kafka producer/consumer
- **kafka-python**: Alternative Kafka client (used in Streamlit)
- **pyspark**: Spark Streaming for real-time processing
- **psycopg2**: PostgreSQL database connector
- **streamlit**: Web dashboard framework
- **pandas**: Data manipulation
- **matplotlib**: Data visualization
- **requests**: HTTP requests for API data generation
- **simplejson**: JSON serialization

### Infrastructure
- **Apache Kafka**: Message streaming platform
- **Apache Spark**: Distributed stream processing
- **PostgreSQL**: Relational database
- **Docker Compose**: Container orchestration

## Execution Flow

1. **Start Infrastructure**: `docker-compose up -d`
2. **Initialize Data**: `python main.py` (creates tables, generates candidates and voters)
3. **Process Votes**: `python voting.py` (consumes voters, generates votes)
4. **Stream Processing**: `python spark-streaming.py` (aggregates votes in real-time)
5. **View Dashboard**: `streamlit run streamlit-app.py` (visualizes results)

## Key Design Patterns

- **Event-Driven Architecture**: All components communicate via Kafka topics
- **Stream Processing**: Spark Streaming enables real-time aggregation
- **Idempotency**: Vote processing prevents duplicate votes using database constraints
- **Fault Tolerance**: Spark checkpoints enable recovery from failures
- **Watermarking**: Handles late-arriving data in stream processing
- **Caching**: Streamlit uses caching for database queries to improve performance

## Data Sources

- **Random User API** (`randomuser.me`): Generates realistic voter and candidate profiles
- **PostgreSQL**: Stores persistent data (voters, candidates, votes)
- **Kafka**: Streams real-time voting events

## Current Limitations & Future Enhancements

### Commented-Out Features in spark-streaming.py
- Aggregation by age group
- Aggregation by gender
- Aggregation by party (separate from candidate)
- Aggregation by region/city
- PostgreSQL joins for data enrichment

### Potential Improvements
- Distributed Spark cluster (currently local mode)
- Additional visualization types
- Historical data analysis
- Real-time alerts for anomalies
- Multi-election support
- Authentication and authorization

## Project Structure
```
realtime-voting-data-engineering/
├── main.py                 # Data generation and initialization
├── voting.py               # Vote processing simulation
├── spark-streaming.py      # Real-time stream processing
├── streamlit-app.py        # Dashboard application
├── docker-compose.yml      # Infrastructure configuration
├── requirements.txt        # Python dependencies
├── README.md              # Project documentation
├── checkpoints/           # Spark streaming checkpoints
│   ├── checkpoint1/       # Votes per candidate checkpoint
│   └── checkpoint2/       # Turnout by location checkpoint
└── images/                # Documentation images
```

