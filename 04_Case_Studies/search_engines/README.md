# Search Engines (Google/Elasticsearch)

## Problem Statement

Design a distributed search engine like Google or Elasticsearch that can index billions of web pages and serve millions of queries per second.

## Requirements

### Functional Requirements
- Web page crawling and indexing
- Query processing and ranking
- Relevance scoring and personalization
- Distributed indexing and storage
- Real-time indexing updates
- Search analytics and reporting
- API for search integration

### Non-Functional Requirements
- **Scale**: Index 100 billion pages, 1 trillion queries per year
- **Latency**: < 100ms for simple queries, < 1 second for complex
- **Availability**: 99.9% uptime
- **Freshness**: Index updates within hours
- **Accuracy**: High relevance and precision

## Scale Estimation

### Traffic Estimates
```
Daily Queries: 5 billion
Peak Queries per Second: 100,000
Indexed Documents: 100 billion
Daily New Documents: 10 million
Index Size: 100 PB
```

### Storage Estimates
```
Document Index: 100B × 10KB = 1PB
Inverted Index: 100B × 5KB = 500TB
Forward Index: 100B × 2KB = 200TB
Metadata: 100B × 1KB = 100TB
```

## High-Level Design

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
└─────────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────────┐
│                 Query Processor                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Parser    │  │   Ranker    │  │   Merger    │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Index Servers                              │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Inverted  │  │   Forward   │  │   Document  │      │
│  │   Index     │  │   Index     │  │   Store     │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Crawling & Indexing                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Crawler   │  │   Parser    │  │   Indexer   │      │
│  │   Service   │  │   Service   │  │   Service   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
    ↓                ↓                ↓
┌─────────────────────────────────────────────────────────┐
│              Data Stores                                │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Bigtable  │  │   GFS       │  │   Spanner   │      │
│  │   (Index)   │  │   (Storage) │  │   (Metadata)│      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
└─────────────────────────────────────────────────────────┘
```

## API Design

### Search APIs
```http
GET /api/v1/search?q=system+design&num=10&start=0
Response:
{
  "query": "system design",
  "total_results": 1250000,
  "results": [
    {
      "title": "System Design Interview Guide",
      "url": "https://example.com/system-design",
      "snippet": "Comprehensive guide to system design interviews...",
      "rank": 1,
      "score": 0.95
    }
  ],
  "search_time": 0.023
}

GET /api/v1/search/advanced
{
  "query": "machine learning",
  "filters": {
    "site": "example.com",
    "date_range": "2023-01-01:2024-01-01",
    "file_type": "pdf"
  },
  "sort": "relevance",
  "num": 20
}
```

### Indexing APIs
```http
POST /api/v1/index
{
  "url": "https://example.com/page",
  "content": "<html><body><h1>Title</h1><p>Content...</p></body></html>",
  "metadata": {
    "title": "Page Title",
    "description": "Page description",
    "keywords": ["keyword1", "keyword2"],
    "last_modified": "2024-01-01T00:00:00Z"
  }
}

PUT /api/v1/index/{document_id}
{
  "url": "https://example.com/page",
  "content": "Updated content...",
  "metadata": {...}
}
```

## Database Design

### Documents Table
```sql
CREATE TABLE documents (
    id BIGINT PRIMARY KEY,
    url VARCHAR(2000) UNIQUE NOT NULL,
    title VARCHAR(1000),
    content TEXT,
    metadata JSON,
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_crawled TIMESTAMP,
    status ENUM('indexed', 'pending', 'failed') DEFAULT 'pending',
    INDEX idx_url (url),
    INDEX idx_status (status),
    INDEX idx_indexed_at (indexed_at),
    FULLTEXT idx_content (content),
    FULLTEXT idx_title (title)
);
```

### Inverted Index Table
```sql
CREATE TABLE inverted_index (
    term VARCHAR(255) NOT NULL,
    document_id BIGINT NOT NULL,
    frequency INTEGER NOT NULL,
    positions JSON,  -- Array of positions in document
    PRIMARY KEY (term, document_id),
    INDEX idx_term (term),
    INDEX idx_document_id (document_id)
);
```

### PageRank Table
```sql
CREATE TABLE pagerank (
    document_id BIGINT PRIMARY KEY,
    score DECIMAL(10,8) NOT NULL,
    inlinks_count INTEGER DEFAULT 0,
    outlinks_count INTEGER DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_score (score)
);
```

## Web Crawling

### Distributed Crawler
```python
import requests
from urllib.parse import urlparse, urljoin
from bs4 import BeautifulSoup
import time
import hashlib
from typing import Set, List, Dict
import concurrent.futures
from dataclasses import dataclass

