# Developing Aptos Indexer Processors in TypeScript

This guide provides detailed instructions on developing custom Aptos Indexer processors using TypeScript. It covers setting up the development environment, creating custom processors, and best practices for efficient and reliable indexing.

---

**Note:** While this guide focuses on TypeScript, it's important to mention that for production-grade indexers, the **Rust** implementation is recommended due to better performance and ecosystem support. The TypeScript implementation is known to have issues when processing large volumes of data and may get stuck. Proceed with caution for production environments.

---

## Overview

Aptos Indexers allow developers to process and store on-chain data from the Aptos blockchain, enabling efficient querying and analysis for decentralized applications (dApps). By creating custom processors, developers can tailor the indexing process to their specific data needs.

The TypeScript implementation provides a straightforward way to write custom processors using familiar tools and libraries.

## Prerequisites

- **Node.js**: Tested with Node.js 18. Later versions should work as well.
- **pnpm**: Preferred package manager. Tested with pnpm 8.6.2.
- **TypeScript**: Development in TypeScript requires familiarity with the language.
- **PostgreSQL**: Database for storing indexed data.
- **API Key**: An API key from the Aptos Developer Portal for accessing the Transaction Stream Service.

## Repository Structure

The example processor (`event_processor`) has the following structure:

```
event_processor
├── src
│   ├── index.ts       # Entry point
│   ├── models.ts      # Database models
│   ├── processor.ts   # Processor logic
├── package.json
├── config.yaml.example
├── tsconfig.json
├── README.md
```

## Development Environment Setup

### 1. Clone the Repository

Clone the `aptos-indexer-processors` repository and navigate to the TypeScript example:

```bash
git clone https://github.com/aptos-labs/aptos-indexer-processors.git
cd aptos-indexer-processors/typescript/event_processor
```

### 2. Install Dependencies

Install the required packages using `pnpm`:

```bash
pnpm install
```

### 3. Set Up PostgreSQL

- **Install PostgreSQL** if not already installed.
- **Start the PostgreSQL service**.
- **Create a database** for your processor (e.g., `event_processor_db`).

### 4. Obtain an API Key

