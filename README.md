# Search Engine Project Report

## 1. Dataset

To store the data, SQLite was used due to its compatibility with Rust and React (the technologies used in this project), simple implementation, and especially fast and low-cost data access. If this were a server with a larger number of queries, I would choose MongoDB, but here for offline, simple, and not handling a large number of queries, SQL is sufficient.

Data was downloaded from: https://dumps.wikimedia.org/enwiki/latest/

The Simple English Wikipedia version was used. Data from the XML file was parsed into TXT files using a slightly modified version of WikiExtractor ([link](https://github.com/attardi/wikiextractor)) for this task.

The TXT files were then imported into the SQLite database.

## Data Extraction Example
```python
import sqlite3
import re
import glob

def parse_file(file_path):
    docs = []
    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()
        pattern = r'<doc id="(\d+)" url="(https?://[^"]+)" title="([^"]+)">([^<]+)</doc>'
        matches = re.findall(pattern, content)
        for match in matches:
            doc_id = match[0]
            url = match[1]
            title = match[2]
            text = match[3]
            docs.append((doc_id, title, url, text))
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

    cursor.executemany('''
        INSERT INTO articles (id, title, url, text)
        VALUES (?, ?, ?, ?)
    ''', docs)

    conn.commit()
    conn.close()

def main():
    folders = ['AA/*', 'AC/*', 'AB/*']
    for file in folders:
        file_paths = glob.glob(f)
        all_docs = []
        for file_path in file_paths:
            print(f'Processing file: {file_path}')
            docs = parse_file(file_path)
            all_docs.extend(docs)

        if all_docs:
            create_db_and_insert_data(all_docs)
            print("Data saved to database.")
        else:
            print("No data to save.")
```
