# Developing Aptos Indexer Processors in Rust

## Introduction

This guide provides a comprehensive walkthrough for developing custom Aptos Indexer processors using **Rust**. Aptos Indexers are powerful tools designed to aggregate, process, and query data from the Aptos blockchain, enabling developers to efficiently access and analyze on-chain data for decentralized applications (dApps).

We will cover:

- Setting up the Rust development environment.
- Creating custom processors in Rust, including database models, processor logic, and event filtering.
- Best practices for efficient and reliable indexing.
- Running and testing your processor.
- Troubleshooting common issues.

By the end of this guide, you'll have the foundational knowledge to build and customize your own Aptos Indexer processors in Rust.

---

## Prerequisites

Before you begin, ensure you have the following installed:

- **Rust** (version 1.79): [Installation Guide](https://www.rust-lang.org/tools/install)
- **Cargo**: Rust's package manager (included with Rust)
- **Diesel CLI**: [Installation Guide](https://diesel.rs/guides/getting-started.html)
- **PostgreSQL**: [Download](https://www.postgresql.org/download/)
- **psql**: PostgreSQL command-line tool
- **Aptos Indexer API Key**: Obtain from the [Aptos Developer Portal](https://aptos.dev)

---

## Setting Up the Development Environment

### Clone the Example Repository

We'll use the [Aptos Indexer Processor Example](https://github.com/aptos-labs/aptos-indexer-processor-example) as a starting point:

```bash
git clone https://github.com/aptos-labs/aptos-indexer-processor-example.git
```

### Install Rust and Cargo

Follow the [Rust Installation Guide](https://www.rust-lang.org/tools/install) to install Rust and Cargo.

### Install Diesel CLI

Install Diesel CLI to manage database migrations:

```bash
cargo install diesel_cli --no-default-features --features postgres
```

### Set Up PostgreSQL

1. **Install PostgreSQL**:

   - **Ubuntu/Linux**:

     ```bash
     sudo apt-get update
     sudo apt-get install postgresql postgresql-contrib
     ```

   - **macOS (Homebrew)**:

     ```bash
     brew install postgresql
     ```

2. **Start PostgreSQL Service**:

   - **Ubuntu/Linux**:

     ```bash
     sudo service postgresql start
     ```

   - **macOS**:

     ```bash
     brew services start postgresql
     ```

3. **Create a Database**:

   - Access PostgreSQL shell:

     ```bash
     psql -U postgres
     ```

   - Create a new database:

     ```sql
     CREATE DATABASE example;
     ```

   - Exit `psql`:

     ```sql
     \q
     ```

---

## Configuring Your Processor

### Prepare the `config.yaml` File

Navigate to the cloned repository and copy the example configuration file. Edit `config.yaml` with your preferred text editor:

```yaml
health_check_port: 8085
server_config:
  processor_config:
    type: "events_processor"
  transaction_stream_config:
    indexer_grpc_data_service_address: "https://grpc.testnet.aptoslabs.com:443"
    starting_version: 0
    # request_ending_version: 10000
    auth_token: "YOUR_API_KEY"
    request_name_header: "events-processor"
  db_config:
    postgres_connection_string: "postgresql://postgres:@localhost:5432/example"
```

Update the following:

- **`auth_token`**: Replace `"YOUR_API_KEY"` with your Aptos Transaction Streaming Service API Key.
- **`postgres_connection_string`**: Ensure it matches your PostgreSQL setup. If you have a password, include it like `postgresql://postgres:your_password@localhost:5432/example`.

### Customize Configuration (Optional)

- **Starting Version**: Set `starting_version` to a specific ledger version if you don't want to start from 0.
- **Ending Version**: Uncomment and set `request_ending_version` to stop processing at a specific version.
- **Network Selection**: Change `indexer_grpc_data_service_address` to switch between Testnet, Devnet, or Mainnet.

---

## Understanding the Processor Structure

An Aptos Indexer processor typically follows these steps:

1. **Transaction Stream**: Receives a stream of transactions from the Aptos blockchain.
2. **Data Extraction**: Parses transactions to extract relevant data (e.g., events).
3. **Data Transformation**: Processes and transforms the extracted data.
4. **Data Storage**: Inserts the transformed data into a database.
5. **Version Tracking**: Keeps track of the last processed transaction version.

---

## Creating Custom Processors in Rust

### Defining Database Models

Database models represent the structure of the data to be stored. In Rust, we use Diesel ORM to define these models.

#### Event Model (`Event` Struct)

```rust
use crate::schema::events;
use diesel::{Identifiable, Insertable};
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Deserialize, Identifiable, Insertable, Serialize)]
#[diesel(primary_key(transaction_version, event_index))]
#[diesel(table_name = events)]
pub struct Event {
    pub sequence_number: i64,
    pub creation_number: i64,
    pub account_address: String,
    pub transaction_version: i64,
    pub transaction_block_height: i64,
    pub type_: String,
    pub data: serde_json::Value,
    pub event_index: i64,
    pub indexed_type: String,
}
```

**Explanation**:

- `#[derive(...)]`: Implements traits for serialization, database operations, etc.
- `#[diesel(...)]`: Diesel-specific annotations for primary keys and table names.

### Creating Database Migrations

Database migrations are used to create and modify tables.

1. **Generate a New Migration**:

   ```bash
   diesel migration generate create_events_table
   ```

2. **Edit Migration Files**:

   - **`up.sql`**: Define the changes to apply (e.g., create table).
   - **`down.sql`**: Define how to revert the changes.

3. **Run Migrations**:

   ```bash
   diesel migration run --database-url="postgresql://postgres:@localhost:5432/example"
   ```

4. **Update Schema**:

   Diesel automatically updates `src/db/postgres/schema.rs` to reflect the current database schema.

### Implementing Processor Logic

#### Processor Steps

Each processor step is an independent unit that performs a specific function.

1. **Events Extractor (`EventsExtractor`)**:

   - Extracts events from transactions.
   - Implements `Processable`, `AsyncStep`, and `NamedStep` traits.

   ```rust
   use aptos_indexer_processor_sdk::{
       traits::{async_step::AsyncRunType, AsyncStep, NamedStep, Processable},
       types::transaction_context::TransactionContext,
   };
   use async_trait::async_trait;

   pub struct EventsExtractor;

   #[async_trait]
   impl Processable for EventsExtractor {
       type Input = Transaction;
       type Output = EventModel;
       type RunType = AsyncRunType;

       async fn process(
           &mut self,
           item: TransactionContext<Transaction>,
       ) -> Result<Option<TransactionContext<EventModel>>, ProcessorError> {
           // Extraction logic here
       }
   }

   impl AsyncStep for EventsExtractor {}

   impl NamedStep for EventsExtractor {
       fn name(&self) -> String {
           "EventsExtractor".to_string()
       }
   }
   ```

2. **Events Storer (`EventsStorer`)**:

   - Stores extracted events into the database.

   ```rust
   pub struct EventsStorer {
       conn_pool: ArcDbPool,
   }

   impl EventsStorer {
       pub fn new(conn_pool: ArcDbPool) -> Self {
           Self { conn_pool }
       }
   }

   #[async_trait]
   impl Processable for EventsStorer {
       type Input = EventModel;
       type Output = EventModel;
       type RunType = AsyncRunType;

       async fn process(
           &mut self,
           events: TransactionContext<EventModel>,
       ) -> Result<Option<TransactionContext<EventModel>>, ProcessorError> {
           // Database insertion logic here
       }
   }

   impl AsyncStep for EventsStorer {}

   impl NamedStep for EventsStorer {
       fn name(&self) -> String {
           "EventsStorer".to_string()
       }
   }
   ```

3. **Version Tracker (`LatestVersionProcessedTracker`)**:

   - Tracks the latest processed transaction version.

#### Connecting Steps with `ProcessorBuilder`

```rust
let (_, buffer_receiver) = ProcessorBuilder::new_with_inputless_first_step(
    transaction_stream.into_runnable_step(),
)
.connect_to(events_extractor.into_runnable_step(), 10)
.connect_to(events_storer.into_runnable_step(), 10)
.connect_to(version_tracker.into_runnable_step(), 10)
.end_and_return_output_receiver(10);
```

**Explanation**:

- **`transaction_stream`**: The initial step that streams transactions.
- **`connect_to`**: Chains steps together, passing output from one to the next.
- **`buffer_receiver`**: Receives the final output for optional further processing.

### Customizing the Processor

To tailor the processor to your needs:

- **Filter Specific Events**: Modify `EventsExtractor` to extract only events of interest.

  ```rust
  if event.type_ == "0x1::YourModule::YourEvent" {
      // Process your custom event
  }
  ```

- **Extend Database Models**: Add fields to `Event` struct and update migrations.

- **Implement Additional Logic**: Create new steps for data transformation or enrichment.

---

## Best Practices for Efficient and Reliable Indexing

- **Chunk Inserts**: Insert data in chunks to prevent exceeding database limits.

  ```rust
  let chunk_size = 100;
  for chunk in items.chunks(chunk_size) {
      // Insert chunk into database
  }
  ```

- **Error Handling**: Implement robust error handling and retries.

- **Asynchronous Processing**: Utilize Rust's async features for non-blocking operations.

- **Connection Pooling**: Use a connection pool to manage database connections efficiently.

- **Data Sanitization**: Clean data before insertion to prevent injection attacks and data corruption.

---

## Running and Testing the Processor

### Build and Run the Processor

Navigate to the root of the cloned repository and execute:

```bash
cargo run --release -- -c config.yaml
```

**Explanation**:

- **`cargo run --release`**: Builds and runs the processor in release mode for better performance.
- **`-c config.yaml`**: Specifies the configuration file.

### Monitor the Output

You should see logs indicating progress:

```plaintext
[Transaction Stream] Received transactions from GRPC.
Events version [0, 4999] stored successfully
Finished processing events from versions [0, 4999]
```

### Verify Database Entries

Use `psql` or a GUI tool to query the `events` table and verify data insertion.

```sql
SELECT * FROM events LIMIT 10;
```

---

## Troubleshooting Common Issues

### Database Connection Errors

- **Ensure PostgreSQL is Running**: Start the PostgreSQL service.
- **Verify Connection String**: Check `postgres_connection_string` in `config.yaml`.

### API Key Errors

- **Invalid API Key**: Ensure the `auth_token` is correct and not expired.
- **Permissions**: Verify that your API Key has the necessary permissions.

### Migration Failures

- **Pending Migrations**: Run `diesel migration run` to apply all migrations.
- **Schema Mismatch**: Ensure your Rust models match the database schema.

### Data Insertion Errors

- **Data Sanitization**: Remove null bytes and sanitize strings.
- **Chunk Size**: Adjust chunk sizes to avoid exceeding parameter limits.

### Performance Issues

- **Connection Pool Size**: Increase `db_pool_size` in `config.yaml` if necessary.
- **Optimize Queries**: Ensure database queries are efficient.

---

## Conclusion

You've now learned how to set up, customize, and run an Aptos Indexer processor in Rust. By leveraging the Aptos Indexer SDK and following best practices, you can efficiently index blockchain data tailored to your application's needs.

---

## References

- **Aptos Indexer SDK**: [GitHub Repository](https://github.com/aptos-labs/aptos-indexer-processor-sdk)
- **Aptos Indexer Processor Example**: [GitHub Repository](https://github.com/aptos-labs/aptos-indexer-processor-example)
- **Diesel ORM Documentation**: [Official Site](https://diesel.rs)
- **Rust Programming Language**: [Official Site](https://www.rust-lang.org)

---

**Note**: The information provided in this guide is based on the Aptos Indexer SDK and the example processor provided by the Aptos team. For the latest updates and detailed documentation, refer to the official Aptos repositories and documentation.