---
title: Cyberthon 2023 - Astockalypse (Author Writeup + Notes)
date: 2023-08-01
tags: Challenge Creation
categories: CTF Writeups
---
Cyberthon is an annual CTF competition targeted towards JC students in Singapore, jointly organised by HCI, CSIT, and the DIS. This year, I had the opportunity to author a challenge for the data science category, and Astockalypse was what I came up with.

At the end of this writeup are my thoughts before, during and after challenge creation, and how I could've improved from a creative and a technical standpoint.

## Challenge

> 📘 Challenge description:
>
> One of your agents has found a proprietary stock market prediction service after enumerating Apocalypse's webservers!
> 
> Are you able to exfiltrate their prediction algorithm for ehem "further analysis" before they disallow public access?

[Link to source code](https://github.com/Iscaraca/CTF-Challenges/tree/main/cyberthon2023/astockalypse)

The challenge involves two separate websites, one being the website exposed on Apocalypse's end and the other a submission portal to hand your investigation results in.

![Astockalypse front page](mainpg.png)
![Submission front page](mainpg2.png)

(I had a field day with these backgrounds btw)

The former site takes in a csv file of data points, and will make a stock market price prediction for every row. The goal of the challenge is to successfully exfiltrate the model using a black-box method.

## Writeup

To extract the model, it doesn't require much to come up with this simple three-step method:
1. Craft and present samples to the target model. These will be your x values.
2. Get enough samples and responses, or your y values, to build a solvable system of equations with the only unknowns now being the weights w and the model architecture f.
3. Find information about f, and solve the equation to find w.

What follows is the solution to the challenge, using python 3.

### Generate CSV to upload

```python
import numpy as np
import pandas as pd

from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error, mean_absolute_error
from sklearn import preprocessing

t1_range = (1, 1000)
t2_range = (1, 1000)
t3_range = (1, 1000)

# Generate 200 random sets of 3 numbers to 1dp
t1 = np.round(np.random.uniform(t1_range[0], t1_range[1], size=200), 1)
t2 = np.round(np.random.uniform(t2_range[0], t2_range[1], size=200), 1)
t3 = np.round(np.random.uniform(t3_range[0], t3_range[1], size=200), 1)

query_df = pd.DataFrame({'t1': t1, 't2': t2, 't3': t3})

query_df.to_csv('query.csv', index=False, header=None)
```

### After uploading query.csv and downloading output.csv

```python
output_df = pd.read_csv('output.csv', index_col=None, header=None, names=['price'])

query_df.reset_index(drop=True, inplace=True)
output_df.reset_index(drop=True, inplace=True)

# Combine x and y for visualisation purposes
training_dataset = pd.concat([query_df, output_df], axis=1)
print(training_dataset.head())
print(training_dataset.columns)

X = training_dataset.drop('price', axis=1)
y = training_dataset['price']

# Fit model to data
model = LinearRegression()
model.fit(X, y)

# Dump as pkl file
import pickle

with open("model.pkl", "wb") as f:
    pickle.dump(model, f)
```

Submitting the pkl file to the portal gives you the flag, `Cyberthon{Ap0c4lypt1c_m4rk3t_cr45h!!!!!}`.

## Thoughts

When I was asked to make a data science challenge for Cyberthon, I was kinda hesitant to do so. I remember being a participant in Cyberthon 3 years back, wondering why data science was even a category in a CTF competition. The standard for challenges of this category over the years has been kaggle-style problem statements, with metric-based grading systems for the models. The minor gripe I had with these problem statements was that they were scarcely related to cybersecurity if at all, and I could see participants being sceptical about the challenges and their relevance to their learning.

Because of this, I started to research on adversarial machine learning techniques ([the wikipedia](https://en.wikipedia.org/wiki/Adversarial_machine_learning) was a big help) and making wild storylines and ideas in my head for every possible attack strategy. Model extraction was by far the most feasible to implement, and I immediately set to work on a proof-of-concept.

In the outset, the challenge relied very heavily on published attack strategies and required a respectable deal of research to solve. I got pretty carried away with what was possible that for a large portion of the initial creation I failed to take into account the target demographic and aim for the challenge, which was to introduce the possibilities data science had in launching cyberattacks without being too intimidating. I quickly double-backed on the original idea and came up with the simplest idea I could think of, which is the final product you see above.

Having been a blur JC student previously also really helped put the difficulty into perspective. During the competition, I was glad to see that over 40 teams managed to solve Astockalypse, and I hope most of those teams left with awe and eager inquisitiveness as to what else is possible in this domain.

In the effort to make the challenge more approchable, I gave plenty of hints and nudges as to how the participants were supposed to extract the model via the 3 step process. In hindsight, this decision was very much regrettable. Coming up with the steps by oneself and seeing it work is undoubtedly an essential part of the learning process, and it being taken away from some participants did take away from the whole challenge of it all. Next time I make a challenge like this, I'll be sure to get plenty of feedback from not just my friends who have plenty of experience in programming and cybersecurity, but also people new to the sport to gauge the level of difficulty I should set the challenge at.

Thank you to everyone who came down to watch me share the solution to the challenge during the Cyberthon meet-and-greet. It was a pleasure talking to some of you.

## Making the challenge

The notebook I used to train the model is [on colab](https://colab.research.google.com/drive/1rQdOvRevISmqfNSZHTnbdHrlZ3wpaOGz?usp=sharing).