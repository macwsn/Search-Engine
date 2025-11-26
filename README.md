# Wikipedia Search Engine: Vector Space Model vs. LSI (SVD)

This project implements a high-performance search engine based on the Simple English Wikipedia dataset. The core indexing and search logic are engineered in **Rust** for efficiency, utilizing **SQLite** for data storage. The system is wrapped with a **Python** layer for data analysis and prototyping, and a **React** frontend for user interaction.

## 1. Dataset and Storage Strategy

### Technology Choice: SQLite
SQLite was selected for data storage due to:
1.  **Compatibility:** Seamless integration with Rust (backend) and React (frontend).
2.  **Implementation:** Simple setup with no separate server process required.
3.  **Performance:** Fast, low-cost access to data for offline/local deployments.
   *Note: For a high-traffic server environment with massive concurrent queries, MongoDB might be preferred, but SQL fits the offline/embedded nature of this project best.*

### Data Source & Processing
*   **Source:** [Wikimedia Dumps (Simple English)](https://dumps.wikimedia.org/enwiki/latest/)
*   **Parser:** [WikiExtractor](https://github.com/attardi/wikiextractor) (modified version).
*   **Pipeline:** XML Dump $\to$ TXT Files $\to$ SQLite Database.

**Python Script for Database Population:**
```python
import sqlite3
import re
import glob

def parse_file(file_path):
    docs = []
    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()
        pattern = r'<doc id="(\d+)" url="(https?://[^"]+)" title="([^"]+)">([^<]+)<\/doc>'
        matches = re.findall(pattern, content)
        for match in matches:
            docs.append((match[0], match[1], match[2], match[3]))
    return docs

def create_db_and_insert_data(docs):
    conn = sqlite3.connect('articles.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS articles (
            id INTEGER PRIMARY KEY,
            title TEXT,
            url TEXT,
            text TEXT
        )
    ''')
    cursor.executemany('INSERT INTO articles (id, title, url, text) VALUES (?, ?, ?, ?)', docs)
    conn.commit()
    conn.close()
```

### Alternative: Wikipedia API Fetcher
An asynchronous Python script was also developed to fetch random articles via the API, though this method proved slower than parsing the dump.

```python
DATABASE_NAME = "wikipedia_fast.db"
WIKIPEDIA_API_URL = "https://en.wikipedia.org/w/api.php"
TARGET_ARTICLE_COUNT = 300000
```

### Data Cleaning
Initial ingestion resulted in ~370,000 documents. However, anomalies were detected during SVD calculation due to extremely short descriptions.
*   **Action:** Removed articles shorter than 150 characters.
*   **Final Document Count:** 223,412.

```sql
DELETE FROM articles WHERE length(text) < 150;
```

---

## 2. Core Implementation (Rust)

The search engine logic is built in Rust for memory safety and speed.

### A. Text Normalization
1.  **Stop Words:** Removal of common non-informative words (e.g., "the", "is", "at").
2.  **Porter Stemming Algorithm:** Implemented to reduce words to their root forms (removing morphological endings).

### B. Search Algorithm: Vector Space Model (TF-IDF)
The system builds a **Term-Document Matrix** using a sparse data structure.
*   **TF (Term Frequency):** How often a word appears in a specific document.
*   **IDF (Inverse Document Frequency):** Weighting mechanism to reduce the importance of common words across the entire corpus.
*   **Similarity Metric:** Cosine Similarity.

---

## 3. Analysis of Approach 1 (VSM / TF-IDF)

We analyzed the search performance based on **Search Time (T)**, **Match Score (S)**, and **Phrase Length (L)**.

### Performance Metrics
*   **Popular Phrases:** Score is inversely proportional to phrase length.
*   **Rare Phrases:** Generally lower scores; search time is highly dependent on phrase length.
*   **Average Query Time:** 0.20s - 0.25s.

### Impact of IDF (Inverse Document Frequency)
We compared search results with and without IDF weighting.
*   **Observation:** For short queries, the difference is negligible. However, as the query length increases, the results diverge significantly, proving IDF's necessity for complex queries.

## 4. Approach 2: Latent Semantic Indexing (SVD)

This approach utilized **Singular Value Decomposition (SVD)** to identify semantic relationships between terms.

### Implementation Details
*   **Algorithm:** Golub-Kahan-Lanczos algorithm for sparse matrices.
*   **Optimization:** Re-orthogonalization was added to improve numerical stability (Bidiagonalization).
*   **Language:** Rust (Custom implementation, as no mature sparse SVD library exists in the crate ecosystem).

### Results & Challenges
*   **Computation Time:** Calculating SVD for Rank $k=350$ with 1000 iterations took approximately **3 days**.
*   **Accuracy Issues:** The results for $k=350$ showed significant noise (see image below), likely due to approximation errors in the custom Lanczos implementation or numerical precision limits.
*   **Optimization:** To keep search times reasonable, we limited the rank to $k \ge 100$.

![SVD Results Analysis](data2.png)
*The visual noise in the matrix above indicates issues with the approximation algorithm for this specific dataset.*

---

## 5. Frontend & API

*   **API:** Implemented using **actix-web** (Rust).
*   **Frontend:** Built with **React** and **Vite**.
---

## 7. Conclusion

The **Vector Space Model (TF-IDF)** provided the best balance of performance and accuracy for this specific dataset, with query times under 250ms. The **SVD** approach, while theoretically powerful for capturing semantic meaning, proved difficult to implement efficiently in Rust without external linear algebra bindings (like LAPACK), leading to high computational costs and stability issues.