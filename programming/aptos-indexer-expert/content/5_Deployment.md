# Deployment and Scaling of Aptos Indexers

This guide covers the deployment process for Aptos Indexer processors, scaling considerations for high-volume indexing, monitoring and logging practices, and strategies for upgrading processors and handling schema changes.

## Deploying Processors

Aptos Indexer processors can be deployed in various environments, from local development setups to cloud-based production systems. Here are the main deployment options:

### Local Deployment

Local deployment is useful for development and testing purposes.

1. Ensure your environment is set up with all prerequisites (Python, PostgreSQL, etc.).
2. Configure your `config.yaml` file with local database and GRPC endpoint settings.
3. Run your processor:

   ```
   poetry run python -m processors.main -c config.yaml
   ```

### Containerized Deployment with Docker

Containerization allows for consistent deployment across different environments.

1. Ensure Docker and Docker Compose are installed on your system.
2. Use the provided `Dockerfile` and `docker-compose.yml` in the repository.
3. Build and run your Docker container:

   ```
   docker compose up --build --force-recreate
   ```

### Cloud Deployment

For production environments, cloud deployment offers scalability and reliability.

1. Choose a cloud provider (e.g., AWS, Google Cloud, Azure).
2. Set up a virtual machine or container orchestration service (e.g., Kubernetes).
3. Deploy your processor using containerization or directly on the VM.
4. Ensure your database is properly set up and accessible from your cloud environment.

Example using Google Cloud Run:

```bash
# Build the container
docker build -t gcr.io/[PROJECT-ID]/aptos-indexer-processor .

# Push to Google Container Registry
docker push gcr.io/[PROJECT-ID]/aptos-indexer-processor

# Deploy to Cloud Run
gcloud run deploy aptos-indexer-processor \
  --image gcr.io/[PROJECT-ID]/aptos-indexer-processor \
  --platform managed \
  --region [REGION] \
  --set-env-vars="DB_CONNECTION_URI=your-db-uri"
```

## Scaling Considerations

As the volume of transactions on the Aptos blockchain grows, your indexer needs to scale accordingly.

### Vertical Scaling

- Increase CPU and memory resources for your processor and database.
- Useful for handling increased load up to a certain point.

### Horizontal Scaling

- Run multiple instances of your processor, each handling a subset of transactions.
- Requires logic to distribute work among processor instances.

Example of horizontal scaling logic:

```python
def should_process_transaction(self, transaction_version: int) -> bool:
    return transaction_version % self.total_processors == self.processor_index
```

### Database Scaling

- Use database read replicas to handle increased query load.
- Consider database sharding for extremely large datasets.

### Optimizing Processor Performance

- Implement batch processing to reduce database write operations.
- Use efficient data structures and algorithms in your processor logic.

## Monitoring and Logging

Proper monitoring and logging are crucial for maintaining healthy indexer operations.

### Logging

Implement comprehensive logging in your processor:

```python
import logging

logging.info(f"Processing transaction version: {transaction.version}")
logging.error(f"Error processing event: {str(e)}")
```

### Metrics

Track key metrics to monitor your processor's performance:

- Transactions processed per second
- Event processing time
- Database insertion time
- Error rates

Consider using a monitoring solution like Prometheus with Grafana for visualization.

### Alerting

Set up alerts for critical issues:

- Processor falling behind real-time blockchain state
- High error rates
- Database connection issues

## Upgrading Processors and Handling Schema Changes

Careful planning is needed when upgrading processors or changing database schemas.

### Processor Upgrades

1. Develop and test the new processor version thoroughly.
2. Deploy the new version alongside the old one.
3. Gradually shift traffic to the new version.
4. Monitor for any issues and be prepared to rollback if necessary.

### Schema Changes

When changing your database schema:

1. Use database migration tools (e.g., Alembic for SQLAlchemy).
2. Plan for backward compatibility if possible.
3. For major changes, consider:
   - Running both old and new schemas in parallel during transition.
   - Implementing a data backfill process for the new schema.

Example Alembic migration:

```python
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.add_column('your_event_table', sa.Column('new_field', sa.String(), nullable=True))

def downgrade():
    op.drop_column('your_event_table', 'new_field')
```

### Handling Breaking Changes

For changes that aren't backward compatible:

1. Plan for downtime or degraded service during the upgrade.
2. Communicate the changes and potential impact to users in advance.
3. Have a detailed rollback plan ready.

## Conclusion

Deploying and scaling Aptos Indexer processors requires careful planning and execution. By following best practices for deployment, implementing effective scaling strategies, maintaining robust monitoring and logging, and carefully managing upgrades and schema changes, you can ensure your indexer remains reliable and performant as it grows with the Aptos ecosystem. Regular review and optimization of your deployment and scaling strategies will help maintain the efficiency and effectiveness of your indexer over time.