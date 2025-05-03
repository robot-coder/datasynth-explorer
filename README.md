# README.md

# Web Data Collection and Visualization Agent

This project develops an intelligent agent that integrates custom Python functions with MCP server interactions to automate web data collection, summarization, and visualization. The agent fetches data from web sources, processes and summarizes the information, and generates interactive reports with visualizations.

## Features

- Connects to MCP server for data management
- Automates web scraping using Playwright and BeautifulSoup
- Summarizes collected data
- Visualizes data insights with Matplotlib
- Generates interactive reports

## Requirements

Ensure you have Python 3.8+ installed. Install the required libraries:

```bash
pip install -r requirements.txt
```

## Files

- `main.py`: Main script orchestrating the agent's workflow
- `requirements.txt`: List of dependencies
- `README.md`: This documentation

## Usage

Run the main script:

```bash
python main.py
```

## License

This project is provided as-is without any warranty.

---

# main.py

import asyncio
from typing import List, Dict, Any
import logging

from playwright.async_api import async_playwright, Playwright, Browser, Page
from bs4 import BeautifulSoup
import pandas as pd
import matplotlib.pyplot as plt
from llama_index import GPTIndex  # Assuming llama_index provides indexing and summarization
from mcp_server_client_library import MCPClient  # Placeholder for MCP server client library

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

class WebDataAgent:
    """
    An agent to automate web data collection, processing, and report generation.
    """

    def __init__(self, mcp_server_url: str):
        """
        Initialize the agent with MCP server URL.
        """
        self.mcp_client = MCPClient(mcp_server_url)
        self.data_frames: List[pd.DataFrame] = []

    async def fetch_web_page(self, url: str, timeout: int = 30) -> str:
        """
        Fetch the content of a web page asynchronously.
        """
        try:
            async with async_playwright() as p:
                browser: Browser = await p.chromium.launch()
                page: Page = await browser.new_page()
                await page.goto(url, timeout=timeout * 1000)
                content: str = await page.content()
                await browser.close()
                logging.info(f"Fetched content from {url}")
                return content
        except Exception as e:
            logging.error(f"Error fetching {url}: {e}")
            return ""

    def parse_html(self, html_content: str) -> BeautifulSoup:
        """
        Parse HTML content with BeautifulSoup.
        """
        try:
            soup = BeautifulSoup(html_content, 'html.parser')
            return soup
        except Exception as e:
            logging.error(f"Error parsing HTML: {e}")
            return BeautifulSoup("", 'html.parser')

    def extract_data(self, soup: BeautifulSoup) -> pd.DataFrame:
        """
        Extract relevant data from BeautifulSoup object.
        Customize this method based on target data.
        """
        try:
            # Example: Extract all table data
            tables = soup.find_all('table')
            data_frames = []
            for table in tables:
                df = pd.read_html(str(table))[0]
                data_frames.append(df)
            if data_frames:
                combined_df = pd.concat(data_frames, ignore_index=True)
                logging.info("Extracted data into DataFrame")
                return combined_df
            else:
                logging.warning("No tables found in HTML")
                return pd.DataFrame()
        except Exception as e:
            logging.error(f"Error extracting data: {e}")
            return pd.DataFrame()

    def summarize_data(self, data: pd.DataFrame) -> str:
        """
        Summarize data using llama_index or other summarization methods.
        """
        try:
            index = GPTIndex(data)
            summary = index.summarize()
            logging.info("Data summarized successfully")
            return summary
        except Exception as e:
            logging.error(f"Error during summarization: {e}")
            return "Summary unavailable."

    def visualize_data(self, data: pd.DataFrame, output_path: str = "visualization.png") -> None:
        """
        Generate and save visualizations from data.
        """
        try:
            plt.figure(figsize=(10,6))
            if 'Value' in data.columns and 'Category' in data.columns:
                data_grouped = data.groupby('Category')['Value'].sum()
                data_grouped.plot(kind='bar')
                plt.title('Category Value Distribution')
            else:
                data.plot()
                plt.title('Data Visualization')
            plt.savefig(output_path)
            plt.close()
            logging.info(f"Visualization saved to {output_path}")
        except Exception as e:
            logging.error(f"Error generating visualization: {e}")

    def store_data(self, data: pd.DataFrame, key: str) -> None:
        """
        Store data in MCP server.
        """
        try:
            self.mcp_client.store(key, data.to_json())
            logging.info(f"Data stored in MCP server with key: {key}")
        except Exception as e:
            logging.error(f"Error storing data: {e}")

    def generate_report(self, summary: str, visualization_path: str) -> str:
        """
        Generate an HTML report combining summary and visualization.
        """
        report_html = f"""
        <html>
        <head><title>Web Data Report</title></head>
        <body>
            <h1>Data Summary</h1>
            <p>{summary}</p>
            <h2>Visualization</h2>
            <img src="{visualization_path}" alt="Data Visualization"/>
        </body>
        </html>
        """
        report_path = "report.html"
        try:
            with open(report_path, 'w') as f:
                f.write(report_html)
            logging.info(f"Report generated at {report_path}")
            return report_path
        except Exception as e:
            logging.error(f"Error generating report: {e}")
            return ""

    async def run(self, urls: List[str]) -> None:
        """
        Main execution method to run the agent workflow.
        """
        for url in urls:
            html_content = await self.fetch_web_page(url)
            if not html_content:
                continue
            soup = self.parse_html(html_content)
            data = self.extract_data(soup)
            if data.empty:
                continue
            self.data_frames.append(data)
            self.store_data(data, key=f"data_{url}")

        if self.data_frames:
            combined_data = pd.concat(self.data_frames, ignore_index=True)
            summary = self.summarize_data(combined_data)
            self.visualize_data(combined_data)
            visualization_path = "visualization.png"
            report_path = self.generate_report(summary, visualization_path)
            logging.info(f"Process completed. Report available at {report_path}")
        else:
            logging.warning("No data collected from URLs.")

if __name__ == "__main__":
    import sys

    # Example URLs to scrape
    target_urls = [
        "https://example.com/data1",
        "https://example.com/data2"
    ]

    agent = WebDataAgent(mcp_server_url="http://localhost:8000")
    asyncio.run(agent.run(target_urls))

# requirements.txt

llama_index
mcp_server_client_library
playwright
beautifulsoup4
matplotlib
pandas