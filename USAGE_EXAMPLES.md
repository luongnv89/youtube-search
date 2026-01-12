# Usage Examples - Local and Render Deployment

This guide shows how to use the YouTube Search Library with both local proxy development and Render production deployment.

## Setup

### 1. Install the Library

```bash
npm install yt-search-lib
```

Or if developing locally:

```bash
npm install
```

### 2. Environment Setup

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env`:

```bash
# For local development
YOUTUBE_PROXY_URL=http://127.0.0.1:3000/proxy?url=
NODE_ENV=development
```

## Local Development

### Start Local Proxy

In one terminal:

```bash
npm run proxy:start
```

Output:
```
CORS Proxy Server running on http://127.0.0.1:3000
Use proxy URL: http://127.0.0.1:3000/proxy?url=
```

### Basic Search Example

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function basicSearch() {
  const client = new YouTubeClient({
    proxyUrl: 'http://127.0.0.1:3000/proxy?url='
  });

  // Search for videos
  const results = await client.search('lofi hip hop radio', { limit: 5 });

  results.forEach((video, i) => {
    console.log(`${i + 1}. ${video.title}`);
    console.log(`   Channel: ${video.author}`);
    console.log(`   URL: ${video.link}\n`);
  });
}

basicSearch().catch(console.error);
```

**Output:**
```
1. lofi hip hop radio 📚 beats to relax/study to
   Channel: Lofi Girl
   URL: https://www.youtube.com/watch?v=jfKfPfyJRdk

2. ❄️ Coffee Shop Radio  - 24/7 Chill Lo-Fi & Jazzy Beats
   Channel: STEEZYASFUCK
   URL: https://www.youtube.com/watch?v=blAFxjhg62k
...
```

### Search with Different Types

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function searchDifferentTypes() {
  const proxyUrl = 'http://127.0.0.1:3000/proxy?url=';

  // Search for videos
  console.log('=== Videos ===');
  let client = new YouTubeClient({ proxyUrl });
  let results = await client.search('JavaScript tutorial', { limit: 3, type: 'video' });
  results.forEach(r => console.log(`• ${r.title}`));

  // Search for channels
  console.log('\n=== Channels ===');
  client = new YouTubeClient({ proxyUrl });
  results = await client.search('Traversy Media', { limit: 3, type: 'channel' });
  results.forEach(r => console.log(`• ${r.title}`));

  // Search for playlists
  console.log('\n=== Playlists ===');
  client = new YouTubeClient({ proxyUrl });
  results = await client.search('JavaScript playlist', { limit: 3, type: 'playlist' });
  results.forEach(r => console.log(`• ${r.title}`));

  // Search for all types
  console.log('\n=== All Types ===');
  client = new YouTubeClient({ proxyUrl });
  results = await client.search('React', { limit: 5, type: 'all' });
  results.forEach(r => console.log(`• [${r.type}] ${r.title}`));
}

searchDifferentTypes().catch(console.error);
```

### Search with Caching

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function searchWithCaching() {
  const client = new YouTubeClient({
    proxyUrl: 'http://127.0.0.1:3000/proxy?url=',
    useCache: true  // Enable caching
  });

  const query = 'web development';

  // First search - hits YouTube
  console.log('First search (network request)...');
  let start = Date.now();
  let results = await client.search(query, { limit: 5 });
  console.log(`Time: ${Date.now() - start}ms`);
  console.log(`Results: ${results.length}`);

  // Second search - uses cache
  console.log('\nSecond search (from cache)...');
  start = Date.now();
  results = await client.search(query, { limit: 5 });
  console.log(`Time: ${Date.now() - start}ms`);  // Should be ~0ms
  console.log(`Results: ${results.length}`);
}

searchWithCaching().catch(console.error);
```

### Environment-Aware Configuration

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function createClient() {
  // Determine proxy URL based on environment
  const proxyUrl = process.env.YOUTUBE_PROXY_URL ||
    'http://127.0.0.1:3000/proxy?url=';

  const client = new YouTubeClient({
    proxyUrl: proxyUrl,
    useCache: true
  });

  const results = await client.search('coding music', { limit: 5 });
  return results;
}

