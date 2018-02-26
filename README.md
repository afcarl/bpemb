# BPEmb

BPEmb is a collection of pre-trained subword embeddings in 275 languages, based on Byte-Pair Encoding (BPE) and trained on Wikipedia. Its intended use is as input for neural models in natural language processing. [arxiv](https://arxiv.org/pdf/1710.02187.pdf)


## tl;dr

- Subwords allow guessing the meaning of unknown / out-of-vocabulary words. E.g., the suffix *-shire* in *Melfordshire* indicates a location.
- Byte-Pair Encoding gives a subword segmentation that is often good enough, without requiring tokenization or morphological analysis. In this case the BPE segmentation might be something like *melf ord shire*.
- Pre-trained byte-pair embeddings work surprisingly well, while requiring no tokenization and being much smaller than alternatives: an 11 MB BPEmb English model matches the results of the 6 GB FastText model in our evaluation.


## Example

Apply [BPE](https://github.com/rsennrich/subword-nmt) with 3000 merge operations, using [SentencePiece](https://github.com/google/sentencepiece):

```bash
$ echo melfordshire | spm_encode --model data/en/en.wiki.bpe.op3000.model
▁mel ford shire
```

Load an English BPEmb model with [gensim](https://github.com/RaRe-Technologies/gensim) and get BPE embedding vectors:

```Python
>>> from gensim.models import KeyedVectors
>>> model = KeyedVectors.load_word2vec_format("data/en/en.wiki.bpe.op3000.d100.w2v.bin", binary=True)
INFO:gensim.models.keyedvectors:loaded (3829, 100) matrix
>>> subwords = "▁mel ford shire".split()
>>> subwords
['▁mel', 'ford', 'shire']
>>> bpe_embs = model[subwords]
>>> bpe_embs.shape
(3, 100)
```

## Overview

- [What and why](#what-are-subword-embeddings-and-why-should-i-use-them)
- [How to use BPEmb](#how-to-use-bpemb)
- [Number of BPE merge operations](#how-should-i-choose-the-number-of-bpe-merge-operations)
- [Download](#download-bpemb)

#### What are subword embeddings and why should I use them?

If you are using word embeddings like word2vec or GloVe, you have probably encountered out-of-vocabulary words, i.e., words for which no embedding exists. A makeshift solution is to replace such words with an `<unk>` token and train a generic embedding representing such unknown words.

Subword approaches try to solve the unknown word problem differently, by assuming that you can reconstruct a word's meaning from its parts. For example, the suffix *-shire* lets you guess that *Melfordshire* is probably a location, or the suffix *-osis* that *Myxomatosis* might be a sickness.

There are many ways of splitting a word into subwords. A simple method is to split into characters and then learn to transform this character sequence into a vector representation by feeding it to a convolutional neural network (CNN) or a recurrent neural network (RNN), usually a long-short term memory (LSTM). This vector representation can then be used like a word embedding.

Another, more linguistically motivated way is a morphological analysis, but this requires tools and training data which might not be available for your language and domain of interest.

Enter Byte-Pair Encoding (BPE) [[Sennrich et al, 2016]](http://www.aclweb.org/anthology/P16-1162), an unsupervised subword segmentation method. BPE starts with a sequence of symbols, for example characters, and iteratively merges the most frequent symbol pair into a new symbol.

For example, applying BPE to English might first merge the characters *h* and *e* into a new symbol *he*, then *t* and *h* into *th*, then *t* and *he* into *the*, and so on.

Learning these merge operations from a large corpus (e.g. all Wikipedia articles in a given language) often yields reasonable subword segementations. For example, a BPE model trained on English Wikipedia splits *melfordshire* into *mel*, *ford*, and *shire*.

Applying BPE to a large corpus and then training embeddings allows capturing semantic similarity on the subword level:

```Python
>>> model.most_similar("shire")
[('ington', 0.7028511762619019),
 ('▁england', 0.700973391532898),
 ('ford', 0.6951344013214111),
 ('▁wales', 0.6882895231246948),
 ('outh', 0.6406722068786621),
 ('▁kent', 0.6272492408752441),
 ('bridge', 0.619121789932251),
 ('well', 0.6175765991210938),
 ('▁scotland', 0.6023901104927063),
 ('orth', 0.5902647972106934)]
```

The most similar BPE symbols include many English place suffixes like *ington* (e.g. Islington), *ford* (Stratford), *outh* (Plymouth), *bridge* (Cambridge), as well as parts of the UK (England, Wales, Scotland).

The symbol *osis* does not exist after 3000 merges, but is created when using more, e.g. 10,000 operations:

```Python
>>> model_10k = KeyedVectors.load_word2vec_format("data/en/en.wiki.bpe.op10000.d100.w2v.bin", binary=True)
INFO:gensim.models.keyedvectors:loaded (10817, 100) matrix
>>> model_10k.most_similar("osis")
[('▁disease', 0.8588078618049622),
 ('▁diagn', 0.8428301811218262),
 ('itis', 0.8259040117263794),
 ('▁cancer', 0.7827620506286621),
 ('▁treatment', 0.7825955748558044),
 ('▁patients', 0.7808188199996948),
 ('▁dise', 0.7452374696731567),
 ('▁tum', 0.7444864511489868),
 ('ysis', 0.738912045955658),
 ('▁therap', 0.7286049127578735)]
```

A similar example with a common German place name suffix:

```Python
>>> model_de = KeyedVectors.load_word2vec_format("data/de/de.wiki.bpe.op10000.d100.w2v.txt")
>>> model_de.most_similar("ingen")
[('lingen', 0.8205140233039856),
 ('hausen', 0.7590259313583374),
 ('hofen', 0.7375717163085938),
 ('heim', 0.714651346206665),
 ('bach', 0.6965473294258118),
 ('sheim', 0.6638030409812927),
 ('weiler', 0.6597662568092346),
 ('dorf', 0.6320345401763916),
 ('▁bad', 0.630476176738739),
 ('berg', 0.6079661846160889)]
```

And with the German equivalent of *-osis*:

```Python
>>>model_de.most_similar("ose")
[('krank', 0.7024262547492981),
 ('▁erkrank', 0.625088095664978),
 ('itis', 0.611713171005249),
 ('▁behandlung', 0.5849611163139343),
 ('▁krankheit', 0.5647835731506348),
 ('hy', 0.55904620885849),
 ('fekt', 0.5524205565452576),
 ('pt', 0.5486388206481934),
 ('apie', 0.5447515249252319),
 ('▁krank', 0.5376874804496765)]
```


#### How to use BPEmb

1. Preprocessing: Lowercase the text you want to encode, replace all digits with 0, and replace all URLs with `<url>`. See `preprocess_text.sh` for the exact commands used.

```bash
$ ./preprocess_text.sh my_text.txt
```

2. Apply BPE: Having installed [SentencePiece](https://github.com/google/sentencepiece) and downloaded a SentencePiece model for the language and number of merge operations you want, e.g. with the English 3000 merge op model downloaded to data/en/:

```bash
$ spm_encode --model data/en/en.wiki.bpe.op3000.model < my_text.txt.clean > my_text.bpe3000
```

If you prefer Python, install the SentencePiece Python wrapper:

```
pip install sentencepiece
```

and use it like this:

```python
import sentencepiece as spm
sp = spm.SentencePieceProcessor()
sp.Load("data/en/en.wiki.bpe.op3000.model")
sp.EncodeAsPieces("This is a test")
```

If you don't want to have any dependencies, you can also use the simple byte-pair encoder in `bpe.py` (thanks to @jbingel https://github.com/bheinzerling/bpemb/issues/10).

3. Use in your favourite deep learning framework: `my_text.bpe3000` now contains a whitespace-separated sequence of BPE symbols. Convert these symbols to indices like you would with a word-based token sequence, load the corresponding embeddings, in this case `en.wiki.bpe.vs3000.d100.w2v.bin`, and create an embedding lookup layer. 


#### How should I choose the number of BPE merge operations?

The number of BPE merge operations determines if the resulting symbol sequences will tend to be short (few merge operations) or longer (more merge operations). Using very few merge operations will produce mostly character unigrams, bigrams, and trigrams, while peforming a large number of merge operations will create symbols representing the most frequent words:

| Merge ops | Byte-pair encoded text |
| - | - |
| 5000 | 豊 田 駅 ( と よ だ え き ) は 、 東京都 日 野 市 豊 田 四 丁目 にある |
| 10000 | 豊 田 駅 ( と よ だ えき ) は 、 東京都 日 野市 豊 田 四 丁目にある |
| 25000 | 豊 田駅 ( とよ だ えき ) は 、 東京都 日 野市 豊田 四 丁目にある |
| 50000 | 豊 田駅 ( とよ だ えき ) は 、 東京都 日 野市 豊田 四丁目にある |
| Tokenized | 豊田 駅 （ と よ だ え き ） は 、 東京 都 日野 市 豊田 四 丁目 に ある |
| | |
| 10000 | 豐 田 站 是 東 日本 旅 客 鐵 道 ( JR 東 日本 ) 中央 本 線 的 鐵路 車站 |
| 25000 | 豐田 站是 東日本旅客鐵道 ( JR 東日本 ) 中央 本 線的鐵路車站 |
| 50000 | 豐田 站是 東日本旅客鐵道 ( JR 東日本 ) 中央 本線的鐵路車站 |
| Tokenized | 豐田站 是 東日本 旅客 鐵道 （ JR 東日本 ） 中央本線 的 鐵路車站 |
| | |
| 1000 | to y od a \_station is \_a \_r ail way \_station \_on \_the \_ch ū ō \_main \_l ine|
| 3000 | to y od a \_station \_is \_a \_railway \_station \_on \_the \_ch ū ō \_main \_line|
| 10000 | toy oda \_station \_is \_a \_railway \_station \_on \_the \_ch ū ō \_main \_line|
| 50000 | toy oda \_station \_is \_a \_railway \_station \_on \_the \_chū ō \_main \_line |
| 100000 | toy oda \_station \_is \_a \_railway \_station \_on \_the \_chūō \_main \_line |
| Tokenized | toyoda station is a railway station on the chūō main line |


The advantage of having few operations is that this results in a smaller vocabulary of symbols. You need less data to learn representations (embeddings) of these symbols. The disadvantage is that you need data to learn how to compose those symbols into meaningful units (e.g. words).

The advantage of having many operations is that many frequent words get their own symbols, so you don't have to learn how what the word *railway* means by composing it from the symbols *r*, *ail*, and *way*. The disadvantage is that you need more data to train good embeddings for these longer symbols, which is available for high-resource languages like English, but less so for low-resource languages like Khmer.


## Download BPEmb

Downloads for the 15 largest (by Wikipedia size) languages below. Downloads for all 275 languages are available in binary format readable by gensim or word2vec, and in plain text format: [bin](download.bin.md), [txt](download.txt.md).

