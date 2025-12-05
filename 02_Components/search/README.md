# Search and Indexing Systems

## Definition

Search and indexing systems are technologies that enable efficient retrieval of information from large datasets by creating optimized data structures (indexes) and algorithms for fast querying and ranking of results.

## Search Architecture

### Basic Search Pipeline
```
User Query
    ↓
Query Processing
    ↓
Index Lookup
    ↓
Result Ranking
    ↓
Result Presentation
```

### Advanced Search Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                  Search System                              │
│                                                             │
│  ┌────────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │   Query        │  │   Index     │  │   Ranking       │   │
│  │ Processing     │  │   Manager   │  │   Engine        │   │
│  │                │  │             │  │                 │   │
│  │ Tokenization   │  │ Inverted    │  │ TF-IDF          │   │
│  │ Normalization  │  │ Index       │  │ BM25            │   │
│  │ Spell Check    │  │ Bloom Filter│  │ ML Models       │   │
│  │ Query Expansion│  │ Cache       │  │ Personalization │   │
│  └────────────────┘  └─────────────┘  └─────────────────┘   │
│           ↓              ↓              ↓                   │
│  ┌─────────────────────────────────────────────┐            │
│  │            Result Store                     │            │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐      │            │
│  │  │ Cache   │  │ Database│  │ Document│      │            │
│  │  │         │  │         │  │ Store   │      │            │
│  │  └─────────┘  └─────────┘  └─────────┘      │            │
│  └─────────────────────────────────────────────┘            │
│                     ↓                                       │
│                User Interface                               │
└─────────────────────────────────────────────────────────────┘
```

## Indexing Fundamentals

### Inverted Index
```
Documents:
Doc 1: "The quick brown fox jumps over the lazy dog"
Doc 2: "The lazy dog sleeps in the sun"
Doc 3: "Quick brown foxes are fast"

Inverted Index:
the: [Doc 1, Doc 2]
quick: [Doc 1, Doc 3]
brown: [Doc 1, Doc 3]
fox: [Doc 1]
jumps: [Doc 1]
over: [Doc 1]
lazy: [Doc 1, Doc 2]
dog: [Doc 1, Doc 2]
sleeps: [Doc 2]
in: [Doc 2]
sun: [Doc 2]
foxes: [Doc 3]
are: [Doc 3]
fast: [Doc 3]

With Positions:
the: [(Doc 1, [0]), (Doc 2, [0])]
quick: [(Doc 1, [1]), (Doc 3, [0])]
brown: [(Doc 1, [2]), (Doc 3, [1])]
fox: [(Doc 1, [3])]
jumps: [(Doc 1, [4])]
over: [(Doc 1, [5])]
lazy: [(Doc 1, [7]), (Doc 2, [4])]
dog: [(Doc 1, [8]), (Doc 2, [5])]
```

### Inverted Index Implementation
```python
import re
from collections import defaultdict, Counter
import pickle
import os

