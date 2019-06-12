# NLP Final Report
## Problem Description	
題目: Finding Synonyms for a Given Word Sense
在常用的word2vec系統中尋找同義詞，常會發生兩種問題：
1.	如果取最高相似度的同義詞，通常會集中在比較高頻的詞性、語意的同義詞。其他較低頻的詞性、語意的同義詞，就不會出現。
2.	這些同義詞隨機排列，不分詞性、語意，就呈現給使用者。

因此我們希望透過其他輔助工具來改善這幾項缺點，讓使用者在搜尋字詞時能根據該字詞的不同定義來獲得相對應的同義詞，並能首先獲得較常使用的字詞。

## Dataset
### WordNet
WordNet是一個由普林斯頓大學認識科學實驗室所建立和維護的英語詞典，其中的單詞皆包含了語意訊息，根據不同的意義分成多個synset，爲同義詞的集合，除此之外每個集合有其相對的定義，並可以查詢其相對應的lemma、hyponym、hypernym等相對應關係。
因此我們期望利用WordNet中包含詞語的語意訊息的特點，來做為優化word2vec的工具。
![](https://i.imgur.com/BrdUemP.png)

 
### Word2vec
使用google news作為訓練文本，利用word2vec內建函示most_similar來找到相似度較高的詞，其中most_similar函式可以填入包含多個詞語的list，以找到與多個詞較相似的字詞，使用方式如下：
![](https://i.imgur.com/pUr1YEX.png)

### Wikipedia
使用Wikipedia提供的15GB的corpus來做為計算字頻的文本
### Wikipedia and WordNet alignment
使用下段paper所提供的16MB的alignment來做兩種dataset的對照

## Paper
Wikipedia的部分我們是參考一篇paper-"The People’s Web meets Linguistic Knowledge: Automatic Sense Alignment of Wikipedia and WordNet"，
這篇paper主要是透過對照Wikipedia和WordNet的字來對照，讓有精準意思分類的WordNet和有multilingual的Wikipedia做結合，所以可以有效的加強sense representation和增加sense coverage。
在WordNet的部份我們會去找一個字的lemmas, hyponym(下位詞), hypernym(上位詞)來當作WordNet要alignment的資料，至於維基百科的部分，我們萃取出可能為該字的alignment來當作之後找同義字的candidate，包含
1.	title符合原字的lemma
e.g., the article Window is retrieved for the synonym term Window
2.	title是title括號description tag
e.g., Window (computing)
3.	article是redirect來的
維基中的redirect是有著相關性的理由，像是一個article的可替代名字,複數名詞,非常相關的文字,縮寫,可替換的拼音方式...等，由於很接近，所以可以做為candidate
e.g., Chaff (counter-measure) has a redirect Window (codename) and, thus, is retrieved for the synonym term Window,
4.	article出現在超連結
e.g., the article Bandwagon effect is retrieved for the term bandwagon, as there exist a hyperlink of the form[[Bandwagon effect|bandwagon]]. 

接著計算ppr和pprd(initialize the PageRank algorithm solely with the synset)，pprd的效果比較好，我們是用他們做完的alignment的結果
系統架構

## 系統架構
![](https://i.imgur.com/C9hBOxf.png)

## Combination

利用word2vec most_similar函式，將目標詞語與其synset相關字詞作為參數，可找到與該目標詞語的特定解釋的相似字。我們實作上使用多種組合作為其synset相關字詞，以找到作為相似字的candidate，包含：
1.	Lemma*n (n=1~3) + Original word
2.	Hypernyms + Lemma + Original word
3.	Hypernyms  + Original word
4.	Hypernyms + Lemma + Original word
5.	Hyponyms  + Original word（在實驗步驟從candidate剔除）
6.	Hyponyms + Lemma + Original word
7.	Wiki title*n (1~3) + Original word
8.	Wiki title + Lemma + Original

![](https://i.imgur.com/ad7HyKL.png)

其中Lemma、Hypernyms、Hyponyms皆從WordNet獲得。在意義上lemma作為該synset的可替代詞，因此可以限制該詞語的特定解釋；Hypernym與Hyponym則為上位詞及下位詞，下位詞的語意領域被包含於上位詞中，兩者有著「type of」的關係，例如：red.n.01這個synset為顏色中的紅色的意思，其上位詞為chromatic_color，因為紅色為色彩的一種；其下為詞為scarlet、turkey red、dark red...等，因為這些皆為紅色的一種，hypernyms與hyponyms皆為特定解釋所相關，也可以限制詞語的解釋作為word2vec參數。

## Experiment
在所有測驗的combination中，根據多個詞的觀察中發現，因為hyponyms為下位詞，若是原本的target詞語本身較抽象，該字的下位詞可能包含較發散的詞語，若沒有結合lemma直接利用word2vec搜尋，可能導致結果較發散，不適合作為combination來尋找相似字，因此在實驗階段將其從candidate中移除。
![](https://i.imgur.com/izkeMdt.png)
![](https://i.imgur.com/7oiGmBN.png)

如上圖中的plant.n.02: ((botany) a living organism lacking the power of locomotion)作為植物的解釋時，因為下位詞包含胚胎所以在做word2vec搜尋時會搜出像是human embryos，較不恰當，因此將c7(combination 7：Hyponyms  + Original word）從candidate中移除。

## Frequency

我們希望能將每一個所選出來的同義詞的常用頻率顯示給使用者，因此我們利用wiki的資料來輔助，我們先做簡單處理讓原始資料可以變成一句一句的樣子，再去算unigram出現的次數和bigram出現的次數，最後針對每個lemma來計算frequency，做法是先取前五多的synonym並除以這個lemma的所有synonym的counts的總和。
做完之後我們發現因為bigram基本上出現的次數都遠小於unigram的數量，所以通常計算的時候都會被篩掉。

## Example Sentence
我們原本只有WordNet的例句，但是我們發現其實很多的詞的lemma都沒有例句，所以我們就想要用別的資源來協助找例句。因為我們是針對一個詞的每一個意思來提供例句，所以並不能單純就搜一個詞來找例句，而是要針對意思來搜例句。
所以我們就利用wiki和WordNet的alignment來做對照，找每一個wiki的第一段的第一句當作例句。為甚麼會選擇第一段的第一句是因為我們經由觀察發現第一段通常最有可能出現這個keyword，而第一句雖然有點像是definition，但是中間、後面的部分的這個keyword的意思就不一定跟我們現在要找的這個意思一樣了。
## Web
![](https://i.imgur.com/Uhb5Hhs.png)

我們的介面是用Django的framework實作的，版面是參考網路資源自己設計的。
另外我們還有把它放到網路上的免費server上，如下:
http://hsuanlyh.pythonanywhere.com/nlpFinal

## Conclusion
我們發現我們的做法可以有效的搜出好的同義字。
p.s.code還沒有整理
