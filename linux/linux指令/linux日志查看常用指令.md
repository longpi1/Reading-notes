# linux日志查看常用指令

## sort

排序.

假设有一個 `test.txt` 如下,

```
c 2
a 4
y 33
b 111
e 44
j 3
k 12
```

预设是看最前面排序.

```
❯ sort test.txt
a 4
b 111
c 2
e 44
j 3
k 12
y 33
```

反向可以加上 `-r`, `--reverse` reverse the result of comparisons

```
❯ sort -r test.txt
y 33
k 12
j 3
e 44
c 2
b 111
a 4
```

也可以搭配其他指令使用, 像是

```
cat test.txt | sort
```

指定栏位下去排序.

```
❯ sort -n -k 2 test.txt
c 2
j 3
a 4
k 12
y 33
e 44
b 111
```

`-n`, `--numeric-sort` 代表使用数字下去排序.

`-k`, `--key=KEYDEF` 代表指定栏位排序. 这边指定第二個栏位.

这边多补充一下，如果是像上面空格隔开，不用特别设置（因为默认）。

如果今天你的文件如下，是用逗号隔开的，

需要多加上 `-t` 设置你的分隔符号。

`-t`, `--field-separator=SEP` use SEP instead of non-blank to blank transition.

`test2.txt` 如下,

```
c,2
a,4
y,33
b,111
e,44
j,3
k,12
```

```
❯ sort -n -t , -k 2 test2.txt
c,2
j,3
a,4
k,12
y,33
e,44
b,111
```

通过`  -t` 设计使用 `,` 当作分隔符号.

## uniq

用來找出(刪除)重复的行.

```
❯ uniq --help
......
Filter adjacent matching lines from INPUT (or standard input),
writing to OUTPUT (or standard output).
......
Note: 'uniq' does not detect repeated lines unless they are adjacent.
You may want to sort the input first, or use 'sort -u' without 'uniq'.
......
```

需注意, 使用 `uniq` 的時候, 请先执行 `sort`.

因为 `uniq` 是去找临近的行做比较而已, 所以你必须先 `sort` 再进行`uniq`.

( 上面说明中也有说 `uniq` 不侦测重复的行, 除他们是临近的 )

示例 `test.txt` 如下,

```
11
33
66
44
55
66
55
11
66
33
```

如果你沒有先执行 `sort`, 直接执行 `uniq`, 你会发现是無效的,

```
❯ uniq test.txt
11
33
66
44
55
66
55
11
66
33
```

將日志內重复的行去掉,

```
❯ sort test.txt | uniq
11
33
44
55
66
```

也可以使用 `sort -u` 代替,

```
❯ sort -u test.txt
11
33
44
55
66
```

`-u`, `--unique` with -c, check for strict ordering.

计算行出现的次数,

```
❯ sort test.txt | uniq -c
      2 11
      2 33
      1 44
      2 55
      3 66
```

`-c`, `--count` prefix lines by the number of occurrences.

如果你有空白行, 可以加上 sed 指令去掉空白行(如下範例)

```
sort test.txt | sed '/^$/d' | uniq -c
```

输出重复的行,

```
❯ sort test.txt | uniq -D
11
11
33
33
55
55
66
66
66
```

`-D`print all duplicate lines

只输出重复的行 (只展示一次),

```
❯ sort test.txt | uniq -d
11
33
55
66
```

`-d`, `--repeated` only print duplicate lines, one for each group

只输出重复的行 ,

```
❯ sort test.txt | uniq -u
44
```

`-u`, `--unique` only print unique lines

## grep

```
# 格式
grep match_pattern file_name
```

加上 `--color` 可以把关键字加上顏色, 展示更清楚.

```
grep --color "search name" README.md
```

加上 `-C`, 代表要多展示头尾的行數,

```
grep --color -C 2 "search name" README.md
```

也可以一次搜索多个文档日志

```
grep "name" README_1.md README_2.md
```

也可以使用 星号 `*`

```
grep "print" *.py
```

排除某個字

```
grep -v "match_pattern" README.md
```

`-v`, `--invert-match` select non-matching lines

如果想要排除某些字元又要搜索某些字元,

可以依照需求如下使用,

```
grep -v "ignore" README.md | grep --color "match_pattern"
```

搜索当下目录资料夾內容

```
grep -r "search name" .
```

case insensitive case (不區分大小寫)

```
grep -i "name" README_1.md
```

展示行数

```
grep -n "name" README_1.md
```

要完全符合 `:80` 才會被捞出來

```
grep -w ':80' README_1.md
```

`-w`, `--word-regexp` 仅比较整個单词.

或操作

 grep -E ’123|abc’ filename  // 找出文件（filename）中包含123或者包含abc的行（ -*E 选项*可以用来扩展选项为正则表达式）

 egrep ’123|abc’ filename    // 用egrep同样可以实现

 awk ’/123|abc/’ filename   // awk 的实现方式

与操作

grep pattern1 files | grep pattern2 ：显示既匹配 pattern1 又匹配 pattern2 的行。

## sed

这个指令可以达到快速搜索，取代，删除文字

sed 主要是针对**行**进行处理, 然后处理的不是原文件, 而是复制出來的文件.

语法

```
sed -i '/匹配字串/d' textfile
```

`-i` 加上这个才會写入你的 textfile, 不然只会展示在 terminal 上.

刪除 empty lines

```
sed -i '/^$/d' textfile
```

刪除有数字 7 的行

```
sed -i '/7/d' textfile
```

刪除第一到第五行

```
sed -i '1,5d' textfile
```

刪除从 hello1 到 hello2 之间的所有行

```
sed -i '/hello1/, /hello2/d' textfile
```

替换语法

```
sed -i 's/匹配字串/替代字串' textfile
```

