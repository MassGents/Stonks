<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>StockScope Pro</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap');
  body {
    margin: 0;
    font-family: 'Inter', sans-serif;
    background: url('https://images.unsplash.com/photo-1504384308090-c894fdcc538d?auto=format&fit=crop&w=1470&q=80') no-repeat center center fixed;
    background-size: cover;
    color: #222;
  }
  header {
    text-align: center;
    padding: 2rem 1rem 1rem;
    background: rgba(255 255 255 / 0.85);
    box-shadow: 0 2px 8px rgb(0 0 0 / 0.15);
  }
  header h1 {
    margin: 0;
    font-weight: 600;
    color: #1e3a8a;
  }
  nav {
    display: flex;
    justify-content: center;
    gap: 1rem;
    background: rgba(255 255 255 / 0.75);
    padding: 1rem 0;
    box-shadow: 0 1px 5px rgb(0 0 0 / 0.1);
    position: sticky;
    top: 0;
    z-index: 10;
  }
  nav a {
    color: #1e3a8a;
    font-weight: 600;
    text-decoration: none;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    transition: background-color 0.3s ease;
  }
  nav a:hover, nav a.active {
    background-color: #1e3a8a;
    color: #fff;
  }
  main {
    max-width: 1000px;
    margin: 2rem auto;
    background: rgba(255 255 255 / 0.9);
    padding: 1.5rem 2rem;
    border-radius: 12px;
    box-shadow: 0 4px 12px rgb(0 0 0 / 0.1);
  }
  section {
    display: none;
  }
  section.active {
    display: block;
  }
  h2 {
    color: #2563eb;
    margin-bottom: 1rem;
  }
  .stock-list, .news-list {
    list-style: none;
    padding: 0;
  }
  .stock-list li, .news-list li {
    padding: 0.75rem 1rem;
    border-bottom: 1px solid #ddd;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .stock-list li:last-child, .news-list li:last-child {
    border-bottom: none;
  }
  .stock-symbol {
    font-weight: 700;
    color: #1e40af;
  }
  .stock-price {
    font-weight: 600;
    color: #047857;
  }
  .news-item a {
    color: #1e40af;
    text-decoration: none;
    font-weight: 600;
  }
  .news-item a:hover {
    text-decoration: underline;
  }
  .loading {
    text-align: center;
    font-style: italic;
    color: #555;
  }
  footer {
    text-align: center;
    padding: 1rem 0;
    font-size: 0.9rem;
    color: #555;
  }
</style>
</head>
<body>

<header>
  <h1>StockScope Pro</h1>
</header>

<nav>
  <a href="#" class="active" data-section="prices">Prices</a>
  <a href="#" data-section="news">News</a>
  <a href="#" data-section="recommendations">Recommendations</a>
  <a href="#" data-section="about">About</a>
</nav>

<main>
  <section id="prices" class="active">
    <h2>Live Stock Prices</h2>
    <p class="loading" id="prices-loading">Loading stock prices...</p>
    <ul class="stock-list" id="stock-list"></ul>
  </section>

  <section id="news">
    <h2>Latest Market News</h2>
    <p class="loading" id="news-loading">Loading news...</p>
    <ul class="news-list" id="news-list"></ul>
  </section>

  <section id="recommendations">
    <h2>Recommendations</h2>
    <p>Top trending stocks based on recent price changes:</p>
    <ul class="stock-list" id="recommendations-list"></ul>
  </section>

  <section id="about">
    <h2>About StockScope Pro</h2>
    <p>This site tracks real-time stock prices and news to help you stay informed. Powered by Twelve Data and NewsAPI.</p>
  </section>
</main>

<footer>
  &copy; 2025 StockScope Pro
</footer>