class InvertedIndex:
    def __init__(self):
        self.index = defaultdict(list)
        self.documents = {}
        self.document_count = 0
        self.stop_words = {
            'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for',
            'of', 'with', 'by', 'is', 'are', 'was', 'were', 'be', 'been',
            'have', 'has', 'had', 'do', 'does', 'did', 'will', 'would'
        }

    def tokenize(self, text):
        # Convert to lowercase and extract words
        tokens = re.findall(r'\b\w+\b', text.lower())

        # Remove stop words and filter short tokens
        tokens = [token for token in tokens
                 if token not in self.stop_words and len(token) > 2]

        return tokens

    def add_document(self, doc_id, content):
        self.documents[doc_id] = content
        tokens = self.tokenize(content)

        # Build position index
        token_positions = defaultdict(list)
        for position, token in enumerate(tokens):
            token_positions[token].append(position)

        # Add to inverted index
        for token, positions in token_positions.items():
            self.index[token].append((doc_id, positions))

        self.document_count += 1

    def build_index(self, documents):
        """Build index from document collection"""
        for doc_id, content in documents.items():
            self.add_document(doc_id, content)

    def search(self, query, limit=10):
        query_tokens = self.tokenize(query)

        if not query_tokens:
            return []

        # Find documents containing all query terms
        result_docs = set()

        for token in query_tokens:
            if token in self.index:
                token_docs = set(doc_id for doc_id, _ in self.index[token])
                if not result_docs:
                    result_docs = token_docs
                else:
                    result_docs &= token_docs
            else:
                # Token not found in index
                return []

        # Calculate relevance scores
        scored_results = []
        for doc_id in result_docs:
            score = self._calculate_relevance(doc_id, query_tokens)
            scored_results.append((doc_id, score))

        # Sort by score and return top results
        scored_results.sort(key=lambda x: x[1], reverse=True)

        return [(doc_id, self.documents[doc_id], score)
                for doc_id, score in scored_results[:limit]]

    def _calculate_relevance(self, doc_id, query_tokens):
        """Simple TF-IDF based relevance scoring"""
        score = 0.0
        doc_content = self.documents[doc_id]
        doc_tokens = self.tokenize(doc_content)
        doc_length = len(doc_tokens)

        for token in query_tokens:
            if token in self.index:
                # Term Frequency (TF)
                tf = sum(1 for doc, _ in self.index[token] if doc == doc_id)
                tf_normalized = tf / doc_length

                # Inverse Document Frequency (IDF)
                df = len(set(doc for doc, _ in self.index[token]))
                idf = math.log(self.document_count / df)

                score += tf_normalized * idf

        return score

    def save_index(self, filepath):
        """Save index to disk"""
        index_data = {
            'index': dict(self.index),
            'documents': self.documents,
            'document_count': self.document_count
        }

        with open(filepath, 'wb') as f:
            pickle.dump(index_data, f)

    def load_index(self, filepath):
        """Load index from disk"""
        if os.path.exists(filepath):
            with open(filepath, 'rb') as f:
                index_data = pickle.load(f)
                self.index = defaultdict(list, index_data['index'])
                self.documents = index_data['documents']
                self.document_count = index_data['document_count']

# Usage
index = InvertedIndex()

# Add documents
documents = {
    1: "The quick brown fox jumps over the lazy dog",
    2: "The lazy dog sleeps in the sun",
    3: "Quick brown foxes are fast and agile",
    4: "Search engines use inverted indexes for fast retrieval"
}

index.build_index(documents)

# Search
results = index.search("quick fox")
for doc_id, content, score in results:
    print(f"Doc {doc_id} (Score: {score:.3f}): {content}")
```

## Advanced Search Features

### Spell Correction
```python
import editdistance

class SpellChecker:
    def __init__(self, dictionary):
        self.dictionary = set(dictionary)
        self.word_freq = Counter(dictionary)

    def suggest_corrections(self, word, max_suggestions=3):
        if word in self.dictionary:
            return [word]

        suggestions = []

        # Find words with minimum edit distance
        for dict_word in self.dictionary:
            distance = editdistance.eval(word, dict_word)
            if distance <= 2:  # Maximum edit distance of 2
                suggestions.append((dict_word, distance))

        # Sort by edit distance and frequency
        suggestions.sort(key=lambda x: (x[1], -self.word_freq[x[0]]))

        return [word for word, _ in suggestions[:max_suggestions]]

    def correct_query(self, query):
        tokens = query.split()
        corrected_tokens = []

        for token in tokens:
            suggestions = self.suggest_corrections(token)
            corrected_tokens.append(suggestions[0] if suggestions else token)

        return ' '.join(corrected_tokens)

# Usage
dictionary = ["quick", "brown", "fox", "jumps", "lazy", "dog", "search", "engine"]
spell_checker = SpellChecker(dictionary)

