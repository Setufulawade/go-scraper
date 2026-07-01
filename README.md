# GO Scrapper Engine

A concurrent web scraping system written in Go that efficiently distributes scraping work across a pool of workers, evades anti-bot defenses through proxy rotation, and persists structured results to a database.

## Features

- **Concurrent Scraping**: Uses Go goroutines and channels for high throughput
- **Worker Pool**: Configurable number of workers to balance throughput against rate limits
- **Proxy Rotation**: Built-in proxy rotator to evade rate-limiting and anti-bot measures
- **Local Caching**: Thread-safe in-memory cache to minimize redundant network calls
- **Anti-Bot Evasion**: Integration with anti-bot API for CAPTCHA solving and browser fingerprint spoofing
- **Structured Output**: Data parsing and normalization with database persistence

## Architecture

The system consists of two main subsystems:

1. **Job Pipeline**: Target URL intake → job queue → worker pool → result queue → parser → database
2. **Network Support Layer**: Thread-safe cache, outbound proxy rotator, and anti-bot API

See the [Design Documentation](Design_Docs/High_level_Design.md) for detailed architecture and component breakdown.

## Requirements

- Go 1.26+

## Running

`TODO`
## Project Structure

`TODO`