---
title: Watermarking PDFs in Ruby and Rails
author: philip
date: 2025-04-12 16:22:00 +0800
categories: [Blogging, Tutorial]
tags: [Rails, Ruby, Vector, Rasterization, MiniMagick, ImageMagick, combine_pdf, PDF, watermark]
render_with_liquid: false
---



A project I had been developing required that when users of the web-app open a PDF, their name is automatically watermarked with the standard large, partially transparent letters across every page. This was my first time implementing this feature so I first explored using the popular image processing library, *Image Magick*. Since the web-app is built with Rails, I did some experimenting using *MiniMagick*, a Ruby wrapper for the Image Magick library.

### MiniMagick

```ruby
# watermark_pdf.rb
require "mini_magick"

pdf = MiniMagick::Image.open('test_pdf.pdf')
watermark_text = "Johann Sebastian Bach"

pdf.combine_options do |mogrify|
  mogrify.gravity 'center'
  mogrify.pointsize '100'
  mogrify.draw "rotate -45 text 0,0 '#{watermark_text}'"
  mogrify.fill "rgba(174,180,181,0.4)"
  mogrify.density '200'
  mogrify.write 'output/test-output.pdf'
end
```

Here we open the test_pdf file using the `MiniMagick::Image.open` class method which makes a copy of the PDF. Then we assign the text we want to be watermarked to the `watermark_text` variable.

We call the `combine_options` method on the pdf which allows us to use Image Magick's `mogrify` program to modify the image, implicitly translating the code through `MiniMagick::Command`. 

We can now use the various options to modify the image:

- `.gravity 'center'` sets the origin point to the center of the PDF for operations like `draw`.

- `.pointsize '100'` controls and sets the size of the font text, where 1 point is 1/72 of an inch.  The large the pointsize, the larger the font size.

- `.draw "rotate -45 text 0,0 '#{watermark_text}'"`  adds the text element to the PDF. 
  - `rotate -45` uses the text gravity primitive to rotate the text to show from the bottom left to the top right.

  - The `text 0,0 '#{watermark_text}'` option sets the origin of the text, which we set to the center of the page using `gravity 'center'` earlier, and then specify the `watermark_text` to be added via string interpolation.
- `.fill "rgba(174,180,181,0.4)"` sets the grey color of the text. By using `rgba`, the first three arguments make up the standard RGB model, and last one is an alpha channel which lets us control the opacity of the color. This last field lets you control the opacity of the text using a range between `0` and `1`.
- `.density '200'` sets the DPI of the outputted PDF file. The higher the density, the higher the image quality. 
- `.write 'output/test-output.pdf'` is what outputs the modified PDF to the destination and filename specified.

Now that we have a functioning Ruby script that watermarks PDFs, we can examine the performance.

### Performance: Vector vs Bitmap

When comparing the file size of the original PDF with the output PDF, I realized there was a fundamental flaw with using ImageMagick to modify PDFs. The test PDF in my case has a file size of 222 KBs. After watermarking the PDF using Minimagick with a density of 200, the output is 7.7 MBs. That's ~35x the file size! The reason for this is because of rasterization. 

Rasterization converts vector graphics, which describe the image using mathematical equations to define points, lines, and shapes, to a pixel-based, *raster*, format.  Because vector graphics use math to portray the image, they can be recreated at any size without losing resolution or changing the quality. 

PDFs can contain a mixture of vector and raster elements. So how can we check the type of element in a PDF? One way is to zoom in closely to the elements. If they pixelate, then it's likely a raster (bitmap) type, if the quality stays the same, then it's probably a vector.

Zooming in on my test PDF shows that it is made up of vector graphics. So why does the file size increase so dramatically after using the ImageMagick library? The reason is because ImageMagick is a raster processor, meaning it rasterizes the image (or in our case the PDF), which changes it to a bitmap from the original vector format. This is the reason the file size increases, and because the PDFs the project uses are music scores containing notes and complex non-text elements, this likely augments the file size even more.

Implementing this in the Rails app locally didn't really pose an issue on performance because the local environment is using the resources of my computer, but when deploying to the web, poor performance rears its ugly head.