corrected_query = spell_checker.correct_query("quik brwn fox")
print(f"Corrected query: {corrected_query}")
```

### Auto-complete and Suggestions
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end_of_word = False
        self.frequency = 0
        self.suggestions = []

class AutoCompleteTrie:
    def __init__(self, max_suggestions=10):
        self.root = TrieNode()
        self.max_suggestions = max_suggestions

    def insert(self, word, frequency=1):
        node = self.root

        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        node.is_end_of_word = True
        node.frequency += frequency

    def _collect_suggestions(self, node, prefix, suggestions):
        if node.is_end_of_word:
            suggestions.append((prefix, node.frequency))

        # Sort children by character for consistent results
        for char in sorted(node.children.keys()):
            self._collect_suggestions(node.children[char], prefix + char, suggestions)

    def get_suggestions(self, prefix, limit=None):
        if limit is None:
            limit = self.max_suggestions

        node = self.root

        # Navigate to prefix
        for char in prefix:
            if char not in node.children:
                return []
            node = node.children[char]

        # Collect suggestions
        suggestions = []
        self._collect_suggestions(node, prefix, suggestions)

        # Sort by frequency and return top results
        suggestions.sort(key=lambda x: x[1], reverse=True)

        return [word for word, _ in suggestions[:limit]]

    def build_from_corpus(self, corpus):
        """Build trie from text corpus"""
        word_freq = Counter(corpus.lower().split())

        for word, freq in word_freq.items():
            if len(word) > 2:  # Filter very short words
                self.insert(word, freq)

# Usage
corpus = """
The quick brown fox jumps over the lazy dog
Search engines use inverted indexes
Auto completion helps users type faster
Quick brown foxes are agile
"""
auto_complete = AutoCompleteTrie()
auto_complete.build_from_corpus(corpus)

suggestions = auto_complete.get_suggestions("qu")
print(f"Suggestions for 'qu': {suggestions}")
```

### Faceted Search
```python
class FacetedSearch:
    def __init__(self):
        self.documents = {}
        self.facets = {}
        self.index = InvertedIndex()

    def add_document(self, doc_id, content, facets):
        self.documents[doc_id] = {
            'content': content,
            'facets': facets
        }

        # Add to main index
        self.index.add_document(doc_id, content)

        # Build facet indexes
        for facet_name, facet_value in facets.items():
            if facet_name not in self.facets:
                self.facets[facet_name] = defaultdict(list)

            self.facets[facet_name][facet_value].append(doc_id)

    def search_with_facets(self, query, facet_filters=None, limit=10):
        # Perform basic search
        results = self.index.search(query, limit=100)

        # Apply facet filters
        if facet_filters:
            filtered_results = []

            for doc_id, content, score in results:
                doc_facets = self.documents[doc_id]['facets']
                matches_all_facets = True

                for facet_name, allowed_values in facet_filters.items():
                    if facet_name in doc_facets:
                        if doc_facets[facet_name] not in allowed_values:
                            matches_all_facets = False
                            break

                if matches_all_facets:
                    filtered_results.append((doc_id, content, score))

            results = filtered_results

        # Calculate facet counts for results
        facet_counts = self._calculate_facet_counts(results)

        return {
            'results': results[:limit],
            'facet_counts': facet_counts,
            'total_found': len(results)
        }

    def _calculate_facet_counts(self, results):
        facet_counts = {}

        for facet_name in self.facets.keys():
            facet_counts[facet_name] = Counter()

            for doc_id, _, _ in results:
                doc_facets = self.documents[doc_id]['facets']
                if facet_name in doc_facets:
                    facet_value = doc_facets[facet_name]
                    facet_counts[facet_name][facet_value] += 1

        return facet_counts

# Usage
faceted_search = FacetedSearch()

# Add documents with facets
faceted_search.add_document(1, "iPhone 13 Pro review", {
    'category': 'Electronics',
    'brand': 'Apple',
    'price_range': 'High'
})

faceted_search.add_document(2, "Samsung Galaxy S21 comparison", {
    'category': 'Electronics',
    'brand': 'Samsung',
    'price_range': 'High'
})

faceted_search.add_document(3, "Budget smartphone guide", {
    'category': 'Electronics',
    'brand': 'Various',
    'price_range': 'Low'
})

# Search with facets
results = faceted_search.search_with_facets(
    "smartphone",
    facet_filters={'category': ['Electronics']}
)

print("Search Results:")
for doc_id, content, score in results['results']:
    print(f"Doc {doc_id}: {content}")

print("\nFacet Counts:")
for facet_name, counts in results['facet_counts'].items():
    print(f"{facet_name}: {dict(counts)}")
```

