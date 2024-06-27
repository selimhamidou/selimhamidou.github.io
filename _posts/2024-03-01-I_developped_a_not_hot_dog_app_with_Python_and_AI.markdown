---
layout: post
title: "I've created a Not Hot Dog app"
date: 2024-06-27 09:00:00 +0300
categories: jekyll update
---

![Jian Yang - Not Hot Dog](/Images/nothotdog_jian_yang.jpg)

Hey! Remember that scene episode in Silicon Valley where Jian Yang creates an app to identify hot dogs? Personally, I've always been inspired by this scene and by AI in general. So, let's channel our inner Jian Yang for a day and create our first "Not Hot Dog" app using Python and FastAI.

<iframe width="420" height="315" src="https://www.youtube.com/watch?v=vIci3C4JkL0" frameborder="0" allowfullscreen></iframe>

### Our stack for this project
For this project, we will use Kaggle, which is essentially a powerful computer where you can perform complex operations. On Kaggle, you can install all the necessary AI libraries, making it much simpler than using your own computer. The only downside is that if you leave your computer for a while, you may need to reinstall and relaunch everything.
So now, let's begin. Just go to <a href="https://www.kaggle.com/">Kaggle</a>, and register.
And I we said, Python and FastAI will be our best friends for this project.

### Making it real : Let's code now
Once registered, create a new Notebook and add the following line of code:

There, you write : 
{% highlight python %}
pip install fastbook
{% endhighlight %}

This command will install FastAI, which may take a few minutes.

Next, import everything we need for FastAI:

{% highlight python %}
from fastbook import *
from time import sleep
from fastai.vision.all import *
from fastdownload import download_url
{% endhighlight %}

Now, let's dive into the main part:

{% highlight python %}
searches = 'hotdog','sandwich','banana','bus','shoe','bread','panini', 'saucisse'
path = Path('is_hotdog')

for o in searches:
    if (o == 'hotdog'):
        dest = (path/'hotdog')
        dest.mkdir(exist_ok=True, parents=True)
        download_images(dest, urls=search_images_ddg(f'{o}'))
        resize_images(path/'hotdog', max_size=400, dest=path/'hotdog')
        sleep(10)  # Pause between searches to avoid over-loading server
    elif (o != 'hotdog'):
        dest = (path/'nothotdog')
        dest.mkdir(exist_ok=True, parents=True)
        download_images(dest, urls=search_images_ddg(f'{o}'))
        sleep(10)  # Pause between searches to avoid over-loading server
        resize_images(path/'nothotdog', max_size=400, dest=path/'nothotdog')
{% endhighlight %}

We have a tuple of strings, and we will use DuckDuckGo to search for images corresponding to each string. Every image labeled "hotdog" will go into the "hotdog" folder, while images of other items (like "shoe" or "banana") will be placed in the "nothotdog" folder. We also resize the images to ensure they have the same format, which makes it easier for our AI to process them.

Some images might be problematic and need to be deleted:

{% highlight python %}
failed = verify_images(get_image_files(path))
failed.map(Path.unlink)
len(failed)
{% endhighlight %}

Let's check the number of images we have:

{% highlight python %}
import os

_, _, files = next(os.walk("is_hotdog/nothotdog"))
file_count = len(files)
print(file_count)
{% endhighlight %}

![Verifying Images](/Images/nothotdog_verify_images.jpg)

189 hot dog images is quite low for training our model, but it will suffice for our purposes. Why? Because FastAI uses pretrained models, which saves us a lot of time. Without pretrained models, we would need millions of images and hours of training.

Now, let's create our model:

{% highlight python %}
dls = DataBlock(
    blocks=(ImageBlock, CategoryBlock), #We define the input and output(image as an input, and category as an output)
    get_items=get_image_files, #We use the get_image_files method to retrieve the files we just saved. We will use thes files to train our model 
    splitter=RandomSplitter(valid_pct=0.2, seed=42),    #We take 20% of our data, and use it to know if our model is good, or not
    get_y=parent_label, #we get the label of each image from its parent folder
    item_tfms=[Resize(192, method='squish')],   #We transform our images, squish them to get the same format, and make our model more accurate
).dataloaders(path, bs=32) #To process our data, we need to process them with the dataloaders method. There, we set a batch size of 32.
dls.show_batch(max_n=6) #Then, we ask FastAI to show us a sample of 6 images of our model, as you can see on the following picture.
{% endhighlight %}

![Getting a sample of our model](/Images/nothotdog_batch_images.jpg)

And let's train it:

{% highlight python %}
learn = vision_learner(dls, resnet18, metrics=error_rate)
learn.fine_tune(3)
{% endhighlight %}

![Training our model](/Images/nothotdog_training_model.jpg)

Let's test it:

{% highlight python %}
from fastdownload import download_url
urls = search_images_ddg('dentifrice', max_images=1)
urls[0]
dest = 'ishotdog.jpg'
download_url(urls[0], dest, show_progress=False)

im = Image.open(dest)
im.to_thumb(256,256)
{% endhighlight %}


### So, is this image a hot dog ?

{% highlight python %}
is_hotdog,_,probs = learn.predict(PILImage.create('ishotdog.jpg'))
print(f"This is a: {is_hotdog}.")
print(f"Probability it's a hotdog: {probs[0]:.4f}")
{% endhighlight %}

![Getting an image for the test](/Images/nothotdog_getting_image_test.jpg)
<br>
![Verifying if it's a hot dog](/Images/nothotdog_final_result.jpg)

### Sources
<ul>
<li><a href="https://course.fast.ai/Lessons/lesson1.html">A great course to learn FastAI</a></li>
<li><a href="https://www.learnpython.org/">If you need to learn Python</a></li>
<li><a href="https://www.kaggle.com/docs">How to use Kaggle</a></li>
</ul>

Cheers,
Sélim HAMIDOU






