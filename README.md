## Troll and Toad Card Price Scraper

A script that scrapes Pokemon card prices from the Troll & Toad online store to appraise a client's collection by updating the Google Sheets spreadsheet of the client. 

## Initialization


```python
import pandas as pd
import numpy as np
import requests
from bs4 import BeautifulSoup as bs
import pygsheets
import time
```


```python
client = pygsheets.authorize(service_account_file="credentials.json")
spreadsheet = client.open("Pokemon Card Value Sheet")
sheet = spreadsheet.worksheet("title", "Sheet1")
```


```python
cards = sheet.get_as_df(empty_value = "null", end = (None, 4))
exempt_cards = ["pokemon-oversized-cards", "prize-pack-series", "psa-graded-pokemon-cards", "professionally-graded-cards", "beckett-graded-pokemon-cards"]
search_urls = []
price_urls = []
card_prices = []
```

## Formulate the search URL
Takes the name and set number of the Pokemon cards present within the client's spreadsheet to construct search URL's for the Troll & Toad website. 


```python
for index in cards.index:
    card_query = ""
    card_words = cards.Card[index].split()
    for word in card_words:
        card_query += "{}+".format(word)
    if "/" in cards.Set_Number[index]:
        card_number, set_number = cards.Set_Number[index].split("/")
        card_query += "{}%2F{}".format(card_number, set_number)
    else:
        card_query += "{}".format(cards.Set_Number[index])
    search_url = "https://www.trollandtoad.com/category.php?selected-cat=0&search-words={}".format(card_query)
    search_urls.append(search_url)
```

Conducts GET requests for each of the search URL's and parses the metadata to construct product price URL's. 


```python
for url in search_urls:
    time.sleep(5)
    search_page = requests.get(url)
    search_parsed_page = bs(search_page.text, "html.parser")
    
    if search_parsed_page.find(class_ = "font-italic font-smaller").text == "0 Products Found":
        url_words = url.split("+")
        url_set_num = url_words[-1]
        if "%2F" in url_set_num:
            card_num, set_num = url_set_num.split("%2F")
            new_card_num = ""
            new_set_num = ""
            is_activated = False
            for char in card_num:
                if char.isalpha():
                    new_card_num += char
                elif char == "0":
                    if is_activated:
                        new_card_num += char
                    else:
                        pass
                else:
                    is_activated = True
                    new_card_num += char
            is_activated = False
            for char in set_num:
                if char.isalpha():
                    new_set_num += char
                elif char == "0":
                    if is_activated:
                        new_set_num += char
                    else:
                        pass
                else:
                    is_activated = True
                    new_set_num += char
            new_id = new_card_num + "%2F" + new_set_num
            new_url = url.replace(url[url.rindex("+") + 1:], new_id)
        else:
            card_set_num = ""
            is_activated = False
            for char in url_set_num:
                if char.isalpha():
                    card_set_num += char
                elif char == "0":
                    if is_activated:
                        card_set_num += char
                    else:
                        pass
                else:
                    is_activated = True
                    card_set_num += char
            new_url = url.replace(url[url.rindex("+") + 1:], card_set_num)

        search_page = requests.get(new_url)
        search_parsed_page = bs(search_page.text, "html.parser")

    card_options = search_parsed_page.find_all("a",  class_ = "card-text")
    for option in card_options:
        is_exempt = False
        option_string = option["href"].replace("/", " ")
        for exempt in exempt_cards:
            if exempt in option_string:
                is_exempt = True
        if is_exempt:
            continue
        else:
            price_url = "https://www.trollandtoad.com{}".format(option["href"])
            price_urls.append(price_url)
            break
```

Iterates through each price GET request to find the sale price of the Pokemon card. Price appraisals are cut by 30% to bring sale price closer to rival market prices from TCGPlayer. The client spreadsheet is then updated with current prices for each card as well as a total collection evaluation. 


```python
for url in price_urls:
    time.sleep(5)
    price_page = requests.get(url)
    price_parsed_page = bs(price_page.text, "html.parser")
    card_prices.append(price_parsed_page.find(id = "sale-price").text)
```


```python
card_prices = list(map(float, card_prices))
depreciation = [x * 0.3 for x in card_prices]
card_prices = np.subtract(card_prices, depreciation)
card_prices = np.around(card_prices, 2)
card_prices = list(map(str, card_prices))
for i in range(len(card_prices)):
    card_prices[i] = "$" + card_prices[i]
```


```python
cards.Price = card_prices
sheet.set_dataframe(cards, (1, 1))
```