## Search Ranking Algorithms

### TF-IDF (Term Frequency-Inverse Document Frequency)
```python
import math
from collections import Counter, defaultdict

class TFIDFRanker:
    def __init__(self):
        self.documents = {}
        self.document_count = 0
        self.term_document_frequency = defaultdict(int)
        self.term_frequency = defaultdict(lambda: defaultdict(int))

    def add_document(self, doc_id, content):
        self.documents[doc_id] = content
        self.document_count += 1

        # Calculate term frequencies
        terms = self._tokenize(content)
        term_counts = Counter(terms)

        for term, count in term_counts.items():
            self.term_frequency[doc_id][term] = count
            self.term_document_frequency[term] += 1

    def _tokenize(self, text):
        # Simple tokenization
        return re.findall(r'\b\w+\b', text.lower())

    def calculate_tf_idf(self, doc_id, term):
        """Calculate TF-IDF score for a term in a document"""
        if doc_id not in self.term_frequency:
            return 0.0

        # Term Frequency (TF)
        tf = self.term_frequency[doc_id][term]
        max_tf = max(self.term_frequency[doc_id].values())
        normalized_tf = tf / max_tf if max_tf > 0 else 0

        # Inverse Document Frequency (IDF)
        df = self.term_document_frequency[term]
        idf = math.log(self.document_count / df) if df > 0 else 0

        return normalized_tf * idf

    def search(self, query, limit=10):
        query_terms = self._tokenize(query)

        # Calculate scores for all documents
        doc_scores = defaultdict(float)

        for doc_id in self.documents.keys():
            score = 0.0

            for term in query_terms:
                score += self.calculate_tf_idf(doc_id, term)

            doc_scores[doc_id] = score

        # Sort by score and return top results
        sorted_results = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)

        return [(doc_id, self.documents[doc_id], score)
                for doc_id, score in sorted_results[:limit]]

# Usage
tfidf_ranker = TFIDFRanker()

documents = {
    1: "The cat sat on the mat",
    2: "The dog ate the cat food",
    3: "The cat and dog are friends",
    4: "The mat is blue and soft"
}

for doc_id, content in documents.items():
    tfidf_ranker.add_document(doc_id, content)

results = tfidf_ranker.search("cat dog")
for doc_id, content, score in results:
    print(f"Doc {doc_id} (Score: {score:.3f}): {content}")
```

