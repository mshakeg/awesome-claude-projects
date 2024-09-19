# Setting Up and Developing Aptos Indexers

This guide provides detailed instructions on setting up the development environment and creating custom processors for Aptos Indexers using the `aptos-indexer-processors` repository. It covers everything from setting up your environment to developing custom processors, including creating database models, implementing processor logic, and event filtering.

---

**Note:** While this guide uses Python for demonstration purposes, it is strongly recommended to use **Rust** for production-grade indexers due to better performance and ecosystem support. The Python implementation has known issues with gRPC deserialization and may not handle large data volumes efficiently.

---

## Repository Structure

The `aptos-indexer-processors` repository supports custom processors in multiple languages, including **Python**, **Rust**, and **TypeScript**. The structure for Python processors is as follows:

```
aptos-indexer-processors
├── python
│   ├── processors
│   │   ├── coin_flip (example processor)
│   │   │   ├── models.py
│   │   │   ├── processor.py
│   │   │   └── README.md
│   │   ├── your_processor (your custom processor)
│   │   │   ├── models.py
│   │   │   ├── processor.py
│   │   │   └── __init__.py
│   │   ├── main.py
│   │   └── __init__.py
│   ├── scripts
│   ├── utils
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── poetry.lock
│   ├── pyproject.toml
│   └── README.md
```

## Prerequisites