@dataclass
class CrawlTask:
    url: str
    depth: int
    priority: int

class WebCrawler:
    def __init__(self, max_depth: int = 3, max_workers: int = 10):
        self.max_depth = max_depth
        self.max_workers = max_workers
        self.visited: Set[str] = set()
        self.queue: List[CrawlTask] = []
        self.domain_delays = {}  # Respect robots.txt
        self.user_agent = 'WebCrawler/1.0'

    def crawl(self, start_urls: List[str]):
        """Start crawling from seed URLs"""
        # Initialize queue with start URLs
        for url in start_urls:
            self.queue.append(CrawlTask(url, 0, 1))

        with concurrent.futures.ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            while self.queue:
                # Sort queue by priority (highest first)
                self.queue.sort(key=lambda x: x.priority, reverse=True)

                # Take batch of URLs to crawl
                batch_size = min(self.max_workers, len(self.queue))
                batch = self.queue[:batch_size]
                self.queue = self.queue[batch_size:]

                # Submit crawl tasks
                futures = [executor.submit(self._crawl_url, task) for task in batch]

                # Process results
                for future in concurrent.futures.as_completed(futures):
                    try:
                        new_tasks = future.result()
                        self.queue.extend(new_tasks)
                    except Exception as e:
                        print(f"Crawl error: {e}")

    def _crawl_url(self, task: CrawlTask) -> List[CrawlTask]:
        """Crawl a single URL"""
        url = task.url

        # Check if already visited
        url_hash = hashlib.md5(url.encode()).hexdigest()
        if url_hash in self.visited:
            return []

        self.visited.add(url_hash)

        try:
            # Respect domain delays
            domain = urlparse(url).netloc
            if domain in self.domain_delays:
                time.sleep(self.domain_delays[domain])

            # Fetch page
            headers = {'User-Agent': self.user_agent}
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()

            # Parse content
            soup = BeautifulSoup(response.content, 'html.parser')

            # Extract text content
            title = soup.title.string if soup.title else ""
            content = soup.get_text()

            # Index the page
            self._index_page(url, title, content)

            # Extract links if within depth limit
            new_tasks = []
            if task.depth < self.max_depth:
                links = self._extract_links(soup, url)
                for link in links:
                    if self._should_crawl_link(link):
                        new_tasks.append(CrawlTask(
                            link,
                            task.depth + 1,
                            self._calculate_priority(link)
                        ))

            # Update domain delay based on response time
            self.domain_delays[domain] = min(response.elapsed.total_seconds(), 1.0)

            return new_tasks

        except Exception as e:
            print(f"Error crawling {url}: {e}")
            return []

    def _extract_links(self, soup: BeautifulSoup, base_url: str) -> List[str]:
        """Extract all links from page"""
        links = []
        for a_tag in soup.find_all('a', href=True):
            href = a_tag['href']
            absolute_url = urljoin(base_url, href)
            links.append(absolute_url)
        return links

    def _should_crawl_link(self, url: str) -> bool:
        """Determine if link should be crawled"""
        parsed = urlparse(url)

        # Only HTTP/HTTPS
        if parsed.scheme not in ['http', 'https']:
            return False

        # Skip certain file types
        skip_extensions = ['.pdf', '.jpg', '.png', '.gif', '.css', '.js']
        if any(url.lower().endswith(ext) for ext in skip_extensions):
            return False

        # Skip already visited
        url_hash = hashlib.md5(url.encode()).hexdigest()
        if url_hash in self.visited:
            return False

        return True

    def _calculate_priority(self, url: str) -> int:
        """Calculate crawl priority for URL"""
        priority = 1

        # Higher priority for certain domains or paths
        if 'system-design' in url.lower():
            priority = 10
        elif '.edu' in url or '.gov' in url:
            priority = 5

        return priority

    def _index_page(self, url: str, title: str, content: str):
        """Index the crawled page (placeholder)"""
        # In real implementation, send to indexing service
        print(f"Indexing: {title} - {url}")

