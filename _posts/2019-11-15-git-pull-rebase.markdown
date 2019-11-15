---
layout: post
title:  "使用git pull --rebase避免無用的merge出現"
date:   2019-11-15 12:02:00 +09
categories: Git
---

本篇分享了共同編輯同一個 branch的 時候避免一堆無用 merge 的方法。

### 前情提要

相信大家都有共同編輯同一個branch(分支)的經驗吧？例如，阿肥和心儀的女同事Kathy一起開發一個feature/xxx，阿肥做A function，Kathy做B function，Kathy工作效率神速，早早就把功能做完，push到cloud然後準備下班了，阿肥因為上班都在看批踢踢科科笑，所以做比較慢，等到要上傳的時候發現不能push ...

狀況如圖所示:
![git_pull_rebase](/assets/img/git_pull_rebase_tompqnmls.jpg)

心急的阿肥心想，Ｘ！寫完程式還不能上傳，等等怎麼約Kathy吃晚餐 T_T
阿肥看了一下錯誤訊息，唷，原來是Kathy先上傳了，這個輕鬆，我只要git pull就好啊，簡單~

所以阿肥快速的在Terminal下了以下指令
```bash
git pull origin feature/xxx
```
然後也不管中間跳出了什麼Merge訊息，都直接用vim 指令```:wq```一路往下去啦！！
最後直接
```bash
git push origin feature/xxx
```

打完收工，跟Kathy吃晚餐去囉！！

### 隔天

Kathy到辦公室以後第一件事情就是做git pull，用tig看一下commit tree以後開始尖叫

X!!! 死阿肥，你為什麼要做沒有用的merge !!!!!!!


### 原因解釋

原來阿肥昨天在做git pull的時候因為cloud上面的status和阿肥本機的status不同，所以系統會自動幫他做一個merge
畢竟git pull的原始意義其實就是git fetch加上git merge。

當然，做這樣的merge沒有不行，只是在很多只是小小共同編輯的情況下，這樣的merge會顯得沒有意義，而且會讓commit history變得有點亂亂的，見仁見智，有些人不在意，有些人對於這個可是很要求的呢 ^.<

### 解決方案
那有什麼辦法可以解決這樣的問題呢？事後亡羊補牢總是要的，不然Kathy以後都不跟阿肥吃晚餐怎麼辦Q_Q?

如果遇到這樣的問題，其實只要下一個rebase指令
```shell
git rebase -i a2
# 其中a2是Kathy的commit id
```
這樣就可以了。如果不熟悉rebase，可以參閱 [這裡](https://gitbook.tw/chapters/branch/merge-with-rebase.html)
也可以把rebase簡單想成*更改歷史的方法*，重新更改這個branch的基底

(當然，回到未來也有教我們，更改歷史是很危險的)

經過rebase以後，分支歷史就會被改變，如圖所示

![git_pull_rebase_after](/assets/img/git_pull_rebase_after.jpg)

這樣就可以回到一條主線了 :)

那如果想要在pull的時候就一次到位不要事後修改呢？


其實如同之前提到的， git pull 其實就等於 git fetch 加上 git merge ，所以 git pull 的預設值就是使用 merge ，

那我們是否可以讓 git pull 改成使用 git fetch 加 git rebase 呢？

答案是**可以**的，在執行 git pull 的時候，在後面加上一個 --rebase 的flag就可以了

```shell
git pull origin feature/xxx --rebase
```

### GUI 工具使用方式
上面講解了一些使用command line的方法，那如果是比較熟悉視覺化工具的朋友(例如SourceTree)呢？

這邊以SourceTree為例:

![螢幕快照 2019-11-15 下午11.37.10](/assets/img/螢幕快照%202019-11-15%20下午11.37.10.png)

當按下 Pull 的時候，記得選取以下選項:

![螢幕快照 2019-11-15 下午11.37.31](/assets/img/螢幕快照%202019-11-15%20下午11.37.31.png)

就可以達到一樣的效果了 :) 

### 結語
該如何選擇 rebase 和 merge 呢？其實沒有標準答案，不過經過今天筆者和同事討論以後，得到以下的小小結論

> 如果今天是多人協作同一個branch，並且功能性相同，造成diverge，那我們可以選擇使用 rebase ，畢竟我們是實作**相同**功能，但是如果今天是不同類型的功能，則會偏向使用merge，因為merge 有一種一個功能實作結束的味道XD 

所以，最重要的還是跟你的同事或是同學們溝通清楚，訂定一個統一的標準會比較好 :cool: 

<br><br><br><br>