### BM25 Ranking Algorithm
```python
class BM25Ranker:
    def __init__(self, k1=1.2, b=0.75):
        self.k1 = k1  # Controls term frequency saturation
        self.b = b    # Controls document length normalization
        self.documents = {}
        self.document_lengths = {}
        self.avg_document_length = 0
        self.term_document_frequency = defaultdict(int)
        self.term_frequency = defaultdict(lambda: defaultdict(int))

    def add_document(self, doc_id, content):
        self.documents[doc_id] = content
        terms = self._tokenize(content)

        self.document_lengths[doc_id] = len(terms)
        term_counts = Counter(terms)

        for term, count in term_counts.items():
            self.term_frequency[doc_id][term] = count
            self.term_document_frequency[term] += 1

    def _tokenize(self, text):
        return re.findall(r'\b\w+\b', text.lower())

    def calculate_average_document_length(self):
        if self.document_lengths:
            self.avg_document_length = sum(self.document_lengths.values()) / len(self.document_lengths)

    def calculate_bm25_score(self, doc_id, query_terms):
        """Calculate BM25 score for a document"""
        if doc_id not in self.document_lengths:
            return 0.0

        doc_length = self.document_lengths[doc_id]
        score = 0.0

        for term in query_terms:
            # Term frequency in document
            tf = self.term_frequency[doc_id].get(term, 0)

            # Document frequency
            df = self.term_document_frequency[term]
            if df == 0:
                continue

            # IDF component
            N = len(self.documents)
            idf = math.log((N - df + 0.5) / (df + 0.5))

            # TF component with saturation
            tf_component = (tf * (self.k1 + 1)) / (
                tf + self.k1 * (1 - self.b + self.b * doc_length / self.avg_document_length)
            )

            score += idf * tf_component

        return score

    def search(self, query, limit=10):
        self.calculate_average_document_length()
        query_terms = self._tokenize(query)

        # Calculate scores for all documents
        doc_scores = {}

        for doc_id in self.documents.keys():
            score = self.calculate_bm25_score(doc_id, query_terms)
            if score > 0:
                doc_scores[doc_id] = score

        # Sort by score and return top results
        sorted_results = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)

        return [(doc_id, self.documents[doc_id], score)
                for doc_id, score in sorted_results[:limit]]

# Usage
bm25_ranker = BM25Ranker()

documents = {
    1: "The cat sat on the mat",
    2: "The dog ate the cat food",
    3: "The cat and dog are friends",
    4: "The mat is blue and soft",
    5: "Cats and dogs are popular pets"
}

for doc_id, content in documents.items():
    bm25_ranker.add_document(doc_id, content)

results = bm25_ranker.search("cat dog")
for doc_id, content, score in results:
    print(f"Doc {doc_id} (Score: {score:.3f}): {content}")
```

## Search System Architecture

### Distributed Search
```python
import hashlib
from concurrent.futures import ThreadPoolExecutor

class DistributedSearchEngine:
    def __init__(self, num_shards=4):
        self.num_shards = num_shards
        self.shards = [InvertedIndex() for _ in range(num_shards)]

    def _get_shard_id(self, doc_id):
        """Determine shard for a document"""
        return int(hashlib.md5(str(doc_id).encode()).hexdigest(), 16) % self.num_shards

    def add_document(self, doc_id, content):
        shard_id = self._get_shard_id(doc_id)
        self.shards[shard_id].add_document(doc_id, content)

    def search(self, query, limit=10):
        # Search all shards in parallel
        with ThreadPoolExecutor(max_workers=self.num_shards) as executor:
            futures = [
                executor.submit(shard.search, query, limit)
                for shard in self.shards
            ]

            all_results = []
            for future in futures:
                shard_results = future.result()
                all_results.extend(shard_results)

        # Merge and rank results
        all_results.sort(key=lambda x: x[2], reverse=True)

        return all_results[:limit]

    def get_shard_stats(self):
        """Get statistics for each shard"""
        stats = []
        for i, shard in enumerate(self.shards):
            stats.append({
                'shard_id': i,
                'document_count': shard.document_count,
                'index_size': len(shard.index)
            })
        return stats

# Usage
distributed_search = DistributedSearchEngine(num_shards=4)

# Add documents
for doc_id, content in documents.items():
    distributed_search.add_document(doc_id, content)

# Search across all shards
results = distributed_search.search("cat dog")
for doc_id, content, score in results:
    print(f"Doc {doc_id} (Score: {score:.3f}): {content}")

# Check shard distribution
stats = distributed_search.get_shard_stats()
for stat in stats:
    print(f"Shard {stat['shard_id']}: {stat['document_count']} documents")
```