Get an API key from the [Aptos Developer Portal](https://developers.aptoslabs.com/) to access the Transaction Stream Service.

### 5. Configure Your Environment

- **Copy the example configuration**:

  ```bash
  cp config.yaml.example config.yaml
  ```

- **Update `config.yaml`** with your settings:

  ```yaml
  chain_id: 2  # Use 1 for Mainnet, 2 for Testnet, 3 for Devnet
  grpc_data_stream_endpoint: grpc.testnet.aptoslabs.com:443
  grpc_data_stream_api_key: "YOUR_API_KEY"
  starting_version: 0
  db_connection_uri: "postgresql://username:password@localhost:5432/event_processor_db"
  ```

---

## Creating a Custom Processor

To create a custom processor, you need to implement two main components:

1. **Database Models**
2. **Processor Logic**

### 1. Database Models

Define your database models in `models.ts` using TypeORM.

**Example `models.ts`:**

```typescript
// src/models.ts

import { Base } from "@aptos-labs/aptos-processor-sdk";
import { Column, Entity, PrimaryColumn } from "typeorm";

@Entity("events")
export class Event extends Base {
  @PrimaryColumn({ type: "bigint" })
  transactionVersion!: string;

  @PrimaryColumn({ type: "bigint" })
  eventIndex!: string;

  @Column({ type: "bigint" })
  creationNumber!: string;

  @Column()
  accountAddress!: string;

  @Column({ type: "bigint" })
  sequenceNumber!: string;

  @Column()
  type!: string;

  @Column({ type: "bigint" })
  transactionBlockHeight!: string;

  @Column()
  data!: string;

  @Column({ type: "timestamptz", nullable: true })
  insertedAt!: Date;
}
```

**Notes:**

- **Entity Definition**: Use the `@Entity` decorator to define a database table.
- **Columns and Types**: Define columns using the `@Column` and `@PrimaryColumn` decorators, specifying the data types.
- **Base Class**: Extend from `Base` provided by the SDK to include standard fields and functionality.

### 2. Processor Logic

Implement your processor logic in `processor.ts` by extending the `TransactionsProcessor` class from the SDK.

**Example `processor.ts`:**

```typescript
// src/processor.ts

import { protos, TransactionsProcessor, ProcessingResult, grpcTimestampToDate } from "@aptos-labs/aptos-processor-sdk";
import { Event } from "./models";
import { DataSource } from "typeorm";

export class EventProcessor extends TransactionsProcessor {
  name(): string {
    return "event_processor";
  }

  async processTransactions(params: {
    transactions: protos.aptos.transaction.v1.Transaction[];
    startVersion: bigint;
    endVersion: bigint;
    dataSource: DataSource;
  }): Promise<ProcessingResult> {
    const { transactions, startVersion, endVersion, dataSource } = params;
    let allEvents: Event[] = [];

    // Process transactions
    for (const transaction of transactions) {
      // Filter user transactions
      if (
        transaction.type !==
        protos.aptos.transaction.v1.Transaction_TransactionType.TRANSACTION_TYPE_USER
      ) {
        continue;
      }

      const transactionVersion = transaction.version!.toString();
      const transactionBlockHeight = transaction.blockHeight!.toString();
      const insertedAt = grpcTimestampToDate(transaction.timestamp!);

      const userTransaction = transaction.user!;
      const events = userTransaction.events!;

      events.forEach((event, index) => {
        const eventEntity = new Event();
        eventEntity.transactionVersion = transactionVersion;
        eventEntity.eventIndex = index.toString();
        eventEntity.sequenceNumber = event.sequenceNumber!.toString();
        eventEntity.creationNumber = event.key!.creationNumber!.toString();
        eventEntity.accountAddress = `0x${event.key!.accountAddress}`;
        eventEntity.type = event.typeStr!;
        eventEntity.data = event.data!;
        eventEntity.transactionBlockHeight = transactionBlockHeight;
        eventEntity.insertedAt = insertedAt;

        allEvents.push(eventEntity);
      });
    }

    // Insert events into the database
    await dataSource.transaction(async (transactionalEntityManager) => {
      const chunkSize = 100;
      for (let i = 0; i < allEvents.length; i += chunkSize) {
        const chunk = allEvents.slice(i, i + chunkSize);
        await transactionalEntityManager.insert(Event, chunk);
      }
    });

    return {
      startVersion,
      endVersion,
    };
  }
}
```

**Notes:**

- **Processor Class**: Extend `TransactionsProcessor` and implement the `processTransactions` method.
- **Name**: Provide a unique name for your processor using the `name` method.
- **Transaction Processing**:
  - Filter for user transactions.
  - Extract relevant data from each transaction and its events.
  - Create instances of your model (`Event`) with the extracted data.
- **Database Insertion**:
  - Use transactions for database operations to ensure atomicity.
  - Insert data in chunks to handle large volumes efficiently.
- **Processing Result**: Return a `ProcessingResult` object indicating the range of processed versions.

---

## Running Your Processor

### 1. Prepare the Database

- Ensure your PostgreSQL database is running.
- Create the necessary tables using TypeORM's synchronization feature or migrations.

### 2. Run the Processor

Use the provided `index.ts` as the entry point.

**Example `index.ts`:**

```typescript
// src/index.ts

import { program } from "commander";
import { Config, Worker } from "@aptos-labs/aptos-processor-sdk";
import { EventProcessor } from "./processor";
import { Event } from "./models";

program
  .command("process")
  .requiredOption("--config <config>", "Path to a YAML config file")
  .action(async (options) => {
    const config = Config.from_yaml_file(options.config);
    const processor = new EventProcessor();
    const worker = new Worker({
      config,
      processor,
      models: [Event],
    });
    await worker.run();
  });

program.parse();
```

Run the processor using the following command:

```bash
pnpm start process --config ./config.yaml
```

**Notes:**

- **Command Line Interface**: Uses `commander` to parse command-line options.
- **Configuration**: Loads configuration from the specified YAML file.
- **Worker Initialization**: Creates a `Worker` instance with the processor and models.
- **Execution**: Calls `worker.run()` to start processing transactions.

### 3. Verify Processing

- Monitor the console output for any errors or processing logs.
- Check the database to confirm that events are being stored correctly.

---

## Best Practices

1. **Event Filtering**

   - Implement precise event filtering within your processor to process only relevant events.
   - Modify the `processTransactions` method to include custom filtering logic based on your needs.

2. **Error Handling**

   - Include try-catch blocks to handle potential exceptions during processing.
   - Log errors with detailed information for easier debugging.

3. **Performance Optimization**

   - Use bulk insertion methods and process data in chunks to handle large volumes efficiently.
   - Avoid processing unnecessary transactions by filtering early.

4. **Logging and Monitoring**

   - Use a logging library to capture and store logs systematically.
   - Monitor processor performance and resource usage.

5. **Configuration Management**

   - Store sensitive data like API keys securely (e.g., environment variables or a secrets manager).
   - Maintain separate configurations for different environments (development, testing, production).

6. **Testing**

   - Write unit tests for your processor logic using a testing framework like Jest.
   - Test with a local Aptos network if possible to simulate real-world scenarios.

7. **Documentation**

   - Document your code thoroughly, including comments and README files.
   - Explain the purpose of your processor, how to set it up, and how to run it.

---

## Troubleshooting

- **Processor Getting Stuck**

  - The TypeScript implementation may get stuck when processing large amounts of data.
  - Consider processing smaller data ranges or switching to the Rust implementation for better performance.

- **GRPC Connectivity Issues**

  - Ensure the `grpc_data_stream_endpoint` in `config.yaml` is correct.
  - Verify that your API key is valid and has the necessary permissions.

- **Database Connection Errors**

  - Confirm that the `db_connection_uri` in `config.yaml` is correct.
  - Ensure that PostgreSQL is running and accessible.

- **Missing Data**

  - Check your event filtering logic to ensure relevant events are being processed.
  - Review logs for any errors that might have occurred during processing.

---

## Additional Tips

- **TypeORM Migrations**

  - Use TypeORM migrations to manage database schema changes over time.
  - This ensures consistency across different environments and simplifies deployment.

- **Handling Large Data Volumes**

  - Implement pagination or batching when processing transactions.
  - Consider setting `starting_version` and `ending_version` in your configuration to limit the processing range.

- **Using Environment Variables**

  - Replace hardcoded values in your configuration with environment variables for flexibility and security.

  ```yaml
  # config.yaml
  grpc_data_stream_api_key: ${GRPC_API_KEY}
  db_connection_uri: ${DATABASE_URL}
  ```

- **Docker Deployment**

  - Containerize your processor using Docker for consistent deployment across environments.
  - Ensure that your Dockerfile includes all necessary dependencies and configurations.

---

## Conclusion

By following this guide, you should be able to set up your development environment and create custom Aptos Indexer processors using TypeScript. Remember to monitor performance and be cautious when processing large volumes of data due to known issues with the TypeScript implementation.

---

## Additional Resources

- **Aptos Developer Documentation**: [https://aptos.dev/](https://aptos.dev/)
- **Aptos GitHub Repositories**: [https://github.com/aptos-labs](https://github.com/aptos-labs)
- **TypeORM Documentation**: [https://typeorm.io/](https://typeorm.io/)
- **gRPC Node.js Guide**: [https://grpc.io/docs/languages/node/basics/](https://grpc.io/docs/languages/node/basics/)
- **TypeScript Documentation**: [https://www.typescriptlang.org/docs/](https://www.typescriptlang.org/docs/)

---

**Disclaimer:** The information provided in this guide is based on the state of the Aptos Indexer as of October 2023. Please refer to the official Aptos documentation and repositories for the most current information.

---