# Data Engineering Principles: A Comprehensive Reference

A 500+ page technical book covering modern data engineering from first principles.

## About This Book

This book is designed for experienced software engineers transitioning into data engineering. Rather than providing prescriptive tutorials, it builds mental models and decision frameworks that enable readers to independently analyze, design, and evaluate data architectures.

### Key Themes

- **First principles**: Understand WHY technologies exist, not just HOW to use them
- **Historical evolution**: Learn how data warehouses became data lakes became lakehouses
- **Decision frameworks**: Evaluate options against principles, not just follow trends
- **Systems thinking**: Understand components, boundaries, and trade-offs

### Target Reader

- Senior/Staff software engineers with 10+ years experience
- Strong programming skills (Python, Java, or similar)
- Database expertise (PostgreSQL, MySQL)
- Familiarity with Kafka or similar messaging systems
- Building or evolving data platforms

### Topics Covered

- Table formats: Apache Iceberg, Delta Lake
- Distributed compute: Apache Spark
- Orchestration: Airflow 3, Prefect, Dagster
- Storage: Columnar formats, object storage
- Batch vs streaming processing
- Data modeling for analytics
- OLAP databases (ClickHouse, DuckDB)
- Transformation (dbt)
- Data quality and observability

## Start Reading

- [Preface](src/front-matter/preface.md)
- [Introduction](src/front-matter/introduction.md)

### Chapters

1. [The Data Engineering Landscape](src/chapters/01-landscape/chapter.md)
2. [First Principles of Data Systems](src/chapters/02-first-principles/chapter.md)
3. [The Evolution of Data Architectures](src/chapters/03-evolution/chapter.md)
4. [Storage Fundamentals](src/chapters/04-storage-fundamentals/chapter.md)
5. [Table Formats — The Lakehouse Foundation](src/chapters/05-table-formats/chapter.md)
6. [Analytical Databases (OLAP)](src/chapters/06-olap/chapter.md)
7. [Distributed Compute Fundamentals](src/chapters/07-distributed-compute/chapter.md)
8. [Apache Spark Deep Dive](src/chapters/08-spark/chapter.md)
9. [Batch vs Streaming: A Unified Perspective](src/chapters/09-batch-streaming/chapter.md)
10. [Pipeline Orchestration Principles](src/chapters/10-orchestration-principles/chapter.md)
11. [Orchestration Tools: Airflow, Prefect, and Dagster](src/chapters/11-orchestration-tools/chapter.md)
12. [The Transformation Layer: dbt](src/chapters/12-dbt/chapter.md)
13. [Data Modeling for Analytics](src/chapters/13-data-modeling/chapter.md)
14. [Data Quality and Observability](src/chapters/14-data-quality/chapter.md)
15. [Architecture Patterns](src/chapters/15-architecture-patterns/chapter.md)
16. [Case Studies: Architectures in Practice](src/chapters/16-case-studies/chapter.md)
17. [Evaluating New Technologies](src/chapters/17-evaluating-technologies/chapter.md)

### Appendices

- [A — Technology Comparison Tables](src/appendices/a-comparison-tables.md)
- [B — Decision Flowcharts](src/appendices/b-decision-flowcharts.md)
- [C — Glossary](src/appendices/c-glossary.md)
- [D — Further Reading](src/appendices/d-further-reading.md)

## License

All rights reserved.