### Real-time Search Updates
```python
import threading
import queue
import time

class RealTimeSearchEngine:
    def __init__(self):
        self.index = InvertedIndex()
        self.update_queue = queue.Queue()
        self.running = False
        self.update_thread = None

    def start(self):
        """Start the real-time update thread"""
        self.running = True
        self.update_thread = threading.Thread(target=self._process_updates)
        self.update_thread.daemon = True
        self.update_thread.start()

    def stop(self):
        """Stop the real-time update thread"""
        self.running = False
        if self.update_thread:
            self.update_thread.join()

    def add_document(self, doc_id, content):
        """Add document to the update queue"""
        self.update_queue.put(('add', doc_id, content))

    def update_document(self, doc_id, content):
        """Update document in the update queue"""
        self.update_queue.put(('update', doc_id, content))

    def delete_document(self, doc_id):
        """Delete document from the update queue"""
        self.update_queue.put(('delete', doc_id, None))

    def _process_updates(self):
        """Process updates in background thread"""
        while self.running:
            try:
                # Get update from queue (with timeout)
                operation, doc_id, content = self.update_queue.get(timeout=1)

                if operation == 'add':
                    self.index.add_document(doc_id, content)
                elif operation == 'update':
                    # Remove old version and add new version
                    self._remove_document(doc_id)
                    self.index.add_document(doc_id, content)
                elif operation == 'delete':
                    self._remove_document(doc_id)

                self.update_queue.task_done()

            except queue.Empty:
                continue
            except Exception as e:
                print(f"Error processing update: {e}")

    def _remove_document(self, doc_id):
        """Remove document from index"""
        if doc_id in self.index.documents:
            content = self.index.documents[doc_id]
            tokens = self.index.tokenize(content)

            # Remove from inverted index
            for token in set(tokens):
                self.index.index[token] = [
                    (doc_id, positions) for doc_id, positions in self.index.index[token]
                    if doc_id != doc_id
                ]

                # Remove empty entries
                if not self.index.index[token]:
                    del self.index.index[token]

            # Remove from documents
            del self.index.documents[doc_id]
            self.index.document_count -= 1

    def search(self, query, limit=10):
        """Search with real-time index"""
        return self.index.search(query, limit)

# Usage
realtime_search = RealTimeSearchEngine()
realtime_search.start()

# Add initial documents
for doc_id, content in documents.items():
    realtime_search.add_document(doc_id, content)

# Perform search
results = realtime_search.search("cat")
print("Initial search results:")
for doc_id, content, score in results:
    print(f"Doc {doc_id}: {content}")

# Update document in real-time
realtime_search.update_document(1, "The black cat sat on the red mat")

# Wait for update to process
time.sleep(2)

# Search again
results = realtime_search.search("cat")
print("\nSearch results after update:")
for doc_id, content, score in results:
    print(f"Doc {doc_id}: {content}")

realtime_search.stop()
```

## Search Analytics

