# Preface

Data engineering moves fast. New tools launch monthly. Yesterday's best practice becomes today's anti-pattern. A book attempting to document the current state of the field would be obsolete before publication.

This book takes a different approach. Instead of cataloging tools and technologies, it builds mental models—frameworks for thinking about data systems that remain useful as specific technologies evolve. The goal isn't to make you an expert in Apache Spark or Apache Iceberg (though you'll learn plenty about both). The goal is to help you reason about any data system, including ones that don't exist yet.

## Who This Book Is For

This book is written for software engineers moving into data engineering. You already know how to build software. You understand databases, APIs, and distributed systems at some level. What you may lack is the mental model for thinking about data at scale—the principles that guide architectural decisions when you're dealing with terabytes instead of gigabytes, batch jobs instead of request-response, and analytics instead of transactions.

If you're a:

**Software engineer transitioning to data engineering:** You'll find the "why" behind data engineering practices, not just the "how." The book connects concepts you already know (databases, distributed systems, software engineering practices) to their data engineering counterparts.

**Data engineer seeking deeper understanding:** If you've learned tools through tutorials but want to understand the principles underneath, this book provides that foundation. You'll learn why your tools work the way they do and when to choose different approaches.

**Technical leader evaluating data technologies:** The decision frameworks throughout the book help you assess technologies based on your specific context rather than following trends.

**Graduate or bootcamp student:** The comprehensive coverage provides a map of the data engineering landscape and the mental models to navigate it.

## How This Book Is Organized

The book follows a logical progression from foundations to synthesis:

**Part I: Foundations (Chapters 1-3)** establishes the landscape, first principles, and historical context. These chapters build the vocabulary and mental models used throughout the book.

**Part II: Storage Layer (Chapters 4-6)** covers how data is stored—from file formats to table formats to analytical databases. Understanding storage is prerequisite to understanding everything built on top.

**Part III: Compute Layer (Chapters 7-9)** examines how data is processed—distributed computing fundamentals, Apache Spark, and the batch vs. streaming spectrum.

**Part IV: Orchestration (Chapters 10-12)** addresses how pipelines are coordinated—orchestration principles, tools (Airflow, Prefect, Dagster), and the transformation layer (dbt).

**Part V: Quality and Modeling (Chapters 13-14)** covers how data is organized and validated—data modeling paradigms and quality/observability practices.

**Part VI: Synthesis (Chapters 15-17)** brings everything together—architecture patterns, case studies, and frameworks for evaluating new technologies.

Each chapter is designed to be self-contained after Part I. If you need to understand table formats, jump to Chapter 5. If you're evaluating orchestrators, start with Chapter 10. The cross-references help you fill in prerequisites as needed.

## What This Book Is Not

**Not a tutorial collection.** You won't find step-by-step instructions for installing Spark or configuring Airflow. Official documentation does that better and stays current. This book explains when and why you'd use these tools.

**Not a tool comparison.** While the book discusses many technologies, it doesn't declare winners. Instead, it provides frameworks for making decisions based on your context.

**Not a certification prep guide.** The book prioritizes understanding over memorization. You'll learn principles that help you answer any question, not just the ones on tests.

**Not cloud-specific.** Examples use AWS terminology where needed, but concepts apply across cloud providers and on-premises deployments.

## Conventions Used

**Code examples** appear in monospaced font. They're designed to illustrate concepts rather than be copy-pasted into production. Real implementations would need error handling, configuration, and testing that would obscure the educational point.

**SQL examples** use standard SQL where possible. Dialect-specific features are noted when used.

**Diagrams** use ASCII art or Mermaid notation for reproducibility. Some diagrams are simplified to highlight key concepts.

**Chapter cross-references** appear as "see Chapter X" and indicate where related concepts are covered in depth.

## Staying Current

Technology changes; principles endure. This book emphasizes principles that remain stable:

- Distributed systems fundamentals don't change when new databases launch
- Data modeling patterns have persisted for decades
- The trade-offs between consistency, availability, and partition tolerance remain constant
- Software engineering practices (testing, version control, documentation) apply regardless of tools

For current technology details, the Further Reading sections point to official documentation and authoritative sources that are maintained over time.

## Acknowledgments

This book synthesizes knowledge from countless practitioners who've shared their experiences through blog posts, conference talks, open source contributions, and conversations. The data engineering community's openness in discussing successes and failures makes learning possible.

Special recognition goes to the teams at companies like Netflix, Uber, Airbnb, LinkedIn, and Spotify who publish detailed technical blogs. These real-world accounts of data systems at scale inform the case studies and examples throughout.

The open source communities behind Apache Spark, Apache Kafka, Apache Iceberg, dbt, Airflow, and the many other tools discussed have built the infrastructure that makes modern data engineering possible.

## Feedback

Data engineering continues to evolve. If you find errors, have suggestions, or want to share how you've applied these concepts, your feedback helps improve future editions.

Let's build better data systems.