<script>
  const twelveApiKey = '5ef5daf5f4454f209b8a74ecd5907d92';
  const newsApiKey = 'fb972affa3a5480799e8ebebadc275ff';

  const navLinks = document.querySelectorAll('nav a');
  const sections = document.querySelectorAll('main section');

  navLinks.forEach(link => {
    link.addEventListener('click', e => {
      e.preventDefault();
      navLinks.forEach(l => l.classList.remove('active'));
      link.classList.add('active');
      const target = link.getAttribute('data-section');
      sections.forEach(sec => {
        sec.id === target ? sec.classList.add('active') : sec.classList.remove('active');
      });
    });
  });

  // Top popular stocks list (50 symbols) - you can customize or expand
  const popularStocks = [
    'AAPL','MSFT','GOOGL','AMZN','TSLA','NVDA','META','BRK.B','JPM','JNJ',
    'V','PG','MA','DIS','UNH','HD','BAC','ADBE','XOM','NFLX',
    'KO','CMCSA','PFE','NKE','CSCO','ABT','MRK','PEP','CRM','INTC',
    'T','WMT','ORCL','ACN','CVX','MCD','MDT','NEE','COST','BMY',
    'QCOM','TXN','LIN','LOW','AMGN','SBUX','BA','HON','IBM','MMM'
  ];

  async function fetchStockPrices() {
    const pricesList = document.getElementById('stock-list');
    const loading = document.getElementById('prices-loading');
    pricesList.innerHTML = '';
    loading.style.display = 'block';

    // Twelve Data limits batch size, so chunk the symbols to 10 each
    const chunks = [];
    for(let i=0; i < popularStocks.length; i += 10) {
      chunks.push(popularStocks.slice(i, i + 10));
    }

    let allData = [];
    try {
      for (const chunk of chunks) {
        const symbolsStr = chunk.join(',');
        const url = `https://api.twelvedata.com/quote?symbol=${symbolsStr}&apikey=${twelveApiKey}`;
        const res = await fetch(url);
        const data = await res.json();

        // If chunk length 1, data is an object, else array
        if (chunk.length === 1) {
          if (data.status === 'ok') allData.push(data);
        } else {
          if (data.status === 'ok' && Array.isArray(data)) {
            allData = allData.concat(data);
          }
        }
      }

      loading.style.display = 'none';

      if (!allData.length) {
        pricesList.innerHTML = '<li>No stock data available right now.</li>';
        return;
      }

      allData.forEach(stock => {
        const li = document.createElement('li');
        li.innerHTML = `<span class="stock-symbol">${stock.symbol}</span>
          <span class="stock-price">$${parseFloat(stock.close).toFixed(2)}</span>`;
        pricesList.appendChild(li);
      });

      // Also update recommendations based on price change percent (top gainers)
      updateRecommendations(allData);
    } catch (err) {
      loading.style.display = 'none';
      pricesList.innerHTML = `<li>Error loading stock prices: ${err.message}</li>`;
    }
  }

  async function fetchNews() {
    const newsList = document.getElementById('news-list');
    const loading = document.getElementById('news-loading');
    newsList.innerHTML = '';
    loading.style.display = 'block';

    try {
      const url = `https://newsapi.org/v2/everything?q=stock%20market&language=en&sortBy=publishedAt&pageSize=10&apiKey=${newsApiKey}`;
      const res = await fetch(url);
      const data = await res.json();

      loading.style.display = 'none';

      if (data.status !== 'ok' || !data.articles.length) {
        newsList.innerHTML = '<li>No recent news found.</li>';
        return;
      }

      data.articles.forEach(article => {
        const li = document.createElement('li');
        li.classList.add('news-item');
        li.innerHTML = `<a href="${article.url}" target="_blank" rel="noopener noreferrer">${article.title}</a>
          <small> — ${new Date(article.publishedAt).toLocaleDateString()}</small>`;
        newsList.appendChild(li);
      });
    } catch (err) {
      loading.style.display = 'none';
      newsList.innerHTML = `<li>Error loading news: ${err.message}</li>`;
    }
  }

  function updateRecommendations(stockData) {
    // Sort by highest percent change (day)
    const recList = document.getElementById('recommendations-list');
    recList.innerHTML = '';

    const gainers = stockData
      .filter(s => s.percent_change && !isNaN(parseFloat(s.percent_change)))
      .sort((a,b) => parseFloat(b.percent_change) - parseFloat(a.percent_change))
      .slice(0, 10);

    if (!gainers.length) {
      recList.innerHTML = '<li>No recommendation data available.</li>';
      return;
    }

    gainers.forEach(stock => {
      const li = document.createElement('li');
      li.innerHTML = `<span class="stock-symbol">${stock.symbol}</span> 
        <span class="stock-price" style="color:#16a34a;">+${parseFloat(stock.percent_change).toFixed(2)}%</span>`;
      recList.appendChild(li);
    });
  }

  // Initial data load
  fetchStockPrices();
  fetchNews();

  // Refresh prices & news every 5 minutes
  setInterval(fetchStockPrices, 300000);
  setInterval(fetchNews, 300000);
</script>

</body>
</html>
