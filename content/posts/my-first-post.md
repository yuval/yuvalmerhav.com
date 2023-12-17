+++
title = 'First Impression of GPT4'
date = 2023-03-16T13:34:48-05:00
draft = false
+++

## Introduction

GPT4 was just released and it's amazing. I use ChatGPT daily for all sorts of tasks, esp coding (in addition to Co-pilot etc.). I feel I now do more code reviews and apply fixes than actual coding.

I've seen posts on Twitter about the future of NLP and what problems are still worth tackling outside of large LMs. This is an interesting topic. I don't have the answer. I do have some thoughts but instead of rambling I will show an example in a domain GPT4 knows well. 

### Python and Text Encodings

I noticed that GPT doesn't specify encoding when it shows code examples involving opening a text file in Python. For example:

```
with open(file_path, 'r') as file:
    file_content = file.read()
```

I'm assuming that's because most code it has seen doesn't specify encoding even though it's still needed for portability. If you tell it you want the code to be portable or that you live in China for example, it'd recommend to add 'encoding='utf-8'. It clearly knows what it is for. 

A while ago I had a discussion with a friend about this topic so I decided to challenge GPT4 a bit. I showed it the following command line snippet:

```
$ python -c 'print("\u20ac \u9474")' >x
$ cat x
€ 鑴
```
There are 3 chars here. The Euro sign, space, and a [Chinese character](https://www.compart.com/en/unicode/U+9474).

I then asked it to tell me if the following command would crash if I'm using Python 3.7. 

```
$ LANG=C python -c 'import sys; print(open(sys.argv[1]).read())' x
```

(it doesn't; it correctly prints € 鑴)

GPT4's answer:

> The LANG=C environment setting forces the use of the C/POSIX locale, which is an ASCII-based locale. In this case, since the file x contains non-ASCII characters (the Euro sign (€) and the CJK Unified Ideograph (鑴)), the command:

> $ LANG=C python -c 'import sys; print(open(sys.argv[1]).read())' x

> might crash or raise a UnicodeDecodeError, as the C/POSIX locale cannot properly handle these characters.


The explanation is right but the answer is not. The reason is [PEP-540](https://peps.python.org/pep-0540) that was added in Python 3.7. I'm sure that GPT4 has seen a lot of related data during training, including the PEP-540 page and the Python 3.7 release notes, but it's a needle in a haystack of data it has seen about Python. Such dynamic domains with frequent behaviour change are confusing for all of us. Search is still a problem for complex domains and questions. 

Finally, I gave it the PEP-500 page as context as part of the prompt and asked it if it changes anything about its answer:

> Yes, it does. With the introduction of PEP 540 in Python 3.7, there's a new "UTF-8 Mode" that uses the UTF-8 encoding, regardless of the locale encoding, and changes the stdin and stdout error handlers to surrogateescape. This mode is off by default, but is automatically activated when using the "POSIX" locale.

> Since the command you provided sets LANG=C, which is an ASCII-based locale, the UTF-8 Mode will be activated:

> With the UTF-8 Mode active, Python will use the UTF-8 encoding to read the file x, and it will not crash due to UnicodeDecodeError, as it would have without the UTF-8 Mode.

You're hired! I didn't try it with GPT3/3.5. From playing with GPT4 in the last 24 hours I don't think any prior models would do better.