# Usage
crawler = WebCrawler(max_depth=2, max_workers=5)
crawler.crawl([
    'https://en.wikipedia.org/wiki/System_design',
    'https://www.example.com'
])
```

## Inverted Index Construction

### MapReduce Indexing
```python
from typing import List, Dict, Tuple, Iterator
import hashlib
from collections import defaultdict

class InvertedIndexBuilder:
    def __init__(self):
        self.stop_words = {'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for', 'of', 'with', 'by'}

    def build_index(self, documents: List[Dict]) -> Dict[str, List[Tuple[int, int, List[int]]]]:
        """Build inverted index using MapReduce pattern"""
        # Map phase
        intermediate_results = []
        for doc in documents:
            intermediate_results.extend(self._map_function(doc))

        # Shuffle and sort
        intermediate_results.sort(key=lambda x: x[0])  # Sort by term

        # Reduce phase
        inverted_index = self._reduce_function(intermediate_results)

        return inverted_index

    def _map_function(self, document: Dict) -> List[Tuple[str, Tuple[int, str]]]:
        """Map function: extract terms from document"""
        doc_id = document['id']
        content = document['content'].lower()

        # Tokenize and stem
        terms = self._tokenize(content)

        # Remove stop words and stem
        filtered_terms = [self._stem(term) for term in terms if term not in self.stop_words]

        # Count term frequencies and positions
        term_positions = defaultdict(list)
        for pos, term in enumerate(filtered_terms):
            term_positions[term].append(pos)

        # Emit intermediate key-value pairs
        results = []
        for term, positions in term_positions.items():
            results.append((term, (doc_id, len(positions), positions)))

        return results

    def _reduce_function(self, intermediate: List[Tuple[str, Tuple[int, int, List[int]]]]) -> Dict[str, List[Tuple[int, int, List[int]]]]:
        """Reduce function: aggregate postings for each term"""
        inverted_index = defaultdict(list)

        i = 0
        while i < len(intermediate):
            term = intermediate[i][0]
            postings = []

            # Collect all postings for this term
            while i < len(intermediate) and intermediate[i][0] == term:
                postings.append(intermediate[i][1])
                i += 1

            # Sort postings by document ID
            postings.sort(key=lambda x: x[0])
            inverted_index[term] = postings

        return dict(inverted_index)

    def _tokenize(self, text: str) -> List[str]:
        """Simple tokenization"""
        # Remove punctuation and split
        import re
        return re.findall(r'\b\w+\b', text)

    def _stem(self, word: str) -> str:
        """Simple stemming (Porter stemmer simplified)"""
        # Very basic stemming - in production use proper stemmer
        if word.endswith('ing'):
            word = word[:-3]
        elif word.endswith('ly'):
            word = word[:-2]
        elif word.endswith('s'):
            word = word[:-1]

        return word

    def search(self, inverted_index: Dict, query: str) -> List[int]:
        """Search using inverted index"""
        query_terms = [self._stem(term.lower()) for term in self._tokenize(query)
                      if term.lower() not in self.stop_words]

        if not query_terms:
            return []

        # Find documents containing all query terms (AND query)
        posting_lists = []
        for term in query_terms:
            if term in inverted_index:
                posting_lists.append(inverted_index[term])

        if not posting_lists:
            return []

        # Intersect posting lists
        result_docs = self._intersect_postings(posting_lists)

        return [doc_id for doc_id, _, _ in result_docs]

    def _intersect_postings(self, posting_lists: List[List[Tuple[int, int, List[int]]]]) -> List[Tuple[int, int, List[int]]]:
        """Intersect multiple posting lists"""
        if not posting_lists:
            return []

        # Start with first posting list
        result = posting_lists[0][:]

        # Intersect with remaining lists
        for postings in posting_lists[1:]:
            result = self._intersect_two_postings(result, postings)

        return result

    def _intersect_two_postings(self, list1: List[Tuple[int, int, List[int]]],
                               list2: List[Tuple[int, int, List[int]]]) -> List[Tuple[int, int, List[int]]]:
        """Intersect two posting lists"""
        result = []
        i, j = 0, 0

        while i < len(list1) and j < len(list2):
            doc1, freq1, pos1 = list1[i]
            doc2, freq2, pos2 = list2[j]

            if doc1 == doc2:
                result.append((doc1, min(freq1, freq2), pos1))  # Simplified
                i += 1
                j += 1
            elif doc1 < doc2:
                i += 1
            else:
                j += 1

        return result