const videos = await createClient();
console.log(videos);
```

## Production with Render

### Deploy to Render

Follow [RENDER_DEPLOYMENT.md](./RENDER_DEPLOYMENT.md) to deploy.

Your proxy URL will be something like:
```
https://youtube-proxy.onrender.com/proxy?url=
```

### Update .env for Production

```bash
# .env.production
YOUTUBE_PROXY_URL=https://youtube-proxy.onrender.com/proxy?url=
NODE_ENV=production
```

### Production Search Example

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function productionSearch() {
  // Use environment variable for proxy URL
  const proxyUrl = process.env.YOUTUBE_PROXY_URL;

  if (!proxyUrl) {
    throw new Error('YOUTUBE_PROXY_URL environment variable not set');
  }

  const client = new YouTubeClient({
    proxyUrl: proxyUrl,
    useCache: true
  });

  // Search with error handling
  try {
    const results = await client.search('lofi hip hop', { limit: 5 });
    console.log(`Found ${results.length} results`);
    return results;
  } catch (error) {
    console.error('Search failed:', error.message);
    throw error;
  }
}

export default productionSearch;
```

## Advanced Examples

### Search with Error Handling and Retry

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function searchWithRetry(query, options = {}, maxRetries = 3) {
  const {
    limit = 5,
    type = 'video',
    timeout = 10000,
    proxyUrl = process.env.YOUTUBE_PROXY_URL
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      console.log(`Attempt ${attempt}/${maxRetries}: Searching for "${query}"...`);

      const client = new YouTubeClient({
        proxyUrl: proxyUrl,
        useCache: true
      });

      // Create timeout promise
      const searchPromise = client.search(query, { limit, type });
      const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Search timeout')), timeout)
      );

      // Race between search and timeout
      const results = await Promise.race([searchPromise, timeoutPromise]);

      console.log(`✓ Success! Found ${results.length} results`);
      return results;
    } catch (error) {
      lastError = error;
      console.log(`✗ Failed: ${error.message}`);

      if (attempt < maxRetries) {
        // Exponential backoff
        const delay = Math.pow(2, attempt - 1) * 1000;
        console.log(`Retrying in ${delay}ms...\n`);
        await new Promise(r => setTimeout(r, delay));
      }
    }
  }

  throw new Error(`Search failed after ${maxRetries} attempts: ${lastError.message}`);
}

// Usage
searchWithRetry('JavaScript', { limit: 10 }).then(results => {
  console.log('Final results:', results);
}).catch(console.error);
```

### Multiple Searches in Parallel

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function parallelSearch(queries) {
  const proxyUrl = process.env.YOUTUBE_PROXY_URL ||
    'http://127.0.0.1:3000/proxy?url=';

  const client = new YouTubeClient({
    proxyUrl: proxyUrl,
    useCache: true
  });

  // Perform all searches in parallel
  const searchPromises = queries.map(query =>
    client.search(query, { limit: 3 })
      .then(results => ({ query, results }))
      .catch(error => ({ query, error: error.message }))
  );

  const results = await Promise.all(searchPromises);
  return results;
}

// Usage
const queries = ['lofi music', 'jazz', 'ambient'];
parallelSearch(queries).then(results => {
  results.forEach(({ query, results, error }) => {
    if (error) {
      console.log(`❌ ${query}: ${error}`);
    } else {
      console.log(`✓ ${query}: ${results.length} results`);
      results.forEach(r => console.log(`  - ${r.title}`));
    }
  });
});
```

### Search Results as JSON

```javascript
import { YouTubeClient } from 'yt-search-lib';

async function searchAsJSON(query, limit = 5) {
  const proxyUrl = process.env.YOUTUBE_PROXY_URL ||
    'http://127.0.0.1:3000/proxy?url=';

  const client = new YouTubeClient({
    proxyUrl: proxyUrl,
    useCache: true
  });

  const results = await client.search(query, { limit });

  return {
    query: query,
    timestamp: new Date().toISOString(),
    resultCount: results.length,
    results: results.map(video => ({
      id: video.id,
      title: video.title,
      author: video.author,
      link: video.link,
      type: video.type,
      thumbnail: video.thumbnail
    }))
  };
}

// Usage
const json = await searchAsJSON('coding tutorial', 10);
console.log(JSON.stringify(json, null, 2));
```

