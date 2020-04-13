# Arxiv Paper Web Scraping
A note to build a web crawler by using beautifulsoup and selenium. This program crawls arviv to get the information of five certain areas (computer science, mathematics, statistics, electrical engineering and systems science, and quantitative finance). Paper title, subject and link are fetched from the arxiv webpage and stored as a csv file. Jupyter notebook is available as well through this [link](https://github.com/penguinwang96825/arxiv_paper_web_scraping/blob/master/arxiv.ipynb).

## Import packages
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import time
import requests
import bs4
from bs4 import BeautifulSoup
from selenium.common.exceptions import *
from selenium import webdriver
from tqdm.notebook import tqdm
%matplotlib inline
```

## Main Code
### Get total entries
Get the total entries of each categories in each year.

```python
def get_arxiv_total_entries(year, category):
    origin_url = r"https://arxiv.org/list/{}/{}?skip=0".format(category, int(str(year)[-2:]))
    headers = {
        'user-agent': 'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36'}
    r = requests.get(origin_url, headers=headers)
    soup = BeautifulSoup(r.text, "html.parser")
    text = soup.find(name="div", attrs={"id": "dlpage"})
    total_entries = text.find(name="small")
    total_entries = int(total_entries.contents[0].split()[3])
    return total_entries
```

### Get the arxiv paper list
It can show at most 2000 papers in one page. Therefore, I have to make a "for" loop to change to the next page until all papers of each category have been appended into a lost.

```python
def get_arxiv_paper_list(total_entries, year, category):
    y = int(str(year)[-2:])
    driver_path = r"C:\Users\YangWang\Desktop\machineLearning\indiaNewsClassification\chromedriver.exe"
    driver = webdriver.Chrome(driver_path)
    time.sleep(2)
    paper_list = []

    for num in range(int(total_entries/2000)+1):
        url = r"https://arxiv.org/list/{}/{}?skip={}&show=2000".format(category, y, num*2000)
        driver.get(url)
        time.sleep(5)
        soup = BeautifulSoup(driver.page_source, "html.parser")
        content = soup.find(name="div", attrs={"id": "content"})
        dl = content.find(name="dl")

        dd = dl.find_all(name="dd")
        dt = dl.find_all(name="dt")
        for e1, e2 in zip(dd, dt):
            if e1.find(name="div", attrs={"class": "list-title mathjax"}) is not None:
                title = e1.find(name="div", attrs={"class": "list-title mathjax"}).text
            else:
                title = None
            if e1.find(name="span", attrs={"class": "primary-subject"}) is not None:
                subject = e1.find(name="span", attrs={"class": "primary-subject"}).string
            else:
                subject = None
            if e2.find(name="span", attrs={"class": "list-identifier"}) is not None:
                link = e2.find(name="span", attrs={"class": "list-identifier"}).find(name="a").get("href")
            else:
                link = None
            year = str(year)
            category = str(category)
            paper_list.append([title, subject, link, year, category])

    driver.quit()
    
    return paper_list
```

### Dataframe preprocessing
1. Input the list.
2. Convert the list into a dataframe.
3. Do some dataframe preprocessing.

```python
def dataframe_preprocessing(paper_list):
    df = pd.DataFrame(paper_list)
    df.columns = ["title", "subject", "link", "year", "category"]
    df.reset_index(drop=True)

    df["title"] = df["title"].map(lambda x: x.split(":")[1])
    df["title"] = df["title"].map(lambda x: x.split("\n")[0])
    df["link"] = df["link"].map(lambda x: "https://arxiv.org" + x if x is not None else None)
    df["year"] = df["year"].map(int)
    df["category"] = df["category"].map(str)
    
    return df
```

### Extract certain category of the research area
Conduct a `get_arxiv_dataframe` function to get the certain category of research area. (computer science, mathematics, statistics, electrical engineering and systems science, and quantitative finance)

```python
def get_arxiv_dataframe(category, start_year=2010, end_year=2020):
    """
    category:
        "cs": computer science
        "math": mathematics
        "stat": statistics
        "eess": electrical engineering and systems science
        "q-fin": quantitative finance
    """
    final_df = pd.DataFrame()
    for year in tqdm(range(start_year, end_year+1)):
        total_entries = get_arxiv_total_entries(year, category)
        paper_list = get_arxiv_paper_list(total_entries, year, category)
        df = dataframe_preprocessing(paper_list)
        final_df = pd.concat([final_df, df])
        
        final_df = final_df.drop_duplicates()
        final_df.dropna(inplace=True)
    return final_df
```

### Take a look at the dataframe
```python
df = get_arxiv_dataframe(category = "cs, start_year=2010, end_year=2020)
df.sample(5)
```
|  | title | subject | link | year |
| --- | --- | --- | --- | --- |
| 32916 | Periodic particle arrangements using standing... | Numerical Analysis (math.NA) | https://arxiv.org/abs/1908.08664 | 2019 |
| 17914 | Analytic connectivity of kk-uniform hypergraphs | Combinatorics (math.CO) | https://arxiv.org/abs/1507.02763 | 2015 |
| 14420 | Kernel and wavelet density estimators on mani... | Probability (math.PR) | https://arxiv.org/abs/1805.04682 | 2018 |
| 1743 | Analysis of Evidence Using Formal Event Recon... | Formal Languages and Automata Theory (cs.FL) | https://arxiv.org/abs/1302.2308 | 2013 |
| 12006 | Sequential Bayesian optimal experimental desi... | Methodology (stat.ME) | https://arxiv.org/abs/1604.08320 | 2016 |