# Usage
builder = InvertedIndexBuilder()

# Sample documents
documents = [
    {
        'id': 1,
        'content': 'System design is important for software engineering interviews'
    },
    {
        'id': 2,
        'content': 'Design patterns help in building scalable systems'
    },
    {
        'id': 3,
        'content': 'System design interviews require knowledge of distributed systems'
    }
]

# Build inverted index
inverted_index = builder.build_index(documents)
print("Inverted Index:", inverted_index)

# Search
results = builder.search(inverted_index, "system design")
print("Search results:", results)
```

## PageRank Algorithm

### Distributed PageRank
```python
from typing import Dict, List, Set
import math
from collections import defaultdict

class PageRankCalculator:
    def __init__(self, damping_factor: float = 0.85, convergence_threshold: float = 1e-6):
        self.damping_factor = damping_factor
        self.convergence_threshold = convergence_threshold

    def calculate_pagerank(self, graph: Dict[int, List[int]],
                          max_iterations: int = 100) -> Dict[int, float]:
        """Calculate PageRank for all nodes in graph"""
        # Initialize PageRank values
        num_nodes = len(graph)
        pagerank = {node: 1.0 / num_nodes for node in graph}

        # Build reverse graph (incoming links)
        reverse_graph = self._build_reverse_graph(graph)

        for iteration in range(max_iterations):
            new_pagerank = {}
            total_diff = 0.0

            for node in graph:
                # Calculate PageRank contribution from incoming links
                sum_pr = 0.0
                if node in reverse_graph:
                    for incoming_node in reverse_graph[node]:
                        outgoing_links = len(graph[incoming_node])
                        if outgoing_links > 0:
                            sum_pr += pagerank[incoming_node] / outgoing_links

                # Apply damping factor
                new_pr = (1 - self.damping_factor) / num_nodes + self.damping_factor * sum_pr
                new_pagerank[node] = new_pr

                # Calculate convergence
                total_diff += abs(new_pr - pagerank[node])

            pagerank = new_pagerank

            # Check convergence
            if total_diff < self.convergence_threshold:
                print(f"Converged after {iteration + 1} iterations")
                break

        return pagerank

    def _build_reverse_graph(self, graph: Dict[int, List[int]]) -> Dict[int, List[int]]:
        """Build reverse graph for efficient PageRank calculation"""
        reverse_graph = defaultdict(list)

        for node, outgoing in graph.items():
            for target in outgoing:
                reverse_graph[target].append(node)

        return dict(reverse_graph)

    def calculate_pagerank_with_teleportation(self, graph: Dict[int, List[int]],
                                            personalization: Dict[int, float] = None) -> Dict[int, float]:
        """Calculate PageRank with teleportation (personalized PageRank)"""
        num_nodes = len(graph)

        # Initialize
        pagerank = {node: 1.0 / num_nodes for node in graph}

        # Use personalization vector if provided
        if personalization:
            total_personalization = sum(personalization.values())
            teleport_vector = {node: personalization.get(node, 0) / total_personalization
                             for node in graph}
        else:
            teleport_vector = {node: 1.0 / num_nodes for node in graph}

        reverse_graph = self._build_reverse_graph(graph)

        for iteration in range(100):
            new_pagerank = {}
            total_diff = 0.0

            for node in graph:
                sum_pr = 0.0
                if node in reverse_graph:
                    for incoming_node in reverse_graph[node]:
                        outgoing_links = len(graph[incoming_node])
                        if outgoing_links > 0:
                            sum_pr += pagerank[incoming_node] / outgoing_links

                # Apply teleportation
                teleport_pr = sum(pagerank[other] * teleport_vector[other]
                                for other in graph if other != node)

                new_pr = (1 - self.damping_factor) * teleport_vector[node] + self.damping_factor * sum_pr
                new_pagerank[node] = new_pr

                total_diff += abs(new_pr - pagerank[node])

            pagerank = new_pagerank

            if total_diff < self.convergence_threshold:
                break

        return pagerank