- **Python 3.11+**
- **Poetry** (for dependency management): [Installation Guide](https://python-poetry.org/docs/#installation)
- **PostgreSQL**: [Installation Guide](https://www.postgresql.org/download/)
- **Aptos CLI** (optional, for interacting with the Aptos blockchain): [Installation Guide](https://aptos.dev/tools/cli/)
- **Docker and Docker Compose** (optional, for containerized deployment): [Docker Installation](https://docs.docker.com/get-docker/)
- **API Key**: An API key from the Aptos Developer Portal for accessing the Transaction Stream Service.

## Development Environment Setup

### 1. Clone the Repository

```bash
git clone https://github.com/aptos-labs/aptos-indexer-processors.git
cd aptos-indexer-processors/python
```

### 2. Install Dependencies

```bash
poetry install
```

### 3. Set Up PostgreSQL

- **Install PostgreSQL** if you haven't already.
- **Start the PostgreSQL service**:
  - On Linux:
    ```bash
    sudo service postgresql start
    ```
  - On macOS (using Homebrew):
    ```bash
    brew services start postgresql
    ```
- **Create a database** for your processor (e.g., `your_processor_db`).
- **Optionally**, create a user and set a password if needed.

### 4. Configure Your Environment

- **Copy the example configuration**:
  ```bash
  cp config.yaml.example config.yaml
  ```
- **Update `config.yaml`** with your database credentials and other settings:
  - Set the `postgres_connection_string`:
    ```yaml
    postgres_connection_string: "postgresql://username:password@localhost:5432/your_processor_db"
    ```
  - Set the `processor_name`:
    ```yaml
    processor_name: "your_processor_name"
    ```
  - Set the `indexer_grpc_data_service_address` to the appropriate endpoint:
    - **Mainnet**: `grpc.mainnet.aptoslabs.com:443`
    - **Testnet**: `grpc.testnet.aptoslabs.com:443`
    - **Devnet**: `grpc.devnet.aptoslabs.com:443`
  - Set the `auth_token` with your API key obtained from the [Aptos Developer Portal](https://developers.aptoslabs.com/):
    ```yaml
    auth_token: "YOUR_API_KEY"
    ```
  - (Optional) Set `starting_version` and `ending_version` if you wish to index a specific range of transactions.

---

## Creating a Custom Processor

To create a custom processor, you'll need to implement three main components:

1. **Database Models**
2. **Processor Logic**
3. **Configuration**

### 1. Database Models

Define your database models in a `models.py` file within your processor's directory. Use SQLAlchemy ORM and the provided base classes.

**Steps to Create a Database Model:**

1. **Create a `models.py` File**

   In your processor directory (`aptos-indexer-processors/python/processors/your_processor/`), create a file named `models.py`.

2. **Import Necessary Types and Base Classes**

   ```python
   # your_processor/models.py

   from utils.models.annotated_types import (
       BooleanType,
       StringType,
       StringPrimaryKeyType,
       BigIntegerType,
       BigIntegerPrimaryKeyType,
       InsertedAtType,
       TimestampType,
       NumericType,
   )
   from utils.models.general_models import Base
   from utils.models.schema_names import YOUR_SCHEMA_NAME
   ```

3. **Define Your Model Class**

   ```python
   # your_processor/models.py

   class YourEventModel(Base):
       __tablename__ = "your_event_table"
       __table_args__ = ({"schema": YOUR_SCHEMA_NAME},)

       sequence_number: BigIntegerPrimaryKeyType
       creation_number: BigIntegerPrimaryKeyType
       account_address: StringPrimaryKeyType
       # Add your custom fields here, for example:
       custom_field: StringType
       another_custom_field: NumericType
       transaction_version: BigIntegerType
       transaction_timestamp: TimestampType
       inserted_at: InsertedAtType
       event_index: BigIntegerType
   ```

**Notes:**

- **Schema Definition**: Ensure `__table_args__` includes your schema name (`YOUR_SCHEMA_NAME`), which should be defined in `utils/models/schema_names.py`.
- **Field Types**: Use the annotated types provided for consistency and compatibility.
- **Primary Keys**: Define primary key fields using types like `BigIntegerPrimaryKeyType` and `StringPrimaryKeyType`.
- **Custom Fields**: Add fields that correspond to the data you're extracting from events or transactions.

### 2. Processor Logic

Implement your processor logic in a `processor.py` file within your processor's directory. Extend the `TransactionsProcessor` class.

**Steps to Implement Processor Logic:**

1. **Create a `processor.py` File**

   In your processor directory, create a file named `processor.py`.

2. **Import Necessary Modules and Your Models**

   ```python
   # your_processor/processor.py

   from aptos_protos.aptos.transaction.v1 import transaction_pb2
   from utils.transactions_processor import TransactionsProcessor, ProcessingResult
   from utils.models.schema_names import YOUR_SCHEMA_NAME
   from utils.processor_name import ProcessorName
   from utils.session import Session
   from your_processor.models import YourEventModel
   import json
   from datetime import datetime
   from typing import List
   from utils import general_utils  # Ensure you have access to general utility functions
   ```

3. **Define Your Processor Class**

   ```python
   # your_processor/processor.py

   class YourCustomProcessor(TransactionsProcessor):
       def name(self) -> str:
           return ProcessorName.YOUR_PROCESSOR_NAME.value

       def schema(self) -> str:
           return YOUR_SCHEMA_NAME

       def process_transactions(
           self,
           transactions: List[transaction_pb2.Transaction],
           start_version: int,
           end_version: int,
       ) -> ProcessingResult:
           event_db_objs = []
           processing_start_time = datetime.utcnow()

           for transaction in transactions:
               # Filter for user transactions
               if transaction.type != transaction_pb2.Transaction.TRANSACTION_TYPE_USER:
                   continue

               user_txn = transaction.user
               txn_version = transaction.version
               txn_timestamp = general_utils.parse_pb_timestamp(transaction.timestamp)

               for event_index, event in enumerate(user_txn.events):
                   # Filter events
                   if not self.included_event_type(event.type_str):
                       continue

                   # Parse event data
                   event_data = json.loads(event.data)

                   # Create YourEventModel instance
                   event_obj = YourEventModel(
                       sequence_number=int(event.sequence_number),
                       creation_number=int(event.key.creation_number),
                       account_address=general_utils.standardize_address(event.key.account_address),
                       custom_field=event_data.get("custom_field"),
                       another_custom_field=event_data.get("another_custom_field"),
                       transaction_version=txn_version,
                       transaction_timestamp=txn_timestamp,
                       inserted_at=datetime.utcnow(),
                       event_index=event_index,
                   )
                   event_db_objs.append(event_obj)

           # Insert into the database
           self.insert_to_db(event_db_objs)

           processing_end_time = datetime.utcnow()
           processing_duration = (processing_end_time - processing_start_time).total_seconds()

           return ProcessingResult(
               start_version=start_version,
               end_version=end_version,
               processing_duration_in_secs=processing_duration,
               db_insertion_duration_in_secs=processing_duration,  # Adjust if needed
           )

       def insert_to_db(self, parsed_objs: List[YourEventModel]) -> None:
           with Session() as session:
               session.bulk_save_objects(parsed_objs)
               session.commit()

       @staticmethod
       def included_event_type(event_type: str) -> bool:
           # Implement your event filtering logic
           parsed_tag = event_type.split("::")
           module_address = general_utils.standardize_address(parsed_tag[0])
           module_name = parsed_tag[1]
           event_name = parsed_tag[2]

           return (
               module_address == "YOUR_MODULE_ADDRESS"
               and module_name == "YOUR_MODULE_NAME"
               and event_name == "YOUR_EVENT_NAME"
           )
   ```

**Notes:**

- **Event Filtering**: Implement efficient filtering in `included_event_type` to process only relevant events.
- **Data Parsing**: Extract and parse event data carefully, handling any possible data inconsistencies.
- **Database Insertion**: Use bulk insertion methods (`bulk_save_objects`) for efficiency.
- **Error Handling**: Include try-except blocks where appropriate to handle potential exceptions.
- **Utility Functions**: Use general utility functions like `standardize_address` and `parse_pb_timestamp` for consistency.

### 3. Configuration

Update the configuration files to include your new processor.

**Steps to Update Configuration:**

1. **Processor Names**

   Add your processor name to `utils/processor_name.py`:

   ```python
   # utils/processor_name.py

   from enum import Enum

   class ProcessorName(Enum):
       YOUR_PROCESSOR_NAME = "your_processor_name"
       # Other processors...
   ```

2. **Processor Registration**

   Register your processor in `utils/worker.py`:

   ```python
   # utils/worker.py

   from processors.your_processor.processor import YourCustomProcessor
   # Other imports...

   class IndexerProcessorServer:
       # Existing code...

       def __init__(self, config):
           self.config = config

           processor_name = self.config.processor_name
           if processor_name == ProcessorName.YOUR_PROCESSOR_NAME.value:
               self.processor = YourCustomProcessor()
           # Add other processors as needed
           else:
               raise ValueError(f"Unknown processor name: {processor_name}")
   ```

3. **Schema Names**

   Add your schema name to `utils/models/schema_names.py`:

   ```python
   # utils/models/schema_names.py

   YOUR_SCHEMA_NAME = "your_schema_name"
   # Other schema names...
   ```

---

## Running Your Processor

### 1. Prepare Your Database

- **Create the Schema** in your PostgreSQL database if it doesn't exist:

  ```sql
  CREATE SCHEMA your_schema_name;
  ```

- **Create Tables** using SQLAlchemy's `create_all` method or by running provided migration scripts.

### 2. Ensure Configuration is Correct

- **Update `config.yaml`** with the correct settings:

  ```yaml
  # config.yaml

  processor_name: "your_processor_name"
  chain_id: 1  # Mainnet (use 2 for Testnet, 3 for Devnet)
  indexer_grpc_data_service_address: "grpc.mainnet.aptoslabs.com:443"
  auth_token: "YOUR_API_KEY"
  postgres_connection_string: "postgresql://username:password@localhost:5432/your_processor_db"
  # Optional: starting_version and ending_version
  ```

- **Verify API Keys and Endpoints**.

### 3. Run the Processor

```bash
poetry run python -m processors.main -c config.yaml
```

- **Logs**: Monitor the console output for any errors or processing status.
- **Database**: Check your database to see if data is being inserted as expected.

---

## Docker Deployment

To run your processor in a Docker container:

### 1. Build the Docker Image

- **Update the `Dockerfile`** if necessary to include your processor.

- **Build the image**:

  ```bash
  docker build -t your_processor_image .
  ```

### 2. Update Docker Compose Configuration

- **Modify `docker-compose.yml`** to include your service and database configurations.

**Example:**

```yaml
version: '3'
services:
  your_processor:
    build: .
    environment:
      POSTGRES_CONNECTION_STRING: "postgresql://username:password@db:5432/your_processor_db"
    depends_on:
      - db
    volumes:
      - ./config.yaml:/app/config/config.yaml
  db:
    image: postgres:15.2
    environment:
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data
volumes:
  db-data:
```

### 3. Run the Services

```bash
docker-compose up --build
```

- **Check Logs**: Use `docker-compose logs` to monitor the services.
- **Verify Data**: Ensure data is being processed and stored correctly.

---

## Best Practices

1. **Event Filtering**

   - Implement precise event filtering to process only relevant events.
   - Use configuration or constants for module addresses and event names.

2. **Error Handling**

   - Wrap critical sections in try-except blocks.
   - Log exceptions with detailed information.

3. **Performance Optimization**

   - Use bulk database operations.
   - Optimize database indexes and schemas.
   - Monitor resource usage.

4. **Logging and Monitoring**

   - Use structured logging (e.g., JSON format).
   - Integrate monitoring tools to track processor performance and health.

5. **Configuration Management**

   - Use environment variables or a secrets manager for sensitive data.
   - Maintain separate configurations for development, testing, and production environments.

6. **Testing**

   - Write unit tests for your processor logic.
   - Use a local Aptos network for integration testing.

7. **Documentation**

   - Document your code and processor logic.
   - Maintain a README in your processor's directory explaining its purpose and usage.

---

## Event Filtering and Processing

Efficient event filtering is crucial for processor performance, as it reduces unnecessary processing and resource consumption.

### Tips for Effective Event Filtering

- **Be Specific**: Filter events by matching the exact module address, module name, and event name.

- **Use Constants or Configuration**: Define module addresses and event names as constants or in configuration files to improve readability and maintainability.

  ```python
  # your_processor/processor.py

  MODULE_ADDRESS = general_utils.standardize_address("0xYourModuleAddress")
  MODULE_NAME = "YourModuleName"
  EVENT_NAME = "YourEventName"
  ```

- **Process Multiple Event Types**: If your processor handles multiple event types, consider using a set or dictionary for faster lookups.

  ```python
  # Define a set of event types to include
  INCLUDED_EVENTS = {
      (MODULE_ADDRESS, "Module1", "Event1"),
      (MODULE_ADDRESS, "Module2", "Event2"),
  }

  @staticmethod
  def included_event_type(event_type: str) -> bool:
      parsed_tag = event_type.split("::")
      module_address = general_utils.standardize_address(parsed_tag[0])
      module_name = parsed_tag[1]
      event_name = parsed_tag[2]
      return (module_address, module_name, event_name) in INCLUDED_EVENTS
  ```

- **Avoid Hardcoding**: Use configuration files or environment variables to store module addresses and names, allowing for easier updates.

- **Logging**: Log unmatched events if needed for debugging.

  ```python
  import logging

  logger = logging.getLogger(__name__)

  @staticmethod
  def included_event_type(event_type: str) -> bool:
      # ... existing code ...
      else:
          logger.debug(f"Excluded event type: {event_type}")
          return False
  ```

---

## Troubleshooting

- **GRPC Deserialization Errors**

  - If you encounter gRPC deserialization recursion limits with Python, consider increasing the recursion limit:

    ```python
    import sys
    sys.setrecursionlimit(10000)
    ```

  - **Recommendation**: For production workloads, switch to Rust for better performance and stability.

- **Database Connection Issues**

  - Ensure your `postgres_connection_string` in `config.yaml` is correct.
  - Verify that the PostgreSQL service is running and accessible.

- **GRPC Endpoint Connectivity**

  - Check that the `indexer_grpc_data_service_address` is correct and that you have internet connectivity.
  - Ensure your `auth_token` (API key) is valid and not expired.

- **Missing Data or Processing Gaps**

  - Verify that your event filtering logic is correct.
  - Check logs for any errors during processing that may have caused skips.

---

## Additional Resources

- **Aptos Developer Documentation**: [https://aptos.dev/](https://aptos.dev/)
- **Aptos GitHub Repositories**: [https://github.com/aptos-labs](https://github.com/aptos-labs)
- **SQLAlchemy Documentation**: [https://docs.sqlalchemy.org/](https://docs.sqlalchemy.org/)
- **gRPC Python Tutorial**: [https://grpc.io/docs/languages/python/](https://grpc.io/docs/languages/python/)

---

By following this guide, you should be able to set up your development environment and create custom processors for Aptos Indexers using the `aptos-indexer-processors` repository. Remember to stay updated with the latest changes in the Aptos ecosystem and consult the official documentation for any updates.

---

**Disclaimer:** The information provided in this guide is based on the state of the Aptos Indexer as of October 2023. Please refer to the official Aptos documentation and repositories for the most current information.

---