### MiniMagick in Rails

After implementing the Minimagick watermarker script in Rails, there were two obvious problems: 

1.  The CPU and memory of the cloud platform the app is deployed on is much more limited than the local environment, so resources must be used as efficiently as possible to prevent issues or memory crashes.
2. Increasing the PDF file size over tenfold is not acceptable because it means transferring that much more info over the network, which depending on the connection or area can be quite limited, and it means storing significantly larger file sizes after download which is turns the watermarker into more of a defect than a feature.

I think we're gonna need a different solution.

After some digging, I came across a wonderful Ruby library called, [combine_pdf](https://github.com/boazsegev/combine_pdf/tree/master). 

### CombinePDF

CombinePDF is a pure Ruby gem that can parse, merge, or watermark PDFs. It performs significantly better than the previous minimagick method, watermarking the PDF quickly and without altering the file size.

Here's how I implemented it in my Rails project:

```ruby
#services/watermark.rb
class Watermark
  def initialize(cue, name)
    @cue = cue
    @name = name
  end

  def stamp
    pdf = CombinePDF.load @cue.score.download.path
    page = pdf.pages[0]
    mediabox = page[:MediaBox]
    watermark_page = CombinePDF.create_page mediabox # make title page same size as first page
    width, height = mediabox[2..3]
    watermark_page.textbox @name, opacity: 0.3, y: 300, x: -(width/2), width: width * 1.3
    watermark_page[:Rotate] = -45
    watermark_page.fix_rotation
    pdf.stamp_pages(watermark_page)

    pdf
  end
end
```

I create a `watermark.rb` service object, which contains the `Watermark` class.

When the class is initialized using `Watermark.new(@cue, name_to_stamp)`, the cue active record object the word to stamp, in this case the user's name, are passed in and assigned to instance variables.

We then call the `stamp` method which similarly to ImageMagic, has several steps to set up the properties of the watermark:

- `pdf = CombinePDF.load @cue.score.download.path` uses CombinePDF's `load` method to load the PDF to watermark.

- `pdf.pages[0]` gets the first page of the PDF and assigns it to `page`.

- `page[:MediaBox]` returns an array of four values giving the physical dimensions of the PDF: `[x1, y1, x2, y2]`

  - `x1` is the x-coordinate of the lower-left corner.

  - `y1` is the y-coordinate of the lower-left corner.

  - `x2` is the x-coordinate of the upper-right corner.

  - `y2` is the y-coordinate of the upper-right corner.

- `CombinePDF.create_page mediabox` creates an empty page with the same dimensions as the original PDF.

- `width, height = mediabox[2..3]` destructures and assigns the width and the height.

- `watermark_page.textbox @name, opacity: 0.3, y: 300, x: -(width/2), width: width * 1.3`  creates a textbox with an opacity of `0.1` . Opacity ranges between 0 and 1 and must be written with a `0.` to be valid. the `x` and `y` coordinates can be set based on the page size, they may require tinkering depending on the page and watermark size. The `width` sets the width (size) of the watermark itself, and may also require some tinkering.

- `watermark_page[:Rotate] = -45` rotates the user's name to the typical watermark orientation and then applies it using `watermark_page.fix_rotation`.

- Now that we have our watermark page to be stamped, it's time to stamp it! We use `pdf.stamp_pages(watermark_page)` to apply the watermark on every page of the PDF.

- We then return the stamped `pdf` object. If we wanted to save it to a specific location we could use `pdf.save "NameOfPDF.pdf"` to save a PDF file.

Once the watermarked PDF is returned, in Rails I assign it a local variable and use the `send_data` method to send the file to the browser:

```ruby
  def download
      name_to_stamp = current_account.profile.name
      stamped_pdf = Watermark.new(@cue, name_to_stamp).stamp
      send_data(stamped_pdf.to_pdf, filename: @cue.score.metadata["filename"], type: 		"application/pdf", disposition: "inline")
  end
```



Hopefully this helps explain the watermarking feature of the CombinePDF gem. It's certainly helped me in getting more performant results compared to ImageMagick.

At some point I would love to dive into the CombinePDF gem and see how and why it works better, but that's for a different day!
