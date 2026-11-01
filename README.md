# JEV-Powered Review Analyzer

A Python pipeline that performs aspect-based review analysis on customer feedback using TypeSafe AI's System One decision model (`jev-latest`).

The analyzer evaluates individual smartphone reviews across seven specific product categories, detects whether an aspect was discussed, computes a calibrated 1-to-5 star rating for each mentioned aspect, and prints an aggregated aspect-by-aspect breakdown similar to e-commerce platforms like Flipkart.

---

## Architecture and Methodology

Instead of prompting a generative large language model to produce unstructured free text or JSON, this project uses the TypeSafe SDK (`typesafe-sdk`) and its `system_one` decision engine.

### 1. Dual-Question Aspect Evaluation

For each review, the system submits 14 questions in a single request to the `jev-latest` model (2 questions per topic across 7 topics):

| Topic | Mention Detection (`Noul`) | Satisfaction Scoring (`Score`) |
| :--- | :--- | :--- |
| **Camera** | `camera_mentioned` | `camera_rating` |
| **Battery** | `battery_mentioned` | `battery_rating` |
| **Display** | `display_mentioned` | `display_rating` |
| **Design** | `design_mentioned` | `design_rating` |
| **Performance** | `performance_mentioned` | `performance_rating` |
| **Build Quality** | `build_quality_mentioned` | `build_quality_rating` |
| **Value for Money** | `value_for_money_mentioned` | `value_for_money_rating` |

### 2. Decision Primitives and Rating Conversion

- **Mention Filtering (`Noul`):** Returns a probability between `0.0` and `1.0` indicating whether the reviewer discussed that aspect. The pipeline applies a strict cutoff (`MENTION_THRESHOLD = 0.5` in `analyzer.py`). If the probability is below `0.5`, the topic is excluded from the review's ratings.
- **Sentiment Scoring (`Score`):** If mentioned, the satisfaction level is evaluated against five discrete ordinal criteria:
  1. `Very dissatisfied` (Score: 0.0)
  2. `Dissatisfied` (Score: 1.0)
  3. `Neither satisfied nor dissatisfied, or balanced mixed feedback` (Score: 2.0)
  4. `Satisfied` (Score: 3.0)
  5. `Very satisfied` (Score: 4.0)
- **Star Calculation:** The score (0.0 to 4.0) is converted to a standard 1-to-5 star scale:
  $$\text{stars} = \text{score} + 1$$
  The result is rounded to one decimal place.

### 3. Aggregation Logic

`aggregation.py` calculates:
- The arithmetic mean of all star ratings for each topic across reviews where that topic met the mention threshold.
- The total number of reviews contributing to each topic's score.
- Topics that were never discussed across any reviews are assigned `None` and reported as `no ratings yet`.

`main.py` also calculates the overall product rating by averaging the `overall_rating` column from the raw dataset.

---

## Repository Structure

```text
JEV-Powered-Review-Analyzer/
├── aggregation.py             # Averages topic ratings and tallies review counts
├── analyzer.py                # TypeSafeClient initialization and per-review analysis
├── main.py                    # Application entry point: CSV ingestion and reporting
├── questions.py               # Topic mapping, criteria, and TypeSafe question schemas
└── synthetic_phone_reviews.csv # Sample dataset containing 50 phone reviews
```

### File Responsibilities

- **`main.py`**: Reads `synthetic_phone_reviews.csv` using Python's standard `csv.DictReader`, passes each review text to `analyzer.py`, logs the intermediate per-review ratings with `review_id`, and outputs the formatted summary table.
- **`analyzer.py`**: Loads environment variables with `python-dotenv`, instantiates `TypeSafeClient`, invokes `client.system_one(model="jev-latest", state=review_text, questions=QUESTIONS)`, filters questions by `MENTION_THRESHOLD`, and maps scores to stars.
- **`questions.py`**: Defines the `TOPICS` dictionary mapping display labels to internal keys, the five-level `SATISFACTION_LEVELS` array, and the 14 `Noul` and `Score` question definitions passed to the model.
- **`aggregation.py`**: Implements `aggregate(all_ratings)` which takes the list of topic rating dictionaries and computes the average score and count per topic.
- **`synthetic_phone_reviews.csv`**: Contains 50 synthetic test reviews for the phone model "Aster M1 5G" with varied customer sentiment, mixed-language phrases (English and Hinglish), and edge cases (e.g., reviews about delivery rather than the product).

---

## Dataset Schema

The input dataset (`synthetic_phone_reviews.csv`) adheres to the following structure:

| Column | Type | Description |
| :--- | :--- | :--- |
| `review_id` | string | Unique review identifier (e.g., `R001`, `R002`) |
| `phone_model` | string | Name of the hardware model being reviewed (`Aster M1 5G`) |
| `overall_rating` | integer | Customer's overall rating on a 1-to-5 scale |
| `review_text` | string | Full text of the review |
| `is_synthetic` | boolean | Flag marking whether the row was synthetically generated |

---

## Setup and Installation

### Prerequisites

- Python 3.10 or later
- A TypeSafe API key

### 1. Install Dependencies

Install the required Python packages:

```bash
pip install typesafe-sdk python-dotenv
```

### 2. Configure API Credentials

The `analyzer.py` module looks for the `TYPESAFE_API_KEY` environment variable via `python-dotenv`.

Create a `.env` file in the project root:

```env
TYPESAFE_API_KEY=your_actual_typesafe_api_key_here
```

Alternatively, export the environment variable in your terminal session:

**Linux / macOS:**
```bash
export TYPESAFE_API_KEY="your_actual_typesafe_api_key_here"
```

**Windows PowerShell:**
```powershell
$env:TYPESAFE_API_KEY="your_actual_typesafe_api_key_here"
```

**Windows Command Prompt:**
```cmd
set TYPESAFE_API_KEY=your_actual_typesafe_api_key_here
```

---

## Running the Application

Execute `main.py` from the project directory:

```bash
python main.py
```

### Execution Flow

1. Reads all rows from `synthetic_phone_reviews.csv`.
2. Iterates through each review and queries the `jev-latest` model.
3. Prints the review ID and identified aspect ratings as processing occurs:
   ```text
   R001 {'Battery': 4.0, 'Camera': 4.8, 'Performance': 4.5, 'Value for Money': 4.9}
   R002 {'Battery': 1.4}
   R003 {'Camera': 2.8}
   ...
   ```
4. Prints the final summary block showing overall score and aspect-level averages:

