---
layout: post
title:  "Extracting vocabulary words from a Japanese text"
date:   2018-06-17 19:29:02 +0200
---

I'm always looking for new tools to help me in my learning of the Japanese language.
I've never been really fond of these applications that promise to teach you a language, because I feel like I'm learning much more when I read a paper book or when I take a pen and write words down.
That's why the tools I love are those who help me learn by myself.

These days, I've been using a few reading tools, that give you a pop-up when you click on a word when reading a text, telling you its meaning.
When the goal is to learn and memorize vocabulary, I find this approach totally inefficient, as it prevents the reader from making any effor.
Therefore, I thought to myself that it would be nice that when I want to read a text, I would be given a list of words appearing in that text, so that I could study it beforehand.

So here is today's objective: extracting the words of vocabulary from a text in Japanese, fetching their meaning from a dictionary, and providing this as a list.


# What's the plan?

I'm going to extract all the words from a text written in Japanese.
For this, I'll use a lexical analyzer, whose output I will have to filter so as to keep only the useful data.
Then, I'll use an online dictionary to retrieve these words' meanings.
For all this process, I'll use Python.


# Extracting segmented words out of an actual text

A lexical analyzer is a tool that can extract the words of a text, an operation called *segmentation*.
It can have other features, such as tagging the words (attaching them information such as their grammatical natures); however I'll only use the segmentation feature.

I've tried with [MeCab](mecab), but the results I got were not really satisfying. It seems pretty good however, so if you're lookin for a lexical analyzer, it's probably worth giving it a chance.
I came across [Kuromoji](kuromoji) as well, but did not try it, since it's designed for Java.
Then I found [Nagisa](nagisa), which looked promising, and it happened to work directly out of the box, so I decided to stick with it.

Nagisa is a module based on neural networks.
It's simply installed with `pip`, and can be tested easily:

```python
In [1]: import nagisa
[dynet] random seed: 1234
[dynet] allocating memory: 512MB
[dynet] memory allocation done.

In [2]: nagisa.tagging("今日はいい天気ですね").words
Out[2]: ['今日', 'は', 'いい', '天気', 'です', 'ね']
```

I'm now focusing on extracting all the words from an actual text.
The goal here is not to have these words in a plain form (辞書形); the issue related to forms will be handled later.

Let's test it on a more concrete file: [Wikipedia's article on Python, in Japanese](wikipedia-python-japanese).
I just made a Ctrl-A Ctrl-V of the page into a `python_wikipedia.txt` file, as all I need is a large amount of words.

I'm using the following script, to print out all the segmentized words from the input text:

```python
# nagisa_segmentize.py

import nagisa
import sys

with open(sys.argv[1], 'r') as file:
    content = file.read()
    for word in nagisa.tagging(content).words:
        print(word)
```

Let's do this:

```shell
python3 nagisa_segmentize.py python_wikipedia.txt > output
```

The `output` file now contains more than 10,000 lines, and the beginning looks like this:

```shell
$ head -25 output 

	
Python


Jump
　
to
　
navigation
Jump
　
to
　
search


曖昧
さ
回避
　
	
この
項目
で
```

There are three issues to address, that are, from the easiest to the hardest:

1. There are empty words
2. There are duplicates
3. There are non-Japanese words


## Removing empty words

I will use for this purpose a simple regex, to remove both empty and all-blank words.
These days I'm found of functional programming, so I'm gonna use `filter`.
Here it is:

```python
#! /bin/env python3

import sys
import re
import nagisa

with open(sys.argv[1], 'r') as file:
    content = file.read()
    for word in filter(
            lambda w: not re.match(r'^\s*$', w), 
            nagisa.tagging(content).words):
        print(word)
```

As expected, the empty words are filtered out.

I need to remove punctuation as well.
The `\W` regex metacharacter will help me out.
So I'm just adding a check against `r'\W'`, which represents a non-word character.
This way, I'll merely filter out any word that contains a non-word character.
This will include punctuation characters, but also potential weir results I'm not interested in.

```python
#! /bin/env python3

import sys
import re
import nagisa

with open(sys.argv[1], 'r') as file:
    content = file.read()
    for word in filter(
            lambda w: not re.match(r'^\s*$', w) and not re.match(r'\W', w), 
            nagisa.tagging(content).words):
        print(word)

```

So here is the filtered output:

```shell
python3 nagisa_segmentize.py ../data/python_wikipedia.txt | head -25
[dynet] random seed: 1234
[dynet] allocating memory: 512MB
[dynet] memory allocation done.
Jump
to
navigation
Jump
to
search
曖昧
さ
回避
この
項目
で
は
プログラミング
言語
に
つい
て
説明
し
て
い
ます
その
他
```

## Removing the duplicates

To remove the duplicates, I'll simply convert the list into a set.
A set is a data structure that cannot contain duplicates, so the conversion will remove them.
Furthermore, I don't really need the words to be contained in a list, and a set should be enough, so I'll not convert the set back into a list.
Note however that this breaks the order, and I might need to fix this later.

I'll modify my script so as it puts the words into a set, and displays it at the end.

```python
#! /bin/env python3

import sys
import re
import nagisa

words = set()

with open(sys.argv[1], 'r') as file:
    content = file.read()
    for word in filter(
            lambda w: not re.match(r'^\s*$', w) and not re.match(r'\W', w), 
            nagisa.tagging(content).words):
        words.add(word)

for word in words:
    print(word)
```

This is not really impressive, so I'm not gonna show the output here.


## Keeping only Japanese words

This is the tricky part.
I'm actually going to solve it in a very simple way, using Python's magic.
Regular expressions have a lot of metacharacter, but there's one that is less known that the rest: `\p`.
It allows to refer to a group of Unicode characters, called a Unicode category.
The categories that I am aiming for are `Hiragana`, `Katakana`, and `Han` (for kanjis, or hanzis in Mandarin Chinese), which are detailed in [this post on Localizing Japan](localizing-japan).

Unfortunately, Python's built-in module `re` does not support these categories.
But there exists a less known regular expression module for Python, called [`regex`][python-regex]G
This module handles a ridiculously broad range of ERE features, and provides `re`'s functions as well.

So using `regex` and the character categories, I'm just gonna filter out words that don't contain hiraganas, nor katakanas, nor kanjis.
This filtering might be a bit harsh, but it's gonna be good enough for now.

I'm going to write a `validate_word` function, so as to avoid writing all the filtering on a single line.

```python
import regex as re

def validate_word(word):
    return (    
            not re.match(r'^\s*$', word)
            and not re.match(r'\W', word)
            and re.match(r'\p{Hiragana}|\p{Katakana}|\p{Han}', word)
            )
```

The main section of the script does not change a lot:

```python
words = set()

with open(sys.argv[1], 'r') as file:
    content = file.read()
    for word in filter(
            validate_word,
            nagisa.tagging(content).words):
        words.add(word)

for word in words:
    print(word)
```

The output of this script is now only 1000 lines long, and contains only full Japanese words.

The goal of extracting a list of untouched words out of a text is now reached, although there might be some points to improve.
I will now focus on fetching the dictionary definitions for all these words.

# Getting the dictionary version of extracted words

In order to get the meaning of the extracted words, I will use Jisho website's API.
Jisho made a wonderful work, aggregating all the knowledge of Japanese language into a single website and a well-working API.
For each extracted word, I will send a request to this API with the module `requests`, and read the result.

The response received from Jisho's API will look like this:

```python
In [1]: import requests

In [2]: requests.get("https://jisho.org/api/v1/search/words?keyword=家").json()
Out[2]: 
{'data': [{'attribution': {'dbpedia': False,
    'jmdict': True,
    'jmnedict': False},
   'is_common': True,
   'japanese': [{'reading': 'いえ', 'word': '家'}],
   'senses': [{'antonyms': [],
     'english_definitions': ['house', 'residence', 'dwelling'],
     'info': [],
     'links': [],
     'parts_of_speech': ['Noun'],
     'restrictions': [],
     'see_also': [],
     'source': [],
     'tags': []},
    {'antonyms': [],
     'english_definitions': ['family', 'household'],
...
```

Let's write now a function that gets the response for a single word.

```python
import requests

def get_meaning(word):
    data = requests.get(API_URL.format(word)).json()['data'][0]

    reading = data['japanese'][0]['reading']
    meanings = [x['english_definitions'][0] for x in data['senses']]

    return reading, meanings
```

Since all the requests to the API are unrelated, I can write them as `asyncio` coroutines, and start them all in the main section.
I decide that each coroutine will fill a field in a global dictionary.

```python
async def get_meaning(dictionary, word):
    data = requests.get(API_URL.format(word)).json()['data'][0]

    reading = data['japanese'][0]['reading']
    meanings = [x['english_definitions'][0] for x in data['senses']]

    dictionary[word] = {'reading': reading, 'meanings': meanings}
```

And the main section is now a `main` function, marked `async` as well:

```python
async def main(meanings):
    words = set()

    with open(sys.argv[1], 'r') as file:
        content = file.read()
        for word in filter(
                validate_word,
                nagisa.tagging(content).words):
            words.add(word)

    print("Extracted {} words".format(len(words)))

    coroutines = [get_meaning(meanings, word) for word in words]
    await aio.wait(coroutines)
```

Finally, I setup the event loop and call the `main` function:

```python
if __name__ == '__main__':
    event_loop = aio.get_event_loop()
    try:
        meanings = {}
        event_loop.run_until_complete(main(meanings))
        print(meanings)
    finally:
        event_loop.close()
```

Let's try it on a short file first.
The `short.txt` file only contains the short sentence `今日はいい天気ですね。`

```shell
$ python3 nagisa_segmentize.py short.txt
[dynet] random seed: 1234
[dynet] allocating memory: 512MB
[dynet] memory allocation done.
Extracted 6 words
Received meaning for word いい
Received meaning for word ね
Received meaning for word 今日
Received meaning for word 天気
Received meaning for word は
Received meaning for word です
{'ね': {'reading': 'ね', 'meanings': ['root (of a plant)', 'root (of a tooth, hair, etc.)', 'root (of all evil, etc.)', "one's true nature", '(fishing) reef', 'Ne', 'Root']}, 'は': {'reading': 'は', 'meanings': ['topic marker particle', 'indicates contrast with another option (stated or unstated)', 'adds emphasis', 'Ha (kana)']}, '今日': {'reading': 'きょう', 'meanings': ['today', 'these days']}, 'です': {'reading': 'です', 'meanings': ['be']}, '天気': {'reading': 'てんき', 'meanings': ['weather', 'fair weather']}, 'いい': {'reading': 'よい', 'meanings': ['good', 'sufficient (can be used to turn down an offer)', 'profitable (e.g. deal, business offer, etc.)', 'OK']}}
```

The output is the following dictionary:

```python
{
    です: {'reading': 'です', 'meanings': ['be']}
    いい: {'reading': 'よい', 'meanings': ['good', 'sufficient (can be used to turn down an offer)', 'profitable (e.g. deal, business offer, etc.)', 'OK']}
    天気: {'reading': 'てんき', 'meanings': ['weather', 'fair weather']}
    は: {'reading': 'は', 'meanings': ['topic marker particle', 'indicates contrast with another option (stated or unstated)', 'adds emphasis', 'Ha (kana)']}
    今日: {'reading': 'きょう', 'meanings': ['today', 'these days']}
    ね: {'reading': 'ね', 'meanings': ['root (of a plant)', 'root (of a tooth, hair, etc.)', 'root (of all evil, etc.)', "one's true nature", '(fishing) reef', 'Ne', 'Root']}
}
```

# Performance issues

- Compile the regexes beforehand
- Use generators instead of a list generation, to filter lazily


# References

[localizing-japan]: http://www.localizingjapan.com/blog/2012/01/20/regular-expressions-for-japanese-text/
[python-regex]: https://pypi.org/project/regex/
[mecab]: https://taku910.github.io/mecab/
[kuromoji]: http://www.atilika.org/
[nagisa]: https://github.com/taishi-i/nagisa
[wikipedia-python-japanese]: https://ja.wikipedia.org/wiki/Python