# Usage
# Sample web graph (node -> list of outgoing links)
web_graph = {
    1: [2, 3],      # Page 1 links to pages 2 and 3
    2: [3],         # Page 2 links to page 3
    3: [1],         # Page 3 links to page 1
    4: [1, 2, 3]    # Page 4 links to pages 1, 2, 3
}

calculator = PageRankCalculator()
pagerank_scores = calculator.calculate_pagerank(web_graph)

print("PageRank scores:")
for node, score in sorted(pagerank_scores.items()):
    print(f"Page {node}: {score:.4f}")

# Personalized PageRank (biased towards page 1)
personalization = {1: 1.0, 2: 0.0, 3: 0.0, 4: 0.0}
personalized_pr = calculator.calculate_pagerank_with_teleportation(web_graph, personalization)

print("\nPersonalized PageRank (biased to page 1):")
for node, score in sorted(personalized_pr.items()):
    print(f"Page {node}: {score:.4f}")
```

## Query Processing and Ranking

### BM25 Ranking Algorithm
```python
import math
from typing import List, Dict, Tuple
from collections import defaultdict

class BM25Ranker:
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        self.k1 = k1  # Term frequency saturation parameter
        self.b = b    # Length normalization parameter

    def rank_documents(self, query: List[str], documents: List[Dict],
                      inverted_index: Dict[str, List[Tuple[int, int, List[int]]]]) -> List[Tuple[int, float]]:
        """Rank documents using BM25"""
        # Pre-calculate document lengths and average length
        doc_lengths = {doc['id']: len(doc['content'].split()) for doc in documents}
        avg_doc_length = sum(doc_lengths.values()) / len(doc_lengths)

        # Calculate IDF for query terms
        idf_scores = self._calculate_idf(query, documents, inverted_index)

        # Calculate BM25 scores
        scores = []
        for doc in documents:
            doc_id = doc['id']
            score = 0.0

            for term in query:
                if term in inverted_index:
                    # Find term frequency in this document
                    tf = 0
                    for posting_doc_id, freq, _ in inverted_index[term]:
                        if posting_doc_id == doc_id:
                            tf = freq
                            break

                    if tf > 0:
                        # BM25 term score
                        numerator = idf_scores[term] * tf * (self.k1 + 1)
                        denominator = tf + self.k1 * (1 - self.b + self.b * (doc_lengths[doc_id] / avg_doc_length))
                        score += numerator / denominator

            scores.append((doc_id, score))

        # Sort by score (descending)
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores

    def _calculate_idf(self, query: List[str], documents: List[Dict],
                      inverted_index: Dict[str, List[Tuple[int, int, List[int]]]]) -> Dict[str, float]:
        """Calculate IDF scores for query terms"""
        num_docs = len(documents)
        idf_scores = {}

        for term in query:
            if term in inverted_index:
                # Number of documents containing the term
                df = len(inverted_index[term])
                idf_scores[term] = math.log((num_docs - df + 0.5) / (df + 0.5))
            else:
                idf_scores[term] = 0.0

        return idf_scores

