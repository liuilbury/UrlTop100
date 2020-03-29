# UrlTop100
## 从大文件内利用小内存找出出现次数最多的前100个URl
### 目的
100GB url 文件，使用 1GB 内存计算出出现次数 top100 的 url 和出现的次数。
### 实现思想
通过Hash的方法将大文件拆分成小文件，分别求出小文件的Top100，最后进行归并将小文件的Top信息合并得出结果。
### 实现方法
- 通过随机生成测试数据，每条数据大小基本为40B,方便控制最后的生产的大文件大小。
- 通过单线程读取大文件，多线程写入小文件，过程用队列维护的方式，将文件拆分成200份。
- 通过多线程读取小文件，将小文件处理出Top100的URl,最后通过归并合并。
### 具体实现过程

#### 1、生成测试数据
  生成测试数据用到了C++11<random>中的random_device。URL的组成被分成三部分，Head+Body+Tail。  
  Head会随机选择"Http://"或"Https://"一种。  
  Body会随机截取预设字符串中随机长度的一段（为了让url的重复度提高，而放弃了全随机）  
  Tail会固定加上".com/problem_pid="并在最后随机生成一个0-99的数字。
  所以最终生成的Url长这样：Http://abcdefghijk.com/problem_pid=66
  
#### 2、拆分大文件
  1. 最初采用了读一行，将字符串hash，写一行的方式进行拆分，发现效率极低。  
  2. 然后发觉文件的打开与关闭占用大量时间，且内存并没有很好的利用起来，于是采取用Vector<std::string>v[200]的方式，将字符串按hash后值的存放在vector内，等到内存使用差不多后再统一执行写入操作。测试后发现时间减少，但仍然很慢。  
  3. 发觉cpu并没有好好利用起来，于是采取了单线程读多线程写的方法。read线程读到的数据放到queue中，write线程从queue中取得数据并将其hash后写入文件中，过程中使用互斥量进行约束。时间花费减少了一些。
  3. 最终考虑将上面两种方法结合起来，write线程取得数据hash后跟之前一样存放在vector内，等到一定数量再进行写入。  
  4. 在优化途中发现自写的hash方法太慢，最终改成了C++11<hash>里的hash函数，时间大幅减少。  
  5. 由于使用vector最后还是要一条一条写入文件，所以改成了用string进行字符串的拼接并一次性写入。

#### 3、合并小文件
  有了拆分文件时的经验，所以在处理小文件的时候也使用了多线程同时处理的方式。  
  首先初始化队列，向队列里放入0-199。  
  之后每个线程尝试取一个数字，并通过unordered_map保存出现次数，通过partial_sort计算出这个文件最多出现的前100个URL，保存在vector内。  
  最后通过归并的操作计算出出现最多的前100个URL。
  合并的操作是从一个原来有0-199数字的队列里取出两个，将这两个数字的vector进行合并，合并完成后将合并完的数据赋值给序号小的vector，0号vector将存放最后的答案。  
  原本打算使用多线程来进行合并操作，后来实验发现由于前面的处理，最后的合并操作很快就可以完成，所以放弃使用多线程。
  
### 程序目录结构
- inclde文件夹内存放.h文件
- main文件夹内存放.cpp文件
   - main.c是程序入口
   - Data.cpp里实现了创建测试数据的相关函数  
   - Url.cpp里实现了创建一条Url的相关函数  
   - spilt_Data.cpp里实现了拆分文件的相关函数  
   - merge_Data.cpp里实现了合并文件的相关函数  
### 程序演示
#### 运行后


### 优点和不足
- 优点
   - 可以在可以接受的时间里正确的算出出现次数top100的url，实现了原本的目的。  
   - 采取了多线程和一次操作的方式，利用了可以用的cpu和内存，降低了时间复杂度。  
- 不足
   - 即使进行了优化，但是时间复杂度依然有些些高。  
   - 结果Url不能满足出现次数相同时字典序小的优先。  
   - 无法保证拆分的文件大小均分，意味着也许会有较大的拆分文件出现，可能无法满足内存要求。