### Query Analytics
```python
class SearchAnalytics:
    def __init__(self):
        self.query_log = []
        self.click_log = []
        self.query_stats = {}

    def log_query(self, query, results_count, user_id=None, timestamp=None):
        """Log search query"""
        if timestamp is None:
            timestamp = time.time()

        log_entry = {
            'query': query,
            'results_count': results_count,
            'user_id': user_id,
            'timestamp': timestamp
        }

        self.query_log.append(log_entry)
        self._update_query_stats(query)

    def log_click(self, query, doc_id, position, user_id=None, timestamp=None):
        """Log result click"""
        if timestamp is None:
            timestamp = time.time()

        log_entry = {
            'query': query,
            'doc_id': doc_id,
            'position': position,
            'user_id': user_id,
            'timestamp': timestamp
        }

        self.click_log.append(log_entry)

    def _update_query_stats(self, query):
        """Update query statistics"""
        if query not in self.query_stats:
            self.query_stats[query] = {
                'count': 0,
                'first_seen': time.time(),
                'last_seen': time.time()
            }

        self.query_stats[query]['count'] += 1
        self.query_stats[query]['last_seen'] = time.time()

    def get_popular_queries(self, limit=10):
        """Get most popular queries"""
        sorted_queries = sorted(
            self.query_stats.items(),
            key=lambda x: x[1]['count'],
            reverse=True
        )

        return sorted_queries[:limit]

    def get_no_result_queries(self):
        """Get queries that returned no results"""
        no_result_queries = []

        for log_entry in self.query_log:
            if log_entry['results_count'] == 0:
                no_result_queries.append(log_entry['query'])

        return Counter(no_result_queries).most_common()

    def calculate_ctr(self, query):
        """Calculate click-through rate for a query"""
        query_count = sum(1 for log in self.query_log if log['query'] == query)
        click_count = sum(1 for log in self.click_log if log['query'] == query)

        if query_count == 0:
            return 0.0

        return click_count / query_count

    def get_search_performance_metrics(self):
        """Calculate search performance metrics"""
        total_queries = len(self.query_log)
        total_clicks = len(self.click_log)

        # Average results per query
        avg_results = sum(log['results_count'] for log in self.query_log) / total_queries if total_queries > 0 else 0

        # Overall CTR
        overall_ctr = total_clicks / total_queries if total_queries > 0 else 0

        # Zero result rate
        zero_result_rate = sum(1 for log in self.query_log if log['results_count'] == 0) / total_queries if total_queries > 0 else 0

        return {
            'total_queries': total_queries,
            'total_clicks': total_clicks,
            'avg_results_per_query': avg_results,
            'overall_ctr': overall_ctr,
            'zero_result_rate': zero_result_rate,
            'unique_queries': len(self.query_stats)
        }

# Usage
analytics = SearchAnalytics()

# Log some searches
analytics.log_query("cat", 5, "user1")
analytics.log_query("dog", 3, "user1")
analytics.log_query("cat", 5, "user2")
analytics.log_query("nonexistent", 0, "user3")

# Log some clicks
analytics.log_click("cat", 1, 1, "user1")
analytics.log_click("cat", 2, 2, "user2")
analytics.log_click("dog", 1, 1, "user1")

# Get analytics
popular_queries = analytics.get_popular_queries(5)
print("Popular Queries:")
for query, stats in popular_queries:
    print(f"  {query}: {stats['count']} searches")

no_result_queries = analytics.get_no_result_queries()
print("\nNo Result Queries:")
for query, count in no_result_queries:
    print(f"  {query}: {count} times")

metrics = analytics.get_search_performance_metrics()
print(f"\nPerformance Metrics: {metrics}")
```

## Real-World Examples

### Google Search Architecture
```
Components:
- Multiple data centers
- Distributed indexing
- PageRank algorithm
- Machine learning ranking
- Real-time updates

Features:
- Web crawling
- Index sharding
- Query understanding
- Personalization
```

### Elasticsearch Architecture
```
Components:
- Lucene indexes
- Cluster management
- Real-time indexing
- Distributed search
- REST API

Features:
- Full-text search
- Aggregations
- Geo search
- Percolation
```

### Amazon Search Architecture
```
Components:
- A9 search engine
- Product catalog
- User behavior analysis
- Personalized ranking
- Auto-complete

Features:
- Faceted search
- Product recommendations
- Search analytics
- A/B testing
```

## Interview Tips

### Common Questions
1. "How does a search engine work?"
2. "What is an inverted index?"
3. "How would you implement auto-complete?"
4. "How do you rank search results?"
5. "How would you scale a search system?"

### Answer Framework
1. **Explain Architecture**: Indexing, query processing, ranking
2. **Discuss Indexing**: Inverted index, tokenization, stemming
3. **Cover Ranking**: TF-IDF, BM25, machine learning
4. **Address Scaling**: Distributed search, sharding, caching
5. **Consider Features**: Auto-complete, spell correction, faceting

### Key Points to Emphasize
- Index structure and efficiency
- Ranking algorithms and relevance
- Scalability and performance
- User experience features
- Analytics and optimization

## Practice Problems

1. **Design a web search engine**
2. **Implement auto-complete system**
3. **Create e-commerce product search**
4. **Design real-time search updates**
5. **Build search analytics system**

## Further Reading

- **Books**: "Introduction to Information Retrieval" by Manning, "Search Engines: Information Retrieval in Practice" by Croft
- **Papers**: "The PageRank Citation Ranking" by Brin and Page, "BM25" by Robertson and Walker
- **Documentation**: Elasticsearch Guide, Apache Solr Documentation, Lucene Documentation