class QueryProcessor:
    def __init__(self, ranker: BM25Ranker):
        self.ranker = ranker
        self.stop_words = {'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for', 'of', 'with', 'by'}

    def process_query(self, query_string: str, documents: List[Dict],
                     inverted_index: Dict[str, List[Tuple[int, int, List[int]]]]) -> List[Dict]:
        """Process search query and return ranked results"""
        # Tokenize and preprocess query
        query_terms = self._preprocess_query(query_string)

        if not query_terms:
            return []

        # Rank documents
        ranked_results = self.ranker.rank_documents(query_terms, documents, inverted_index)

        # Format results
        results = []
        for doc_id, score in ranked_results[:20]:  # Top 20 results
            doc = next((d for d in documents if d['id'] == doc_id), None)
            if doc:
                results.append({
                    'id': doc_id,
                    'title': doc.get('title', ''),
                    'content': doc['content'][:200] + '...',  # Snippet
                    'score': score,
                    'url': doc.get('url', '')
                })

        return results

    def _preprocess_query(self, query: str) -> List[str]:
        """Preprocess query: tokenize, lowercase, remove stop words"""
        import re

        # Tokenize
        tokens = re.findall(r'\b\w+\b', query.lower())

        # Remove stop words
        filtered_tokens = [token for token in tokens if token not in self.stop_words]

        return filtered_tokens

# Usage
ranker = BM25Ranker()
processor = QueryProcessor(ranker)

# Sample documents
documents = [
    {
        'id': 1,
        'title': 'System Design Basics',
        'content': 'System design is crucial for building scalable applications. It involves understanding distributed systems, databases, and networking.',
        'url': 'https://example.com/system-design'
    },
    {
        'id': 2,
        'title': 'Database Design',
        'content': 'Database design principles include normalization, indexing, and choosing the right database for your use case.',
        'url': 'https://example.com/database-design'
    },
    {
        'id': 3,
        'title': 'API Design',
        'content': 'RESTful API design involves proper HTTP methods, status codes, and resource modeling.',
        'url': 'https://example.com/api-design'
    }
]

# Build inverted index (simplified)
inverted_index = {
    'system': [(1, 1, [0]), (2, 1, [5])],
    'design': [(1, 1, [1]), (2, 1, [1]), (3, 1, [1])],
    'database': [(2, 1, [0])],
    'api': [(3, 1, [0])],
    'scalable': [(1, 1, [6])],
    'distributed': [(1, 1, [9])]
}

# Process query
query = "system design"
results = processor.process_query(query, documents, inverted_index)

print(f"Query: {query}")
print("Results:")
for result in results:
    print(f"- {result['title']} (Score: {result['score']:.3f})")
```

## Real-World Examples

### Google Search Architecture
```
Components:
- Distributed crawling infrastructure
- Massive inverted index storage
- PageRank and ML ranking
- Global data centers
- Real-time indexing updates

Technologies:
- C++ for performance-critical components
- Bigtable for storage
- MapReduce for processing
- Custom networking protocols
- Machine learning for ranking
```

### Elasticsearch Architecture
```
Components:
- Distributed Lucene indexes
- RESTful API layer
- Cluster coordination
- Real-time indexing
- Advanced query DSL

Technologies:
- Java (Lucene)
- Custom distributed protocols
- JSON for data storage
- REST APIs
- Plugin architecture
```

## Interview Tips

### Common Questions
1. "How would you design a web crawler?"
2. "What's your approach to building an inverted index?"
3. "How would you implement PageRank?"
4. "How would you rank search results?"
5. "How would you scale search to billions of documents?"

### Answer Framework
1. **Requirements Clarification**: Scale, latency, freshness requirements
2. **High-Level Design**: Crawling, indexing, query processing
3. **Deep Dive**: Inverted index, ranking algorithms, distribution
4. **Scalability**: Sharding, replication, caching
5. **Edge Cases**: Duplicate content, spam, query optimization

### Key Points to Emphasize
- Distributed systems challenges
- Index construction and maintenance
- Ranking algorithm effectiveness
- Scalability for massive data
- Real-time indexing trade-offs

## Practice Problems

1. **Design web crawler with politeness**
2. **Implement inverted index construction**
3. **Create PageRank calculation**
4. **Build BM25 ranking system**
5. **Scale search infrastructure**

## Further Reading

- **Books**: "Information Retrieval" by Manning et al.
- **Papers**: "The Anatomy of a Large-Scale Hypertextual Web Search Engine"
- **Concepts**: Inverted Index, PageRank, BM25, Distributed Systems
