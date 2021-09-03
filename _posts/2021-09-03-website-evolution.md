---
layout: post-no-feature
title: The Evolution of My Personal Website
description: A little piece of personal history.
comments: true
visible: true
date: 2021-09-03
category: articles
---

I just updated my [personal website](https://akshayr.xyz) for the first time in a while, to reflect my dope new job at [schoolhouse.world](https://schoolhouse.world). While doing so, I decided to checkout random commits in the [repository](https://github.com/akshayravikumar/blog) to see how the website has changed over time. This was surprisingly fun, so naturally, I wrote up a script to visualize the changes.

First, I iterated through every commit in the repo, opened the `index.html` page in [Selenium](https://www.selenium.dev/), took a screenshot, and saved it as a PNG. I had to wait three seconds between page loads because my website has a few animations.

```python
import git
import time
import os
import shutil
from selenium import webdriver

REPO_DIR = '/Users/akshay/Documents/random/akshayravikumar.github.io'
SCREENSHOT_DIR_NAME = "screenshots"
FILE_PATH = '/index.html'

repo = git.Repo(REPO_DIR)
driver = webdriver.Chrome()

if os.path.exists(SCREENSHOT_DIR_NAME):
    shutil.rmtree(SCREENSHOT_DIR_NAME)
os.mkdir(SCREENSHOT_DIR_NAME)

index = 0
for commit in repo.iter_commits(rev='master'):
    repo.git.checkout(commit)
    driver.get("file://" + REPO_DIR + FILE_PATH)
    time.sleep(3)
    driver.get_screenshot_as_file(SCREENSHOT_DIR_NAME + "/" + str(index).zfill(3) + ".png")
    index += 1
```

There were ~260 commits, so this took 13 minutes. After that, I used `ffmpeg` to string the PNGs together into a video.

```bash
ffmpeg -framerate 10 -pattern_type glob -i 'screenshots/*.png' -vf reverse -pix_fmt yuv420p -c:v libx264 out.mp4
```

And here's the end result!

<figure style="text-align: center">
	<iframe style="max-width: 600px; max-height: 435px; width: 90vw; height: 65.25vw;" max-height="435" width="90vw" height="65.25vw" src="https://www.youtube.com/embed/48ei9SThEyQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
	<figcaption style="margin-top: 1em;">The Evolution of My Personal Website</figcaption>
</figure>


My personal website is a tiny piece of who I am, and it evolves along with me. While the changes aren't drastic, they document transformations in my age, appearance, maturity, aesthetic, career, and programming skill over the last five years. I can only imagine what it'll look like five years from now!
