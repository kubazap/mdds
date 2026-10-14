# MDDS - Malicious Domain Detection System

A real-time malicious domain detection system, developed as a practical application of the results of the thesis *"Analysis of the Effectiveness of Selected Machine Learning Models in Malicious Website Detection"*.

The project consists of two components:

- a **classification server** (FastAPI + XGBoost model),
- a **Chrome browser extension** that analyzes the domain of every visited page and displays a risk assessment based on a vector of 44 infrastructural, lexical, structural and reputation features.

![MDDS extension popup](extension/icons/UI.png)

## Project structure

```
MDDS/
├── extension/                     # Chrome extension, Manifest V3
│   ├── manifest.json              # Extension configuration and permissions
│   ├── background.js              # Tab monitoring and badge handling
│   ├── popup.html                 # Extension interface
│   ├── popup.js                   # API communication and result presentation
│   └── icons/                     # Extension icons
│
└── server/                        # FastAPI classification server
    ├── main.py                    # Application entry point and CORS configuration
    ├── config/
    │   └── settings.py            # Paths, reference lists and configuration
    ├── src/
    │   ├── api/
    │   │   └── router.py          # /health, /classify and /cache/clear endpoints
    │   └── core/
    │       └── classifier.py      # Model loading and prediction
    ├── features/
    │   ├── pipeline.py            # Aggregation of the 44 domain features
    │   ├── dns.py                 # DNS, SPF, DKIM and DMARC features
    │   ├── whois.py               # Domain age, registrar and country
    │   ├── http.py                # robots.txt, security.txt and WAF/CDN
    │   ├── lexical.py             # Lexical features of the domain name
    │   ├── levenshtein.py         # Similarity detection against brand names
    │   ├── url.py                 # Structural features of the URL
    │   └── tranco.py              # Tranco ranking features
    ├── models/                    # Model, feature list and metadata
    └── cache/                     # HTTP request cache
```

## Installation and setup

### Requirements

- Python 3.12+
- Google Chrome (or another Chromium-based browser that supports Manifest V3)

### 1. Clone the repository

```bash
git clone https://github.com/kubazap/MDDS
cd MDDS
```

### 2. Start the API server

```bash
cd server
pip install fastapi uvicorn scikit-learn pandas joblib dnspython python-whois requests requests-cache tldextract
python main.py
```

The server starts at `http://127.0.0.1:8000`, loading the XGBoost model and the Tranco Top 1M list (downloaded automatically on the first start and refreshed weekly).

### 3. Install the extension in Chrome

1. Open `chrome://extensions`.
2. Enable **Developer mode**.
3. Click **Load unpacked** and select the `extension/` directory.
4. Make sure the API server is running on port **8000**.

### 4. Verify that it works

Open any website and click the MDDS icon in the browser toolbar. The classification result for the analyzed domain should appear, together with a panel of detailed features.

## Features

### Chrome extension

- **Automatic classification** - on every tab change or page load, the domain is sent to the local API (`127.0.0.1:8000`).
- **Icon badge** - the `p_malicious` value as a percentage, shown directly on the extension icon and colored according to the risk threshold.
- **Popup panel** - a risk gauge, a verdict (Safe / Suspicious / Malicious) and the server connection status.
- **Details panel** - a grid of domain features: domain age, Tranco ranking position, registrar group, SPF/DKIM/DMARC, WAF/CDN, and the typosquatting detection result.
- **Two-level caching** - results are cached in the service worker and in `chrome.storage.session` (5-minute TTL), with the option to force a refresh (`force=true`).

The system uses a three-level risk scale based on the probability of the malicious class:

| Score          | Risk level    |
| -------------- | ------------- |
| below 30%      | safe          |
| 30% to 70%     | suspicious    |
| above 70%      | malicious     |

The risk scale complements the binary classification and helps highlight addresses that require additional verification.

### API server

| Endpoint           | Description                                                          |
| ------------------ | -------------------------------------------------------------------- |
| `GET /health`      | Server status, model type, number of features, cache size            |
| `GET /classify`    | Domain classification - extraction of 44 features and model prediction |
| `GET /cache/clear` | Clears the results cache and the DNS/WHOIS/HTTP caches               |

The server also implements an in-memory results cache (LRU, limit of 10,000 entries) and per-module extraction time measurement, returned in the `timings_ms` field of the response.

## Model features

The system uses 44 features from seven data sources, divided into four categories:

| Category           | Groups                              | Number of features | Acquisition method |
| ------------------ | ----------------------------------- | -----------------: | ------------------ |
| Infrastructural    | DNS, WHOIS, HTTP                    |                 24 | network queries    |
| Lexical            | domain name                         |                  9 | local analysis     |
| Structural         | URL, Levenshtein distance           |                  8 | local analysis     |
| Reputation         | Tranco ranking                      |                  3 | local analysis     |

### Feature groups

1. **DNS - 12 features**
   Information about domain resolution, A, AAAA, MX, NS and TXT records, and the SPF, DKIM and DMARC mechanisms.

2. **WHOIS - 6 features**
   Domain age, days until registration expiry, registration status, registrar group and country information.

3. **HTTP - 6 features**
   Availability of the `robots.txt` and `security.txt` resources, server responses, and the presence of WAF or CDN solutions.

4. **Lexical features - 9 features**
   Domain name length, share of digits, number of separators, Shannon entropy, suspicious words, brand names and TLD category.

5. **URL structure - 6 features**
   Address length, number of parameters, path depth, number of subdomains and the presence of percent-encoding.

6. **Levenshtein distance - 2 features**
   Similarity of the analyzed domain to protected brand names, used to detect typosquatting attempts.

7. **Tranco - 3 features**
   The domain's rank and its presence in the Tranco Top 1M and Top 10k rankings.

The highest ANOVA F-statistic values were obtained by the `domain_age_days` and `tranco_in_top1m` features, which indicates the high informational value of a domain's registration history and its presence in the popularity ranking.

## Technologies

### Server

- **Python 3.12**
- **FastAPI 0.111** - REST API layer, asynchronous request handling
- **scikit-learn 1.4** - ColumnTransformer, model quality evaluation
- **dnspython 2.6** - DNS queries, SPF/DKIM/DMARC verification
- **python-whois 0.9** - WHOIS registration data
- **requests** + **requests-cache 1.2** - HTTP requests with caching (SQLite)
- **tldextract** - parsing of domains and TLDs
- **joblib**, **pandas** - model loading and feature vector preparation

### Extension

- **Chrome Extensions API**, Manifest V3
- **Vanilla JavaScript** - service worker + popup logic
- **chrome.storage.session** - passing results between the background script and the popup