将每行出现的第一個 a 替换成 A

```
sed -i 's/a/A' textfile
```

將每行出現的全部的 a 替换成 A

```
sed -i 's/a/A/g' file
```

`g` 代表替换所有匹配字串

只印出有 `test` 的行

```
sed -n '/test/p' test.txt
```

`-n`, `--quiet`, `--silent` suppress automatic printing of pattern space.

`p` Print the current pattern space.

sed 也可以印出文件指定行,

```
❯ cat test.txt
1
2
3
4
5
6
```

展示特定行, 展示第 5 行

```
❯ sed -n 5p test.txt
5
```

展示第 3 行以及第 5 行

```
❯ sed -n -e 3p -e 5p test.txt
3
5
```

```
-e` script, `--expression=script
```

add the script to the commands to be executed

展示第 3 行到第 5 行

```
❯ sed -n 3,5p test.txt
3
4
5
```

展示第 1 行到第 3 行, 以及第 5行

```
❯ sed -n -e 1,3p -e 5p test.txt
1
2
3
5
```

## awk

这个指令是一個非常強大的文字分析工具

假设今天我们的输出如下

[![alt tag](https://camo.githubusercontent.com/707f5d81f9dab4c0587f5861797dea7e7f1a8b2270c4c3c390c8783802ee2b5f/68747470733a2f2f692e696d6775722e636f6d2f4768507136735a2e706e67)](https://camo.githubusercontent.com/707f5d81f9dab4c0587f5861797dea7e7f1a8b2270c4c3c390c8783802ee2b5f/68747470733a2f2f692e696d6775722e636f6d2f4768507136735a2e706e67)

把第 2,3,5,9 列輸出

```
ll | awk '{print $2,$3,$5,$9}'
```

[![alt tag](https://camo.githubusercontent.com/7d35c7c980c97981ef2c4c103e4dbc11f7c2a424f8b7dbc74bf8bbd6e5cb68cb/68747470733a2f2f692e696d6775722e636f6d2f6f3165785943712e706e67)](https://camo.githubusercontent.com/7d35c7c980c97981ef2c4c103e4dbc11f7c2a424f8b7dbc74bf8bbd6e5cb68cb/68747470733a2f2f692e696d6775722e636f6d2f6f3165785943712e706e67)

如果觉得丑, 可以用 printf 來排版

```
ll | awk '{printf "%-5s %-5s %-5s %-5s\n", $2,$3,$5,$9}'
```

[![alt tag](https://camo.githubusercontent.com/8949af755589381627d0ba07cebe37e925042df8e2714e412a9b4a98573ab052/68747470733a2f2f692e696d6775722e636f6d2f3952516a32386f2e706e67)](https://camo.githubusercontent.com/8949af755589381627d0ba07cebe37e925042df8e2714e412a9b4a98573ab052/68747470733a2f2f692e696d6775722e636f6d2f3952516a32386f2e706e67)

接过试着来过滤资料，

把 权限分数（第2列）分数是 2 以及 第3列是 twtrubiks 的取出来。

```
ll | awk '$2 == "2" && $3 == "twtrubiks" {print $0}'
```

`$0` 代表整行的所有內容.

[![alt tag](https://camo.githubusercontent.com/67bda59f7b80e6d81c869ae9f3b592dca17a6505f5a19ee8b030965ebb58ade0/68747470733a2f2f692e696d6775722e636f6d2f496c396a4746702e706e67)](https://camo.githubusercontent.com/67bda59f7b80e6d81c869ae9f3b592dca17a6505f5a19ee8b030965ebb58ade0/68747470733a2f2f692e696d6775722e636f6d2f496c396a4746702e706e67)

还可以进行统计，

把 权限分数（第2列）的分数进行 sum（排除 total）。

先排除掉第一列是 total 字符串的资料，

my_sum 是我们定义的变量。

```
ll | awk '$1 != "total" {my_sum+=$2} END{print my_sum}'
```

[![alt tag](https://camo.githubusercontent.com/2ce8e3bd55ef3f30ea62468fbf37f93d528d91ee6ae52cafbd2e0917a523f6e1/68747470733a2f2f692e696d6775722e636f6d2f6f3379585a6e542e706e67)](https://camo.githubusercontent.com/2ce8e3bd55ef3f30ea62468fbf37f93d528d91ee6ae52cafbd2e0917a523f6e1/68747470733a2f2f692e696d6775722e636f6d2f6f3379585a6e542e706e67)

也可以编写 if 逻辑，

把 权限分数（第2列）的分数为 3 的过滤出来，

接着打印出当前行数，并把第9列的文件名称转为大写。

```
ll | awk '{if ($2 == "3") print NR, toupper($9)}'
```

`NR` current record number in the total input stream.

[![alt tag](https://camo.githubusercontent.com/3a0778095d4eee1d0283622451d1f3016fdfd2dff496a5b54354f162927a4c93/68747470733a2f2f692e696d6775722e636f6d2f647a6c624d41412e706e67)](https://camo.githubusercontent.com/3a0778095d4eee1d0283622451d1f3016fdfd2dff496a5b54354f162927a4c93/68747470733a2f2f692e696d6775722e636f6d2f647a6c624d41412e706e67)

`NF` number of fields in the current record.

示例 `test.txt`

```
❯ cat test.txt
-rw-rw-r-- 1 twtrubiks twtrubiks 5  4月  2 20:08 a.py
```

目前的 field 数量,

```
❯ cat test.txt | awk '{print NF}'
9
```

最后一個 field,

```
❯ cat test.txt | awk '{print $NF}'
a.py
```

示例第一個 field,

```
❯ cat test.txt | awk '{print $1F}'
-rw-rw-r--
```