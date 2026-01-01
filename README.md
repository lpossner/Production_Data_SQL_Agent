# Production Data SQL Agent

An intelligent SQL agent powered by LangChain and LangGraph that enables natural language queries over industrial production data. Ask questions in plain language and get accurate answers about machines, production orders, errors, and maintenance plans.

## Overview

This project demonstrates an AI-powered database query system for industrial process management. It combines:

- **Natural Language Processing**: Ask questions in English or German
- **Intelligent SQL Generation**: Automatically creates and executes correct SQL queries
- **Context-Aware Retrieval**: Uses vector search to find correct filter values
- **Conversational Memory**: Maintains context across multiple questions

## Features

- **Multi-language Support**: Query in English or German, get responses in the same language
- **Smart Value Matching**: Fuzzy matching for column values (e.g., "Machine-027" → "Machine-002")
- **Automatic Schema Discovery**: Agent explores database structure before querying
- **Error Recovery**: Automatically retries failed queries with corrections
- **Conversational Context**: Follow-up questions reference previous queries

## Database Schema

The system manages four key tables:

### Machines
- Machine information (name, type, location, status, year)
- Types: Schweißroboter, Fräsmaschine, Montageroboter
- Locations: Werkhalle A, B, C

### Production Orders
- Product types: Getriebe, Motorblock, Bremssystem
- Quality status: Okay, Nacharbeit, Ausschuss
- Time tracking with start and end dates

### Error Logs
- Error types: Überhitzung, Materialstau, Sensorfehler
- Severity levels: Niedrig, Mittel, Hoch
- Linked to specific machines

### Maintenance Plans
- Scheduled and completed maintenance tracking
- Technician assignments
- Machine-specific maintenance history

## Architecture

```
    User Question (Natural Language)
                ↓
         ┌──────────────┐
         │  LangGraph   │
         │  Agent       │
         └──────────────┘
                ↓
         ┌──────────────┐
         │  Tool        │
         │  Selection   │
         └──────────────┘
                ↓
         ┌──────────────┐
         │  SQL Tools   │
         │──────────────│
         │ - List       │
         │   Tables     │
         │ - Get Schema │
         │ - Query      │
         │   Checker    │
         │ - Execute    │
         │   Query      │
         └──────────────┘
                ↓
         ┌──────────────┐
         │  Retriever   │
         │──────────────│
         │ - Search     │
         │   Values     │
         └──────────────┘
                ↓
    Natural Language Response
```

## Example Conversations

### Machine Problem Analysis
```
Q: Welche Maschinen hatten in den letzten drei Monaten die meisten Probleme?
A: In den letzten drei Monaten hatten die folgenden Maschinen die meisten Probleme:
   1. Machine-009 (17 Probleme)
   2. Machine-008 (17 Probleme)
   3. Machine-002 (14 Probleme)

Q: Kannst Du mir die häufigsten Probleme der Maschinen mit den meisten Fehlern nennen?
A: Die häufigsten Probleme sind:
   - Machine-009: Materialstau, Überhitzung, Sensorfehler
   - Machine-008: Überhitzung, Sensorfehler, Materialstau
   - Machine-002: Sensorfehler, Materialstau, Überhitzung
```

### Production Quality Analysis
```
Q: Welche Produktionsaufträge hatten in den letzten sechs Monaten die höchste Ausschussquote?
A: Die Produktionsaufträge mit der höchsten Ausschussquote waren:
   1. Auftrag 641: Produkt "Bremssystem" (30.06.2025 – 09.07.2025)
   2. Auftrag 517: Produkt "Bremssystem" (29.06.2025 – 08.07.2025)
   3. Auftrag 263: Produkt "Motorblock" (26.06.2025 – 06.07.2025)

Q: Welche Maschinen wurden für den Auftrag mit der höchsten Ausschussquote verwendet?
A: Machine-007 (Montageroboter, Werkhalle C) mit bekannten Fehlern:
   - 13.06.2025: Überhitzung (Mittel)
   - 14.05.2025: Sensorfehler (Niedrig)
   - 08.05.2025: Sensorfehler (Hoch)
```

### Maintenance Tracking
```
Q: Welche Maschinen haben geplante Wartungen nicht durchgeführt?
A: Folgende Maschinen haben Wartungen versäumt:
   - Machine-001 (Fräsmaschine, Werkhalle B), geplant am 14.05.2025
   - Machine-006 (Fräsmaschine, Werkhalle C), geplant am 23.05.2025
   - Machine-002 (Montageroboter, Werkhalle B), geplant am 06.06.2025

Q: Gab es Fehler nach den verpassten Wartungen?
A: Ja, mehrere Fehler traten auf:
   - Machine-001: Sensorfehler (21.05.2025), Überhitzung (03.06.2025)
   - Machine-006: Überhitzung (28.05.2025)
```

## Installation

### 1. Clone the Repository
```bash
git clone https://github.com/lpossner/Production-Data-SQL-Agent.git
cd Production-Data-SQL-Agent
```

### 2. Install Dependencies

```bash
pip install uv
uv sync
```

### 3. Configure OpenAI API
Set up your OpenAI API credentials in the notebook or as environment variables.

## Usage

The workflow consists of three main steps:

### 1. Initialize the Database
Run the database setup cells in the notebook to generate sample industrial production data:
```bash
jupyter notebook gqw_2015.ipynb
```

### 2. Configure LLM and Embeddings
Set up your OpenAI language model and embeddings in the notebook configuration cells.

### 3. Query the Database
Use natural language queries in English or German to interact with the production data through the SQL agent.

## Key Components

### Custom Retriever Tool

The `RetrieverTool` enables fuzzy matching for filter values:
- Input: Approximate spelling (e.g., "Machine-027")
- Output: Correct value, table, and column (e.g., "Machine-002")

### System Prompt

The agent follows specific guidelines:
- Always explore schema before querying
- Limit results to TOP_K (default: 3)
- Use retriever for filter values
- Answer in the question's language
- No destructive operations (INSERT, UPDATE, DELETE)

### Conversational Memory

The agent uses `MemorySaver` to maintain context across questions, enabling natural follow-up queries without repeating context.

## Technical Details

- **Framework**: LangChain + LangGraph
- **Database**: SQLite
- **Vector Store**: InMemoryVectorStore
- **Agent Type**: ReAct (Reasoning + Acting)
- **Model**: OpenAI (configurable)

## Generated SQL Queries

The agent creates optimized queries like:

```sql
-- Machine problem analysis
SELECT m.name, COUNT(e.id) AS problem_count
FROM error_logs e
JOIN machines m ON e.machine_id = m.id
WHERE e.error_date >= date('now', '-3 months')
GROUP BY m.name
ORDER BY problem_count DESC
LIMIT 3;

-- Production quality tracking
SELECT id, product, start_date, end_date, quality_status
FROM production_orders
WHERE end_date >= date('now', '-6 months')
ORDER BY CASE WHEN quality_status = 'Ausschuss' THEN 1 ELSE 0 END DESC
LIMIT 3;
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
