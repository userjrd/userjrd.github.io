---
title: Changes Between the 2007 and 2011 ESV Text
description: The recent revision to the ESV text in 2025 piqued my interest to see how much the translation has changed since its release in 2001. This post is the first of several and will explore the second revision to the ESV text and how I used local language models to find the differences.
author: userjrd
categories: [Theology, Bible Translation]
tags: [bible, esv, ocr, llm, scraping]
---

<!-- ## Outline
1. Finding the 2011 changes
   - Internet Archive
2. Dealing with Flash
   - FFDec
   - Renaming Files
3. Optical Character Recognition (OCR)
   - img2table
   - qwen2.5-vl-7b
4. Formatting
   - difflib
5. Results -->

## Spot the Difference
ESV has conveniently posted documents highlighting the changes for the 2016[^fn1] and 2025[^fn2] text editions on their website. Unfortunately I couldn't find a similar document for the 2011 edition but I figured at one point it had probably been hosted on their website. After digging around I found an old blog post[^fn3] that linked to the 2011 change file that was no longer hosted. Thankfully someone had captured the link in the Internet Archive and I was able to access the changes.[^fn4]

At this point I was confused because the only file I could download from this was an `*.swf` file which I had never encountered before. After some Googling I discovered it stood for Small Web File and was a defunct Flash format. Thankfully there are programs out there that can handle the format still. I chose an open source project called FFDec (Free Flash Decompliler)[^fn5] that was cross-platform compatible with MacOS, Linux, and Windows. Score!

I'm not sure if I did something wrong but I couldn't get the MacOS version to launch so I installed the Linux version instead and it worked great. I don't know anything about Flash files but I found a folder of images and was able to save it to a folder. So now I was left with 27 pages of images of that I needed to extract the text from. The text changes was displayed in three borderless columns of "Reference", "Changed From", and "Changed To".

## Optical Character Recognition (OCR)
Thankfully OCR isn't that big of a deal and is simple to use (or so I thought). I found a Python project called `img2table` and it seemed like it was perfect.[^fn6] It took my a long time even to get it working which was due to me not including the path to the TesseractOCR on my computer. I got it working but I'm still not entirely sure what it is I did that actually fixed it. The following is the code I ended up with:

```python
from img2table.document import Image
from img2table.ocr import TesseractOCR
import pandas as pd

# load image
img = Image(src='.raw/1.png)
tesseract = TesseractOCR()

# Extract tables with Tesseract
tables = img.extract_tables(ocr=tesseract, 
  implicit_columns=True, 
  implicit_rows=True, 
  borderless_tables=True)

# combine each row
frames = [table.df for table in tables]
full_data_df = pd.concat(frames)
```

While this worked, the output was not very clean or accurate. Often time a single row would appear as multiple rows and it didn't recognize spaces between words always and would often notputspacesbetweenwords. Not ideal and with 27 pages to sort through I didn't know if I could trust the output. So I changed course and looked at some vision language models.

## Vision Language (VL) Models
VL models are like OCR+ that can understand more context of what you are trying to extract. Because the images I want to extract are in a borderless table format I think VL models would be a good bet instead of tinkering with traditional OCR. I used LM Studio to search for VL models and landed on `Qwen2.5-VL-7B`. (I tried `Qwen3-VL-8B` because it was newer but it didn't seem to work as well). 

I gave it the simple prompt: "The attached images have three columns of text for Bible Reference, Changed From, and Changed To. Extract the text from each image and display as a table." After about 40 seconds (running on a 2024 Macbook Pro) the model spit out exactly what I was looking for in a nice Markdown formatted table. The only problem was that the context length of the model only allowed 4-5 images before it ran out of space.

LM Studio has a Python API that I've never used before but I thought this would be a great usecase for it since I would need to do a repetitive action of creating a model, uploading the image, giving a prompt, and storing the result for each of the 27 images. Another thing I needed to get rid of was the commentary the model provided before giving the actual results. 

Large-language models (LLMs) love to talk about what they are doing. To avoid this I came up with the following prompt: ""The attached image has three columns of text for Bible Reference, Changed From, and Changed To. Extract the text from the image and display as a table. I only want the table returned with no commentary". The Python implementation ended up like this:

```python
import lmstudio as lms
import time

basepath = '.raw/'

# extract table from each image
for i in range(1,27+1):
    start_time = time.perf_counter()
    filename = f'{i}.png'
    img_path = f'{basepath}{filename}'
    print(f'Extracting table from {img_path}...', end=' ')

    # create LM Studio model for each page
    with lms.Client() as client:
        image_handle = client.files.prepare_image(img_path)
        model = client.llm.model("qwen/qwen3-vl-8b")
        chat = lms.Chat()
        message = "The attached image has three columns of text for Bible Reference, " \
        "Changed From, and Changed To. Extract the text from the image and display as a table." \
        "I only want the table returned with no commentary"
        chat.add_user_message(message, images=[image_handle])
        prediction = model.respond(chat)

        # print execution time
        finish_time = time.perf_counter()
        total_time = finish_time - start_time
        print(f'Finished in {total_time:.3f} seconds')

        # append table to file
        with open('_tables/_esv2011revisions.md', 'a') as f:
            if i == 1:      # include header
                print(prediction.content, file=f)
            else:           # remove header
                table = prediction.content.split('\n')[2:]
                content = '\n'.join(table)
                print(content, file=f)
```

## Changes


## Sources
[^fn1]: [ESV 2016 Changes](https://www.esv.org/about/2016-updates/)
[^fn2]: [ESV 2025 Changes](https://uploads.crossway.org/excerpt/esv-2025-text-changes.pdf)
[^fn3]: [Blog Post Discussing ESV 2011 Changes](https://hjimkeener.wordpress.com/2012/07/05/making-a-good-translation-better-the-2011-esv-update-and-beyond/)
[^fn4]: [ESV 2011 Changes](https://web.archive.org/web/20111123181617/http://d3p91it5krop8m.cloudfront.net/wp-content/uploads/misc/esv_2011_changes.html)
[^fn5]: [Free Flash Decompiler](https://github.com/jindrapetrik/jpexs-decompiler)
[^fn6]: [img2table Python Library](https://github.com/xavctn/img2table)