**Output:**
```json
{
  "query": "coding tutorial",
  "timestamp": "2024-01-12T10:30:00.000Z",
  "resultCount": 10,
  "results": [
    {
      "id": "jfKfPfyJRdk",
      "title": "JavaScript Tutorial for Beginners",
      "author": "Traversy Media",
      "link": "https://www.youtube.com/watch?v=jfKfPfyJRdk",
      "type": "video",
      "thumbnail": "https://i.ytimg.com/..."
    },
    ...
  ]
}
```

### Express.js Integration

```javascript
import express from 'express';
import { YouTubeClient } from 'yt-search-lib';

const app = express();
app.use(express.json());

// Initialize client once
const client = new YouTubeClient({
  proxyUrl: process.env.YOUTUBE_PROXY_URL || 'http://127.0.0.1:3000/proxy?url=',
  useCache: true
});

// Search endpoint
app.get('/api/search', async (req, res) => {
  const { q, limit = 5, type = 'video' } = req.query;

  if (!q) {
    return res.status(400).json({ error: 'Query parameter "q" is required' });
  }

  try {
    const results = await client.search(q, { limit: parseInt(limit), type });
    res.json({
      query: q,
      resultCount: results.length,
      results: results
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Video search endpoint
app.get('/api/videos', async (req, res) => {
  const { q, limit = 10 } = req.query;

  try {
    const results = await client.search(q, { limit: parseInt(limit), type: 'video' });
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Channel search endpoint
app.get('/api/channels', async (req, res) => {
  const { q, limit = 5 } = req.query;

  try {
    const results = await client.search(q, { limit: parseInt(limit), type: 'channel' });
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(3001, () => {
  console.log('Search API running on http://localhost:3001');
});
```

**Usage:**
```bash
curl "http://localhost:3001/api/search?q=lofi&limit=5"
curl "http://localhost:3001/api/videos?q=javascript"
curl "http://localhost:3001/api/channels?q=lofi"
```

### React Component Example

```jsx
import React, { useState } from 'react';
import { YouTubeClient } from 'yt-search-lib';

export function YouTubeSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const handleSearch = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError(null);

    try {
      const client = new YouTubeClient({
        proxyUrl: process.env.REACT_APP_YOUTUBE_PROXY_URL
      });

      const searchResults = await client.search(query, { limit: 10 });
      setResults(searchResults);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="search-container">
      <form onSubmit={handleSearch}>
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Search YouTube..."
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Searching...' : 'Search'}
        </button>
      </form>

      {error && <p className="error">{error}</p>}

      <div className="results">
        {results.map((video) => (
          <div key={video.id} className="video-card">
            <img src={video.thumbnail} alt={video.title} />
            <h3>{video.title}</h3>
            <p>{video.author}</p>
            <a href={video.link} target="_blank" rel="noopener noreferrer">
              Watch on YouTube
            </a>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Troubleshooting

### Proxy Connection Error

```javascript
// Error: Cannot connect to proxy

// Solution: Verify proxy is running
// Local: npm run proxy:start
// Or check YOUTUBE_PROXY_URL environment variable

const client = new YouTubeClient({
  proxyUrl: 'http://127.0.0.1:3000/proxy?url='  // Must be accessible
});
```

### Rate Limited

```javascript
// Solution: Add delay between searches
async function delayedSearch(queries) {
  const client = new YouTubeClient({
    proxyUrl: process.env.YOUTUBE_PROXY_URL
  });

  for (const query of queries) {
    const results = await client.search(query);
    console.log(results);

    // Wait 2 seconds between searches
    await new Promise(r => setTimeout(r, 2000));
  }
}
```

## Next Steps

1. **Local Development**: Follow local proxy examples
2. **Testing**: Use integration tests: `npm run test:integration:proxy`
3. **Deployment**: Deploy to Render using [RENDER_DEPLOYMENT.md](./RENDER_DEPLOYMENT.md)
4. **Production**: Update environment variables and use production proxy URL

---

For more information, see:
- [PROXY_SETUP.md](./PROXY_SETUP.md) - Local proxy setup
- [RENDER_DEPLOYMENT.md](./RENDER_DEPLOYMENT.md) - Render deployment
- [RENDER_QUICK_START.md](./RENDER_QUICK_START.md) - Quick deployment guide
