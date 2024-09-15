# Custom Processor Development for Aptos Indexers

This guide covers the process of developing custom processors for Aptos Indexers, including creating database models, implementing processor logic, event filtering, error handling, and testing.

## Overview

Custom processors are the core components of Aptos Indexers, responsible for processing blockchain data and storing it in a structured format. They allow developers to extract specific data from the Aptos blockchain based on their application needs.

## Creating Database Models

Database models define the structure of the data you want to store. Aptos Indexers use SQLAlchemy ORM for database interactions.

### Steps to Create a Database Model

1. Create a `models.py` file in your processor directory.
2. Import necessary types and base classes:

```python
from utils.models.annotated_types import (
    BooleanType, StringPrimaryKeyType, BigIntegerType,
    BigIntegerPrimaryKeyType, InsertedAtType, TimestampType, NumericType
)
from utils.models.general_models import Base
from utils.models.schema_names import YOUR_SCHEMA_NAME
```

3. Define your model class:

```python
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

Ensure that your model includes all necessary fields to capture the data you're interested in from the blockchain events.

## Implementing Processor Logic

The processor logic is where you define how to extract and process data from blockchain transactions.

### Steps to Implement Processor Logic

1. Create a `processor.py` file in your processor directory.
2. Import necessary modules and your models:

```python
from aptos_protos.aptos.transaction.v1 import transaction_pb2
from utils.transactions_processor import TransactionsProcessor, ProcessingResult
from utils.models.schema_names import YOUR_SCHEMA_NAME
from utils.processor_name import ProcessorName
from utils.session import Session
from your_processor.models import YourEventModel
import json
from datetime import datetime
```

3. Define your processor class:

```python
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
            if transaction.type != transaction_pb2.Transaction.TRANSACTION_TYPE_USER:
                continue

            for event_index, event in enumerate(transaction.user.events):
                if not self.included_event_type(event.type_str):
                    continue

                # Process the event and create a YourEventModel instance
                event_data = json.loads(event.data)
                event_db_obj = YourEventModel(
                    sequence_number=event.sequence_number,
                    creation_number=event.key.creation_number,
                    account_address=event.key.account_address,
                    transaction_version=transaction.version,
                    transaction_timestamp=transaction.timestamp,
                    # Add your custom field assignments here
                    inserted_at=datetime.now(),
                    event_index=event_index,
                )
                event_db_objs.append(event_db_obj)

        self.insert_to_db(event_db_objs)
        return ProcessingResult(
            start_version=start_version,
            end_version=end_version,
            # Add processing and insertion durations
        )

    def insert_to_db(self, parsed_objs: List[YourEventModel]) -> None:
        with Session() as session, session.begin():
            for obj in parsed_objs:
                session.merge(obj)

    @staticmethod
    def included_event_type(event_type: str) -> bool:
        # Implement your event filtering logic
        parsed_tag = event_type.split("::")
        return (
            parsed_tag[0] == "YOUR_MODULE_ADDRESS" and
            parsed_tag[1] == "YOUR_MODULE_NAME" and
            parsed_tag[2] == "YOUR_EVENT_NAME"
        )
```

## Event Filtering and Processing

Efficient event filtering is crucial for processor performance. The `included_event_type` method allows you to filter events based on their type.

### Tips for Effective Event Filtering

- Be as specific as possible to reduce unnecessary processing.
- Consider using constants for addresses and event names to improve readability and maintainability.
- If processing multiple event types, consider using a set or dictionary for faster lookups.

## Error Handling and Optimization

Robust error handling and optimization are essential for reliable and efficient processors.

### Error Handling Best Practices

1. Use try-except blocks to catch and handle specific exceptions:

```python
try:
    # Process event data
except json.JSONDecodeError:
    logging.error(f"Failed to decode event data: {event.data}")
except KeyError as e:
    logging.error(f"Missing expected key in event data: {e}")
```

2. Log errors with sufficient context for debugging.
3. Consider implementing retries for transient errors, especially in database operations.

### Optimization Techniques

1. Use bulk inserts instead of individual inserts when possible:

```python
session.bulk_save_objects(event_db_objs)
```

2. Implement batching for processing large numbers of transactions.
3. Use database indexing strategically on frequently queried fields.
4. Profile your processor to identify performance bottlenecks.

## Testing Custom Processors

Thorough testing is crucial for ensuring the reliability and correctness of your custom processor.

### Testing Strategy

1. Unit Tests: Test individual methods of your processor class.
2. Integration Tests: Test the entire processing pipeline with sample transaction data.
3. Performance Tests: Ensure your processor can handle expected transaction volumes.

### Example Test Case

```python
import unittest
from your_processor.processor import YourCustomProcessor
from aptos_protos.aptos.transaction.v1 import transaction_pb2

class TestYourCustomProcessor(unittest.TestCase):
    def setUp(self):
        self.processor = YourCustomProcessor()

    def test_included_event_type(self):
        valid_event_type = "YOUR_MODULE_ADDRESS::YOUR_MODULE_NAME::YOUR_EVENT_NAME"
        invalid_event_type = "INVALID_ADDRESS::INVALID_MODULE::INVALID_EVENT"

        self.assertTrue(self.processor.included_event_type(valid_event_type))
        self.assertFalse(self.processor.included_event_type(invalid_event_type))

    def test_process_transactions(self):
        # Create a mock transaction with known data
        mock_transaction = transaction_pb2.Transaction()
        # Populate mock_transaction with test data

        result = self.processor.process_transactions([mock_transaction], 1, 1)

        # Assert the expected results
        self.assertEqual(result.start_version, 1)
        self.assertEqual(result.end_version, 1)
        # Add more assertions based on expected behavior

if __name__ == '__main__':
    unittest.main()
```

## Conclusion

Developing custom processors for Aptos Indexers involves creating well-structured database models, implementing efficient processing logic, handling errors gracefully, and thoroughly testing your implementation. By following these guidelines and best practices, you can create robust and performant processors that effectively index the data you need from the Aptos blockchain.