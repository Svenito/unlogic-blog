---
extends: default.liquid
title: Random album generator in python
date: 2012-11-06T21:03:50Z
tags:
---

You may have heard about the "random music album" thing. Basically it goes like this:

- The album cover is the 4th image from a random page of Flickr's interesting pics
- The band name is the title of a random Wikipedia article
- The album name comes from the last 3 or 5 words of a famous quote

This is all well and good, but isn't getting all this data manually, and then making the album cover a bit tedious? Sure it is, so let's see how we can do this in python

Here are some example images generated with this script (click for the full size picture):

<img src="/images/album2.jpg" alt="image2" width="400" />

<img src="/images/album3.jpg" alt="image3" width="400" />

<img src="/images/album4.jpg" alt="image4" width="400" />

Let's cover our dependencies first. You will need:

- python 2.6 (or similar)
- PIL ([python Image Library](https://www.pythonware.com/products/pil/)))
- A [Flickr API key](https://www.flickr.com/services/apps/create/apply/)
- [wget](https://www.gnu.org/software/wget/)

Not much to ask for is it? So once you've made sure you have all that, let's start by getting the album cover. This is handled by the [interestingness](https://www.flickr.com/services/api/flickr.interestingness.getList.html) part of the API and is a very simple call that will return an XML structure of the photos on that page. Once we have a response we parse the XML and get the elements we need to construct the photo URL. It's a short function and here it is:

```python
def getAlbumImage():
    page = random.randint(1,80)
    url = "https://api.flickr.com/services/rest/?method=flickr.interestingness.getList&api_key=YOURAPIKEYHERE&per_page=6&page=%d&format=rest" % (page)
    dom = minidom.parse(urllib.urlopen(url))

    elem = dom.getElementsByTagName('photo')[4]
    farm_id = elem.getAttributeNode('farm').nodeValue
    server_id = elem.getAttributeNode('server').nodeValue
    the_id = elem.getAttributeNode('id').nodeValue
    secret = elem.getAttributeNode('secret').nodeValue

    photo_url = "https://farm%s.staticflickr.com/%s/%s_%s_b.jpg" % (farm_id, server_id, the_id, secret)
    target_photo = 'band.jpg'
    subprocess.call(["/usr/bin/wget", "-O", target_photo, photo_url])
```

First we generate a random number which will be the page number we use in constructing the API URL. Once constructed we use `minidom` to parse it and start extracting our data. If you paste the URL into your browser (with your valid API key) you can see the response format. Now that we have all the data we need, we construct our image URL (the format for this is in the docs) and save it to `band.jpg` using `wget`. You can of course use something else, but this is just easy here.

Right, onto getting our band name. This is the random Wikipedia article. Luckily Wikipedia has an API also, and doesn't require an API key for this purpose. This is an even shorter function:

```python
def getBandName():
    random_wiki_url = "https://en.wikipedia.org/w/api.php?format=xml&action=query&list=random&rnnamespace=0&rnlimit=1"
    dom = minidom.parse(urllib.urlopen(random_wiki_url))

    for line in dom.getElementsByTagName('page'):
        return line.getAttributeNode('title').nodeValue
```

As above we create the URL, parse the output with minidom and fetch our page title. Done.

Album title is a little trickier. I couldn't find a decent quote page that offered a free, easy to use API, so I decided to be a little more hacky and just parse the HTML itself. Hey, it works, don't judge me. We need a helper class for this called MyHTMLParser that derives from python's [HTMLParser](https://docs.python.org/2/library/htmlparser.html?highlight=htmlparser#HTMLParser) class.

```python
class MyHTMLParser(HTMLParser):
    def __init__(self):
        HTMLParser.__init__(self)
        self.get_data = False;
        self.quotes = []

    def handle_starttag(self, tag, attrs):
        if tag == "dt":
            if attrs[0][0] == 'class' and attrs[0][1] == 'quote':
                self.get_data = True

    def handle_endtag(self, data):
        pass

    def handle_data(self, data):
        if self.get_data:
            self.quotes.append(data)
            self.get_data = False

def getAlbumTitle():
    random_quote_url = "https://www.quotationspage.com/random.php3"
    page = urllib.urlopen(random_quote_url).read()
    parser = MyHTMLParser()
    parser.feed(page)

    num_quotes = len(parser.quotes)
    quote = parser.quotes[random.randint(0, num_quotes)].rstrip('.')

    last_set = random.randint(3,5)
    words = quote.split()

    if last_set > len(words):
        last_set = len(words)

    return (" ").join(words[-last_set:])
```

The class here is used to parse the HTML from [<https://www.quotationspage.com/random.php3>](https://www.quotationspage.com/random.php3), specifically the tag that starts with quote. Once we have that we start capturing the data between that tag and store it in an array. Our getAlbumTitle function will use this data to select a random quote and then get the last 3 or 5 words from it and join them with spaces before returning that new string.

So now we have the data that we need, we just need to wrap it all up and generate our final image using `PIL`. Surprise, surprise, this isn't a big deal either.

```python
def main():
    band_name = getBandName()
    album_title = getAlbumTitle()
    cover = getAlbumImage()

    from PIL import ImageFont
    from PIL import Image
    from PIL import ImageDraw

    fnt = ImageFont.truetype("/usr/share/fonts/dejavu/DejaVuSans.ttf",25)
    lineWidth = 20
    image = Image.open("band.jpg")
    imagebg = Image.new('RGBA', image.size, "#000000") # make an entirely black image
    mask = Image.new('L',image.size,"#000000")       # make a mask that masks out all
    draw = ImageDraw.Draw(image)                     # setup to draw on the main image
    drawmask = ImageDraw.Draw(mask)                # setup to draw on the mask
    drawmask.line((0, lineWidth, image.size[0],lineWidth),
                  fill="#999999", width=100)        # draw a line on the mask to allow some bg through
    image.paste(imagebg, mask=mask)                    # put the (somewhat) transparent bg on the main
    draw.text((10,0), band_name, font=fnt, fill="#ffffff")      # add some text to the main
    draw.text((10,40), album_title, font=fnt, fill="#ffffff")      # add some text to the main
    del draw
    image.save("out.jpg","JPEG",quality=100)
```

Let's go over what's happening here. You're welcome to clean it up as an exercise if you wish or think some values (like filenames) etc need configuring. Firstly we call the previously defined functions to fetch our album data and then we start the drawing. I use the `DejaVuSans.ttf` font for this example, but you can use any font you have, or even use different fonts for the title and band name, to make your cover look a bit more pleasing. Once the image we saved from Flickr is open, we start writing our title and band name on the album cover, and save out the result as a `JPEG`. The code here is commented so I won't go over the details here.

And that's all there is to it. If you want the the script as a whole file, you can [get it from this gist](https://gist.github.com/4025200)
