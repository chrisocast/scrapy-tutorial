Scrapy-tutorial-liquor
======================

_**UPDATE:**_ This tutorial is very old at this point and is most likely out of date with the Scrapy library. Also in 2012, Washington State began allowing private-sector businesses to sell liquor and closed all state-owned stores. The Liquor Control Board web site I used in this example was closed along with them. If I get time to post a new Scrapy tutorial, I'll post an update here as well. 

This tutorial will walk you through a web-scraping project from scratch using [Scrapy](http://scrapy.org/), a Python scraping framework. By the end, you'll be able to:

*   Create a spider from scratch using both GET and POST requests
*   Handle the responses via Items and Pipelines
*   Parse your data into multiple files (in this case - CSV)
*   Use XPath selectors to find specific elements within a HTML document

## Planning the data format

In Washington State, the [Liquor Control Board site](http://liq.wa.gov/pricebook/pricebookmenu1.asp) allows users to search all of the state-controlled liquor stores for specific brands, liquor types, prices, amounts, etc. This is the site we'll use for our project. Sounds like some interesting data, no? Before we start writing any code, let's think about what data we'll gather and how to save it. For our purposes we'll focus on the '[Product Availability](http://liq.wa.gov/HomepageServices/brandsearch.asp)' page of the site. This page has a list of brand categories in a dropdown list (Brandy, Vodka, Rum, etc.). When you select a category, the site loads a brands page (Smirnoff, Absolute, Grey Goose, etc). From there, you can click one of the 'Find a store' buttons to see where the product is in stock and how many bottles are left. These three pages will be our focus of our project. We'll break the data into three CSV files - brandCategoryTable.csv, brandsTable.csv, and inStockTable.csv. Here's the format for each file: **brandCategoryTable.csv** _brandCategoryId, brandCategoryName_ **brandsTable.csv** _brandCategoryId, brandId, brandName, totalPrice, specialNote, size_ **storeStockTable.csv** _brandId, stateStoreNumber, amountInStock_

> **NOTE:** This project may seem as if there is a database in its future, but that's not part of this tutorial, so the data format is not a big focus. For now, we're just focusing on scraping.

## Creating a Scrapy project

Open a console and go to the location where you want to create your new project files. Then type: 
````python
scrapy startproject waliquor
````

This creates the basic structure for your project in a folder called "waliquor". Before it will do anything, we need to define the spider, the items and the pipelines.

## Building the spider

> [Spiders](http://doc.scrapy.org/en/0.24/topics/spiders.html) are user-written classes used to scrape information you're after. They return Items.

Our spider will have three major steps:

1.  Request the [brand search page](http://liq.wa.gov/homepageServices/brandsearch.asp) containing the list of brand categories. Once we've received the response containing all of the page data, write the brand category data to CSV and then...
2.  Parse the data for each brand page and create a new request for each brand page. Once we receive the response for a brand page, write the brand data to CSV and then...
3.  Parse the data for each store page that has this brand in stock and write the store data to CSV.

Create a new file in your /waliquor/spiders/ folder. Name it "waliquor_spider.py" and insert the following code: 

````python
from scrapy.spider import BaseSpider
from scrapy.http import FormRequest
from scrapy.http import Request
from scrapy.selector import HtmlXPathSelector
from waliquor.items import BrandCategoryItem

class WaliquorSpider(BaseSpider):
    name = "liq.wa.gov"
    allowed_domains = ["liq.wa.gov"]
    
SPIDER = WaliquorSpider()

    #
    # Setup the initial request to begin the spidering process
    #
    def start_requests(self):
        brandCategoriesRequest = Request("http://liq.wa.gov/homepageServices/brandsearch.asp",
                                  callback=self.parseBrandCategories)
        
        return  [brandCategoriesRequest] 
````

[Start_request()](http://doc.scrapy.org/en/0.24/topics/spiders.html?highlight=start%20request#scrapy.spider.Spider.start_requests) is the method called by Scrapy when the spider is opened for scraping when no particular URLs are specified with a default start_urls list. Once the request is made and the server returns a response, the callback function (self.parseBrandCategories()) takes over. Add this to your spider below the start_requests() function: 

````python
    #
    # Scrape, parse all the brand categories into a list
    # Create a request for every brand category
    #
    def parseBrandCategories(self, response):
        hxs = HtmlXPathSelector(response)
        
        # Gather all brand categories into list
        brandCategoryList = hxs.select('//form/div/center/table/tr/td/select/option[position()>1]/text()').extract()
        
        for i in range(len(brandCategoryList)):
            
            # Generate new request for each brand category's page
            yield FormRequest("http://liq.wa.gov/homepageServices/brandpicklist.asp",
                        method='POST',                          
                        formdata={'BrandName':'','CatBrand':brandCategoryList[i],'submit1':'Find+Product'},
                        callback=self.parseBrandPage,
                        meta={'brandCategoryId':i,'brandCategoryName':brandCategoryList[i]})
            
            # Create items for the brand category pipeline
            item = BrandCategoryItem()
            item['brandCategoryId'] = str(i)
            item['brandCategoryName'] = brandCategoryList[i]
            yield item
````

Let's review what's happening: The function receives the [response](http://doc.scrapy.org/en/0.24/topics/request-response.html?highlight=response#scrapy.http.Response) object from the initial request and creates a [HtmlXPathSelector](http://doc.scrapy.org/en/0.24/topics/selectors.html?highlight=selectors) called **"hxs"**. HtmlXPathSelector gives us a way to look through the HTML and find the specific elements we're after. But before we can write an XPath statement, we have to look at the HTML by hand and figure out the structure. This is where a plugin like Firebug can come in very handy. Remember, in this step we're after the list of brand categories. By looking at the page markup in Firebug, I can see that the brand categories are in a form, then a div, then a center tag, then a table, etc, etc. Eventually, you get to the actual options, where I've used the "[position()>1]" property to avoid grabbing the first item: "(Please select...)". Next is a text() selector to grab only the text, not the tags around it.

````python
brandCategoryList = hxs.select('//form/div/center/table/tr/td/select/option[position()>1]/text()').extract()
````

> **Protip:** "If you are viewing page markup in Firefox, remember that the browser adds its own < tbody > tags to tables, but Scrapy will not receive those in the response. Including < tbody > in your Scrapy XPath will always break it.

Now that we have the full list of brand categories, we must step through them and create a new request for each - to load the brand pages. Unlike the previous request, these pages will require a POST request. ![](/wp-content/uploads/2010/05/scrapy_tutorial_firebug.jpg "scrapy_tutorial_firebug")Again, Firefox/Firebug is a great tool. In this case, you can use it to load a page and then look through the response the browser is sending. Using that information, we added a form request. You may notice another callback function there - we'll get to that soon. For now, don't worry about it. The last part of the parseBrandCategories() function in your spider handles sending the brand category data to its corresponding item. The code creates a BrandCategoryItem item, then the two pieces of data we're after - brandCategoryId and brandCategoryName - are populated. Remember these fields from our "brandCategoryTable.csv" file? 

````python
# Create items for the brand category pipeline
item = BrandCategoryItem()
item['brandCategoryId'] = str(i)
item['brandCategoryName'] = brandCategoryList[i]
yield item
````

## Defining your items

> [Items](http://doc.scrapy.org/en/0.24/topics/items.html?highlight=items) are containers that will be loaded with the scraped data.

Open the items.py file in your project folder and overwrite everything with the following code:
````python
from scrapy.item import Item, Field

class BrandCategoryItem(Item):
    brandCategoryId = Field()
    brandCategoryName = Field()
````

That's it! An item is a simple container for the values you just parsed via the spider. The last part is outputting your data to a CSV file. For this, we need a pipeline.

## Creating pipelines

> [Pipelines](http://doc.scrapy.org/en/0.24/topics/item-pipeline.html?highlight=pipeline) receive the scraped data from the items and stores it in the way you specify.

In your project folder, there is a file called pipelines.py. Open the file and overwrite everything with the following: 

````python
import csv
import items

class WaliquorPipeline(object):
    
    def __init__(self):
        self.brandCategoryCsv = csv.writer(open('brandCategoryTable.csv', 'wb'))
        self.brandCategoryCsv.writerow(['brandCategoryId', 'brandCategoryName'])

    def process_item(self, item, spider):           
            self.brandCategoryCsv.writerow([item['brandCategoryId'], item['brandCategoryName'].title()])
            return item
````

The pipeline is using the built-in CSV abilities of Python to open a new file (or an existing one if it's already been created) and then write a header row to the top. Next, when all the BrandCategoryItem instances are created by your spider, they will automatically be passed in to this pipeline where the [process_item()](http://doc.scrapy.org/en/0.24/topics/item-pipeline.html?highlight=process#process_item) class will add a row to "brandCategoryTable.csv". You've now gone from an initial request to outputting CSV file! We also completed the first step of our project - getting the brand category data. Don't worry, the next two steps are very similar, so we'll fly through them.

## Rinse and repeat for the brand pages

Back in your spider class, add the following code at the top of the file: `from waliquor.items import BrandItem` Also, add this beneath the parseBrandCategories() function: 

````python
#
# Parse each individual brandCategory page
# brandCode (unique key), brandCategoryId, brandName, brandCode, totalPrice, specialNote, size, proof
#
def parseBrandPage(self, response):
    
    hxs = HtmlXPathSelector(response)
    brandRows = hxs.select('//table[@class=\'tbl\']/tr[position()>1]')
    
    for brandRow in brandRows:
    
        brandId = brandRow.select('td[position()=2]/strong/text()').extract()
    
        # Generate new request for this brand category's page
        yield FormRequest("http://liq.wa.gov/homepageServices/find_store.asp",
            method='POST', formdata={'brandCode':brandId,'CityName':'','CountyName':'', 'StoreNo':''},
            callback=self.parseBrandInStockPage,
            meta={'brandId':brandId})

        item = BrandItem()
        item['brandId'] = brandId
        item['brandCategoryId'] = response.request.meta['brandCategoryId']
        item['brandName'] = brandRow.select('td[position()=1]/strong/text()').extract()
        
        item['totalPrice'] = brandRow.select('td[position()=5]/strong/text()').extract()
        item['totalPrice'][0] = item['totalPrice'][0].replace('$','')
        item['totalPrice'][0] = item['totalPrice'][0].replace(',','')
        
        item['specialNote'] = brandRow.select('td[position()=7]/text()').extract()
        item['size'] = brandRow.select('td[position()=8]/text()').extract()
        item['proof'] = brandRow.select('td[position()=10]/text()').extract()
        
        yield item
````

When the parseBrandCategories() function builds its requests for each brand page, this is the callback function that will receive the response. The process is almost identical to the previous function. It receives the response and creates a HtmlXPathSelector, but this time, instead of selecting strings we're selecting table rows. As the for loop steps through the list of rows, the various properties we want are again selected from the table rows using XPath and sent into the item - this time a BrandItem.

> Want to learn more about XPath and selectors? Of course you do! Here's a [nice tutorial](https://developer.mozilla.org/en/XPath) from Mozilla.

Next, open your items.py file and add the following below the BrandCategoryItem:

````python
class BrandItem(Item):
    brandId = Field()
    brandCategoryId = Field()
    brandName = Field()
    totalPrice = Field()
    specialNote = Field()
    size = Field()
    proof = Field()
````

Now, we need to wire up the pipeline to take the item and store the data. But wait! We're going to hit a slight snag here. If you remember, the pipelines.py file uses a function called process_item() to handle each item it receives. But in our case, we have three different items from different functions that need to write to different CSV files. Currently, our pipelines.py class will handle every item the same. Here's how we'll get around that:

````python
import csv
import items

class WaliquorPipeline(object):
    
def __init__(self):
    self.brandCategoryCsv = csv.writer(open('brandCategoryTable.csv', 'wb'))
    self.brandCategoryCsv.writerow(['brandCategoryId', 'brandCategoryName'])
    
    self.brandsCsv = csv.writer(open('brandsTable.csv', 'wb'))
    self.brandsCsv.writerow(['brandCategoryId', 'brandId', 'brandName', 'totalPrice', 'specialNote', 'size', 'proof'])

def process_item(self, item, spider):
            
    if isinstance(item, items.BrandCategoryItem):
        self.brandCategoryCsv.writerow([item['brandCategoryId'], item['brandCategoryName'].title()])
        return item
        
    if isinstance(item, items.BrandItem):
        self.brandsCsv.writerow([item['brandCategoryId'], item['brandId'][0], item['brandName'][0].title(), item['totalPrice'][0], item['specialNote'][0].title(), item['size'][0], item['proof'][0]])  
        return item
````

With this change, the process item function handles the items differently, depending on their type. Now we have lots of data moving through to our CSVs, only one more step - the store page.

## Last step, gather and store data

This process should be starting to make sense now. Let's go back to the beginning - the spider class - one more time. Add the following code at the very beginning: `from waliquor.items import StockItem` Then add this function below the parseBrandPage() function:
````python
#
# Parse each individual brand's store availability page
# brandId ("Brand Code" provided), stateStoreNumber(id), amountInStock
#
def parseBrandInStockPage(self, response):
    
    hxs = HtmlXPathSelector(response)   
    storeRows = hxs.select('//table[@class=\'tbl\']/tr[position()>1]')
    
    items = []
    
    for storeRow in storeRows:
        item = StockItem()
        item['brandId'] = response.request.meta['brandId']
        item['stateStoreNumber'] = storeRow.select('td[position()=1]/text()').extract()
        item['amountInStock'] = storeRow.select('td[position()=5]/font/text()').extract()
        items.append(item)  
        
    return items
````

Next, we'll add our StockItem to the item.py class:
````python
class StockItem(Item):
    brandId = Field()
    stateStoreNumber = Field()
    amountInStock = Field()
````

Lastly, here's what the complete pipelines.py file should look like:
````python
import csv
import items

class WaliquorPipeline(object):
    
    def __init__(self):
        self.brandCategoryCsv = csv.writer(open('brandCategoryTable.csv', 'wb'))
        self.brandCategoryCsv.writerow(['brandCategoryId', 'brandCategoryName'])
        
        self.brandsCsv = csv.writer(open('brandsTable.csv', 'wb'))
        self.brandsCsv.writerow(['brandCategoryId', 'brandId', 'brandName', 'totalPrice', 'specialNote', 'size', 'proof'])
        
        self.storeStockTable = csv.writer(open('storeStockTable.csv', 'wb'))
        self.storeStockTable.writerow(['brandId', 'stateStoreNumber', 'amountInStock'])

    def process_item(self, item, spider):
                
        if isinstance(item, items.BrandCategoryItem):
            self.brandCategoryCsv.writerow([item['brandCategoryId'], item['brandCategoryName'].title()])
            return item
    
        if isinstance(item, items.BrandItem):
        
        # Double check that items in the pipeline exist
        # Otherwise, an item with a an empty list would 
        # be completely skipped over by Scrapy
        
            try:
                item['brandId'][0]
            except:
                item['brandId'].append("")
                
            try:
                item['brandCategoryId']
            except:
                item['brandCategoryId'] = "9999"
                
            try:
                item['brandName'][0]
            except:
                item['brandName'].append("")
                
            try:
                item['totalPrice'][0]
            except:
                item['totalPrice'].append("")
                
            try:
                item['specialNote'][0]
            except:
                item['specialNote'].append("")
                
            try:
                item['size'][0]
            except:
                item['size'].append("")
                
            try:
                item['proof'][0]
            except:
                item['proof'].append("")

            self.brandsCsv.writerow([item['brandCategoryId'], item['brandId'][0], item['brandName'][0].title(), item['totalPrice'][0], item['specialNote'][0].title(), item['size'][0], item['proof'][0]])
            
            return item
            
        
        if isinstance(item, items.StockItem):
            self.storeStockTable.writerow([item['brandId'][0], item['stateStoreNumber'][1], item['amountInStock'][0]])
            return item
````

Besides adding the StockItem in the pipelines class, you'll noticed I also added a stack of try/excepts to the part handling the BrandItems. In some cases, the values in those fields are sometimes empty. These check make sure that the value will be returned as an empty string, not null in those cases.

## Running your spider

If you haven't already, you're probably ready to test your project and scrape some data. To run your scraper, open your console and go to your project folder. Next, type: `scrapy crawl liq.wa.gov` You should begin to see lots of lines whizzing by as Scrapy makes requests, sends items, etc. If things don't work the first time, don't sweat it. Take a close look at any error messages you may be seeing. Python is usually good at pointing you to the file and line number where the error occurred. Also, the link to the working tutorial files is at the beginning of this post. Keep in mind that we're scraping a lot of data from the site and making thousands of requests. Don't be surprised if the spider takes 30-40 minutes to finish. Want to speed up the spider for testing purposes? Go to the parseBrandCategories() function in the spider and change the for loop to read: `for i in range(len(brandCategoryList)-33):` This will limit the brand categories you parse to just the first two in the list.
