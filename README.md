# LinkGenClassifier: A Comprehensive URL Analysis Tool

This Python script analyzes URLs by fetching and parsing web articles, extracting domain information, and performing various analyses such as sentiment analysis, publication date extraction, redirection count, and outbound link collection. The results are saved to a CSV file.

## Requirements

Ensure you have the following libraries installed:

```sh
pip install requests beautifulsoup4 newspaper3k textblob tldextract python-whois pandas
```

## Code Explanation

### Import Libraries

```python
import requests
from bs4 import BeautifulSoup
from newspaper import Article, ArticleException
from textblob import TextBlob
import tldextract
import whois
import pandas as pd
import time
```

- `requests`: For making HTTP requests to fetch web pages.
- `BeautifulSoup`: For parsing HTML content.
- `newspaper`: For downloading and parsing web articles.
- `TextBlob`: For sentiment analysis.
- `tldextract`: For extracting domain information from URLs.
- `whois`: For fetching domain WHOIS information.
- `pandas`: For handling data and saving it to a CSV file.
- `time`: For handling delays if necessary (not explicitly used in the script but can be useful for rate limiting).

### Class: `LinkGenClassifier`

```python
class LinkGenClassifier:
    def __init__(self, url):
        self.url = url
        self.article = None
        self.domain_info = None
        self.soup = None
```

- **Parameters**:
  - `url`: The URL of the web page to analyze.

- **Attributes**:
  - `self.article`: To store the fetched article.
  - `self.domain_info`: To store domain information.
  - `self.soup`: To store the parsed HTML content.

### Method: `fetch_article`

```python
def fetch_article(self):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }
    article = Article(self.url, headers=headers)
    try:
        article.download()
        article.parse()
        self.article = article
        return article
    except ArticleException as e:
        print(f"Error downloading article {self.url}: {e}")
        return None
```

- **Functionality**:
  - Sets custom headers to mimic a browser request.
  - Downloads and parses the article using the `newspaper` library.
  - Handles exceptions and prints an error message if the article cannot be downloaded.

### Method: `analyze_title_content_relevance`

```python
def analyze_title_content_relevance(self):
    if not self.article:
        return None, None
    title = self.article.title
    content = self.article.text
    return TextBlob(content).sentiment, TextBlob(title).sentiment
```

- **Functionality**:
  - Analyzes the sentiment of the article's title and content using `TextBlob`.
  - Returns the sentiment of the content and title.

### Method: `analyze_domain`

```python
def analyze_domain(self):
    try:
        domain = tldextract.extract(self.url).domain
        self.domain_info = whois.whois(self.url)
        return domain, self.domain_info
    except Exception as e:
        print(f"Error analyzing domain {self.url}: {e}")
        return None, None
```

- **Functionality**:
  - Extracts the domain from the URL.
  - Fetches WHOIS information for the domain.
  - Handles exceptions and prints an error message if the domain information cannot be fetched.

### Method: `analyze_sentiment_and_topics`

```python
def analyze_sentiment_and_topics(self):
    if not self.article:
        return None, None
    content = self.article.text
    sentiment = TextBlob(content).sentiment
    return sentiment, content
```

- **Functionality**:
  - Analyzes the sentiment of the article's content.
  - Returns the sentiment and the content of the article.

### Method: `analyze_publication_date`

```python
def analyze_publication_date(self):
    if not self.article:
        return None
    return self.article.publish_date
```

- **Functionality**:
  - Extracts the publication date of the article if available.

### Method: `analyze_redirections`

```python
def analyze_redirections(self):
    try:
        response = requests.get(self.url)
        redirections = len(response.history)
        return redirections
    except Exception as e:
        print(f"Error analyzing redirections {self.url}: {e}")
        return None
```

- **Functionality**:
  - Counts the number of redirections that occurred when accessing the URL.
  - Handles exceptions and prints an error message if redirections cannot be analyzed.

### Method: `analyze_outbound_links`

```python
def analyze_outbound_links(self):
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }
    try:
        page = requests.get(self.url, headers=headers)
        self.soup = BeautifulSoup(page.content, 'html.parser')
        outbound_links = [a['href'] for a in self.soup.find_all('a', href=True)]
        return outbound_links
    except Exception as e:
        print(f"Error analyzing outbound links {self.url}: {e}")
        return None
```

- **Functionality**:
  - Fetches the web page and parses its HTML content.
  - Extracts all outbound links (href attributes) from the page.
  - Handles exceptions and prints an error message if outbound links cannot be analyzed.

### Method: `evaluate`

```python
def evaluate(self):
    if not self.fetch_article():
        return {
            "Title-Content Relevance": None,
            "Domain": None,
            "Sentiment Analysis": None,
            "Publication Date": None,
            "Redirections": None,
            "Outbound Links": None,
        }
    relevance = self.analyze_title_content_relevance()
    domain, domain_info = self.analyze_domain()
    sentiment, topics = self.analyze_sentiment_and_topics()
    pub_date = self.analyze_publication_date()
    redirections = self.analyze_redirections()
    outbound_links = self.analyze_outbound_links()

    report = {
        "Title-Content Relevance": relevance,
        "Domain": domain,
        "Sentiment Analysis": sentiment,
        "Publication Date": pub_date,
        "Redirections": redirections,
        "Outbound Links": outbound_links,
    }
    return report
```

- **Functionality**:
  - Calls various analysis methods to collect data about the URL.
  - Returns a report containing all the analyzed data.

### Function: `process_csv`

```python
def process_csv(input_csv, output_csv):
    df = pd.read_csv(input_csv)
    results = []

    for index, row in df.iterrows():
        url = row['URL']
        classifier = LinkGenClassifier(url)
        report = classifier.evaluate()

        results.append({
            "URL": url,
            "Title-Content Relevance": report["Title-Content Relevance"],
            "Domain": report["Domain"],
            "Sentiment Analysis": report["Sentiment Analysis"],
            "Publication Date": report["Publication Date"],
            "Redirections": report["Redirections"],
            "Outbound Links": len(report["Outbound Links"]) if report["Outbound Links"] is not None else None
        })

    result_df = pd.DataFrame(results)
    result_df.to_csv(output_csv, index=False)
```

- **Parameters**:
  - `input_csv`: The path to the input CSV file containing URLs to be analyzed.
  - `output_csv`: The path to the output CSV file where the analysis results will be saved.

- **Functionality**:
  - Reads URLs from the input CSV file.
  - Instantiates a `LinkGenClassifier` for each URL and evaluates it.
  - Collects the results and saves them to the output CSV file.

### Example Usage

```python
input_csv = 'collected_urls.csv'
output_csv = 'output_analysis.csv'
process_csv(input_csv, output_csv)
```

- Collects URLs from `collected_urls.csv`.
- Analyzes each URL and saves the results to `output_analysis.csv`.

This script provides a comprehensive analysis of URLs, making it useful for web content analysis, sentiment analysis, and domain information extraction.
