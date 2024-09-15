# Setting Up and Development for Aptos Indexers

This guide provides detailed instructions on setting up the development environment and creating custom processors for Aptos Indexers using the `aptos-indexer-processors` repository.

## Repository Structure

The `aptos-indexer-processors` repository supports custom processors in multiple languages. The structure for Python processors is as follows:

```
aptos-indexer-processors
├── python
│   ├── processors
│   │   ├── coin_flip (example processor)
│   │   │   ├── models.py
│   │   │   ├── processor.py
│   │   │   └── README.md
│   │   ├── main.py
│   │   └── README.md
│   ├── scripts
│   ├── utils
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── poetry.lock
│   ├── pyproject.toml
│   └── README.md
```

## Prerequisites

- Python 3.11+
- Poetry (for dependency management)
- PostgreSQL
- Docker and Docker Compose (optional, for containerized deployment)

## Development Environment Setup

1. Clone the repository:
   ```
   git clone https://github.com/aptos-labs/aptos-indexer-processors.git
   cd aptos-indexer-processors/python
   ```

2. Install dependencies:
   ```
   poetry install
   ```

3. Set up PostgreSQL:
   - Install PostgreSQL
   - Create a database for your processor

4. Configure your environment:
   - Copy `config.yaml.example` to `config.yaml`
   - Update `config.yaml` with your database credentials and other settings

## Creating a Custom Processor

To create a custom processor, you'll need to implement three main components:

1. Database Models
2. Processor Logic
3. Configuration

### 1. Database Models

Define your database models in a `models.py` file. Use SQLAlchemy ORM and the provided base classes. Example:

```python
from utils.models.annotated_types import (
    BooleanType, StringPrimaryKeyType, BigIntegerType,
    BigIntegerPrimaryKeyType, InsertedAtType, TimestampType, NumericType
)
from utils.models.general_models import Base
from utils.models.schema_names import YOUR_SCHEMA_NAME

class YourEventModel(Base):
    __tablename__ = "your_event_table"
    __table_args__ = ({"schema": YOUR_SCHEMA_NAME},)

    sequence_number: BigIntegerPrimaryKeyType
    creation_number: BigIntegerPrimaryKeyType
    account_address: StringPrimaryKeyType
    # Add your custom fields here
    transaction_version: BigIntegerType
    transaction_timestamp: TimestampType
    inserted_at: InsertedAtType
    event_index: BigIntegerType
```

### 2. Processor Logic

Implement your processor logic in a `processor.py` file. Extend the `TransactionsProcessor` class:

```python
from utils.transactions_processor import TransactionsProcessor, ProcessingResult
from utils.models.schema_names import YOUR_SCHEMA_NAME
from utils.processor_name import ProcessorName
from your_processor.models import YourEventModel

class YourCustomProcessor(TransactionsProcessor):
    def name(self) -> str:
        return ProcessorName.YOUR_PROCESSOR_NAME.value

    def schema(self) -> str:
        return YOUR_SCHEMA_NAME

    def process_transactions(
        self,
        transactions: list[transaction_pb2.Transaction],
        start_version: int,
        end_version: int,
    ) -> ProcessingResult:
        event_db_objs = []
        for transaction in transactions:
            # Process transactions and events
            # Create YourEventModel instances
            # Append to event_db_objs

        self.insert_to_db(event_db_objs)
        return ProcessingResult(...)

    def insert_to_db(self, parsed_objs: List[YourEventModel]) -> None:
        with Session() as session, session.begin():
            for obj in parsed_objs:
                session.merge(obj)

    @staticmethod
    def included_event_type(event_type: str) -> bool:
        # Implement your event filtering logic
        pass
```

### 3. Configuration

Update the configuration files to include your new processor:

1. In `utils/processor_name.py`, add your processor name:
   ```python
   class ProcessorName(Enum):
       YOUR_PROCESSOR_NAME = "your_processor_name"
   ```

2. In `utils/worker.py`, add your processor to the match cases:
   ```python
   match self.config.processor_name:
       case ProcessorName.YOUR_PROCESSOR_NAME.value:
           self.processor = YourCustomProcessor()
   ```

3. In `utils/models/schema_names.py`, add your schema name:
   ```python
   YOUR_SCHEMA_NAME = "your_schema_name"
   ```

## Running Your Processor

1. Ensure your `config.yaml` is properly set up with your processor details and database configuration.

2. Run the processor:
   ```
   poetry run python -m processors.main -c config.yaml
   ```

## Docker Deployment

To run your processor in a Docker container:

1. Ensure your `config.yaml` is properly configured.

2. Build and run the Docker container:
   ```
   docker compose up --build --force-recreate
   ```

## Best Practices

1. Event Filtering: Implement efficient event filtering in the `included_event_type` method to process only relevant events.

2. Error Handling: Implement robust error handling in your processor to manage potential issues with transaction processing or database operations.

3. Performance Optimization: Monitor and optimize the performance of your processor, especially for high-volume event processing.

4. Testing: Develop comprehensive tests for your processor to ensure reliability and correctness.

5. Documentation: Maintain clear documentation for your processor, including its purpose, event types processed, and any specific configuration requirements.

## Troubleshooting

- If you encounter GRPC deserialization recursion limits with Python, consider using the Rust processors for production-grade indexers.
- Ensure your PostgreSQL connection string in `config.yaml` is correct and the database is accessible.
- Check the Aptos documentation for the latest GRPC endpoints if you're having connection issues.

By following this guide, you should be able to set up your development environment and create custom processors for Aptos Indexers using the `aptos-indexer-processors` repository.