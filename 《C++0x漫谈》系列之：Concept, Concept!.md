## **《C++0x漫谈》系列之：Concept, Concept!**

说明：By 刘未鹏  原文链接：[https://blog.csdn.net/pongba/article/details/1726031](https://blog.csdn.net/pongba/article/details/1726031)

#### 《C++0x漫谈》系列导言

这个系列其实早就想写了，断断续续关注C++0x也大约两年有余了，其间看着各个重要proposals一路review过来：rvalue-reference, concepts, memory-model, variadic-template, template-alias, auto/decltype, GC, initializer-lists...

总的来说C++0x跟C++98相比的变化是及其重大的。这个变化主要体现在三个方面，一个是形式上的变化，即在编码形式层面的支持，也就是对应我们所谓的编程范式（paradigm）。C++0x不会引入新的编程范式，但在对泛型编程（GP）这个范式的支持上会得到质的提高：concepts, variadic-templates, auto/decltype, template-aliases, initializer-lists皆属于这类特性。另一个是内在的变化，即并非代码组织表达方面的，memory-model, GC属于这一类。最后一个是既有形式又有内在的，r-value references属于这类。

这个系列如果能够写下去，会陆续将C++0x的新特性介绍出来。鉴于已经有许多牛人写了很多很好的tutor （这里，这里，还有C++标准主页上的一些introductive的proposals，如[这里](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2006/n2087.pdf)，此外C++社群中老当益壮的Lawrence Crowl也在google做了非常漂亮的talk，注：原文中给出的链接均已失效），所以我就不作重复劳动了:)，我会尽量从一个宏观的层面，如特性引入的动机，特性引入过程中经历的修改，特性本身的最具有代表性的使用场景，特性对编程范式的影响等方面进行介绍。至于细节，大家可以见每篇介绍末尾的延伸阅读。

#### Concept

好吧好吧，我承认我跳票了，上次说这次要写variadic templates的。但p9老大写了[一篇精彩的散记](https://blog.csdn.net/g9yuayon/article/details/1694766)，让我觉得concept应该先写，因为这个实在是个有意思的特性，比variadic templates有意思多了。

###### 我和Concept不得不说的事儿#1

看看下面这坨代码有什么问题：

```c++
std::list<int> li;
std::sort(li.beign(), li.end());
```

如果对人肉编译不在行的话，可以用你手头的编译器试一下。你会发现，你的编译器一碰到这简单而无辜的两行代码便会一反常态，跟个长舌妇似的吐出一大堆&#$@*^，令人牙酸的错误信息来。在使用C++模板库时这种编译错误井喷是家常便饭，动辄喷出令人应接不暇的4K字节错误信息出来。你还以为不是编译器井喷，而是你自己的RP井喷了，于是一脸无辜地跑去问模板达人，后者抬了抬眼皮，告诉你说“把list改成vector因为list的iterator不是random的而std::sort需要random的iterator”，你一边在脑子里给这句话加标点一边想弄明白他是怎么从一堆毛线似的字符里抽象出这么个结论的。

实际上，这个问题比你想象得严重，其根本原因在于降低工作效率，你得在你本不需要花工夫的地方（人肉解析编译错误）花工夫；这个问题比你想象得普遍，乃至于居然有人把“能够独立解决所有的编译与链接问题”也列在了“有实际开发工作经验”要求里面；这个问题比你想象得影响恶略，因为你可以想象可怜的新手在两行貌似无辜的代码面前哭丧脸的模样——C++编译器就是这样把一个可怜的潜在C++用户给扼杀了。你也可以想象为什么有那么多人不喜欢C++模板——其实语法只是其一个非主要的方面。

实际上你请教的那个达人并没有什么火星抽象能力，只不过是吃过的桥比你走过的盐还多而已。而这，还预示着另外一个问题，就是能人肉解析模板编译错误居然也成为了衡量C++达人与否的一个标准… 不信你去各个坛子上转一转看看有多少帖子是询问关于编译错误的问题的，其中又有多少是关于模板编译错误的。

更小概率的是居然还存在一个专门解析STL相关错误信息的“[STL错误解码器](http://www.bdsoft.com/tools/stlfilt.html)”——STLFilt。这玩意帮你把编译错误转换成人能识别的自然语言，不错是不错。可惜STLFilt有了，BoostFilt呢？ACEFilt呢？我自己写的模板库呢？…

其实，造成这个问题的直接原因是C++的类型系统的抽象层次太低。C++的静态强（也有人说C++的类型系统其实是弱类型系统，anyway）类型系统所处的抽象层面是在基本类型（int、double、char…）层面的。一方面，C++虽然拥有对自定义类型的上乘支持（比如，支持将自定义类型的接口装扮得跟内建类型几乎毫无二致——vector vs. build-in array），然而另一方面，C++的类型系统却对于像vector这样的抽象从语意上毫不知情。直接的后果就是，一个高层的类型错误往往以相差了十万八千里的底层类型错误表现出来，结果就是你得充当一次福尔摩斯，从底层错误一直往上回溯最终找到问题的发生点。譬如一开始给出的那个例子：

```c++
std::sort(li.begin(), li.end());
```

的错误，如果C++类型系统的抽象层能高一些的话（所谓抽象层次高，就是知道高层抽象概念（Concept）的存在，如“随机迭代器”这个概念），给出的错误无非就是：“list的迭代器不满足随机迭代器这个概念(concept)的要求(requirements)”。然而由于C++并不知道所谓concept的存在，所以问题到它眼里就变成了“找不到匹配的operator+…”一堆nonsense。

###### 事儿#2

大二上学期的时候我们上一门计算方法的课程，期末考试要写一些矩阵算法。地球上的程序员大抵都知道矩阵算法不用Matlab算基本等于没事找抽，一大堆accidental complexities在那恭候着，一个index错误能让你debug到抓狂。当时我C++用得半斤八两，模板七窍也差不多通了六窍；为了到上机考试的时候节省点时间，就事先写了一个简单的矩阵库，封装了一些基本的操作和像高斯消元这种基本算法。 

那个时候你能指望我知道[TDD](http://en.wikipedia.org/wiki/Test-driven_development)？还是[XP](http://en.wikipedia.org/wiki/Extreme_Programming)？或者[STLLint](http://portal.acm.org/citation.cfm?id=1122569)？于是呢？写了一个简单的程序，简单使用了一下写好的库，发现编译通过后就兴冲冲地告诉哥们说：大家不用怕，有我这Matrix库罩着，写算法跟写伪码差不到哪去！ 

两天后上机考试，程序不同了，等于测试用例不同了，结果原来没有出现的编译错误一下统统跑出来了。原来为什么不出现？一个原因是原来有些成员函数就没用到，C++说，在一个模板类里面，没用到的成员函数是不予编译的。那不予编译就代表不予纠错吗？不予类型检查吗？令人悲伤的是，的确如此。或者把置信度提高一点说，几乎如此。为什么？看看下面的代码：

```c++
template<typename T>
void f(T& t)
{
	t.m();
}
```

你说编译器看着这个函数，它怎么做类型检查？它怎么知道t上面有没有成员函数m？它连t的类型都不知道。“很久很久以前，模板就是这样破坏模块式错误检查的…”

实际上，C++98那会，为了能够尽早尽量检查模板代码中的隐患，以响应“防范胜于救灾，隐患重于明火”的号召，C++甚至将模板上下文中的代码中的名字生生分成了两类，一类叫dependent names，一类叫non-dependent names。举个例子，上面那段代码中的m成员函数就是dependent的，因为它的隐含this参数t的类型是dependent的；对于dependent name，不作类型检查——原因刚才讲过，因为类型信息根本就没有。剩下的就是non-dependent names了，比如：

```c++
void g(double); // #1
template<typename T>
void f()
{
	g(1);
}

void g(int); // #2
int main()
{
	f<int>();
}
```

这里f里面调用的g绑定到哪呢？答案是#1。因为g是个non-dependent name（虽然它位于模板函数（上下文）里面）。而对于non-dependent name，还是赶紧进行类型检查和名字绑定吧，有错误的话也能早点暴露出来，于是g便在它的使用点“g(1)”处被查找绑定了——尽管#2处的g(int)是一个更好的匹配，但在g(1)处只有g(double)是可见的，所以g(double)被编译器看中了，可怜的g(int)只能感叹“既生g(int)，何生g(double)…”。

这，便是臭名昭著的腰斩…sorry…是二段式名字查找，C++著名的复杂性来源之一。说它臭名昭著还有一个原因——在众多编译器支持良莠不齐的C++复杂特性中，它基本可以说是位居第一（第二估计要留给友元声明了），VC挣扎到8.0还是没有实现二段式名字查找，而是把所有的工作留到模板实例化点上进行，结果就是上面的例子中会选中#2。

[D&E](http://www.amazon.com/Design-Evolution-C%2B%2B-Bjarne-Stroustrup/dp/0201543303/ref=pd_bbs_sr_1/102-6714136-3446564?ie=UTF8&s=books&qid=1186197281&sr=8-1)中对此亦有详细介绍。

实际上，这个二段式名字查找的种种问题正从一个侧面证明了早期类型检查是何等重要，动态语言的老大们在Ruby翻出来的旧瓶新酒Duck Typing上[吵翻了天](http://www.artima.com/forums/flat.jsp?forum=106&thread=209353)其实说的也是这个问题（sorry，要加上“之一”）。

###### 事儿#3

在一个无聊的午后，我在敲打一坨代码，这是一个算法，算法要用到一个容器，算法是用模板来实现的：

```c++
template<typename ContainerT>
void XXXAlgo(ContainerT cont)
{

… cont.
```

在我敲打出“cont”加点号“.”之后，我习惯性地心理期待着“智能”的IDE能够告诉我cont上面有哪些成员函数，正如我们每次敲打出“std::cout.”之后一样。习惯成自然，你能说我不对么？难道你金山糍粑用久了不也一样在读影印版纸书遇到不认识单词的时候想着把手指头伸过去指着那个单词等着跳出个词条窗口来？难道只是我？咳咳…

问题是，我知道XXXAlgo的那个模板参数ContainerT是应当符合STL的Container概念（concept）的，我当然希望编译器也能知道，从而根据Container概念所规定它必须具有的成员函数来给我一个成员函数列表提示（begin，end，size…），难道这样的要求很过分吗？它没有道理很过分啊，觉得它很过分我会说的啊，不可能它明明不过分我偏要说它很过分，他很过分我偏要说它不过分啊…你觉得这要求过分你就说嘛…咳…乱敲键盘是不好滴，键帽掉下来砸到花花草草也不好啊…你看，“.”键又给你磨平了… 

一方面，程序员一脸无辜地认为IDE应该能够看到代码里面的ContainerT暗示着这是一个符合STL的Container概念的类型。而另一方面IDE厂商却也是理直气壮：写个ContainerT就了不起啊，万一遇到个C过来的，写成ContT我怎么办？写成CntnrT哪？是不是要我实现一个spell checker？再说你觉得ContainerT是对应STL的Container概念的，别人还用这个单词来对应线程池呢怎么办捏？什么？他不知道“poor”怎么写管我啥事嘞？我身为一个IDE，根据既有的信息，作出这样的假设，既合情，也合理…

###### 事儿#4**（此事纯虚虚构，如有巧合，算你运气背）**

一天，PM跑过来告诉你说：“嘿，猜怎么着，你写的那坨模板代码，隔壁部门人用了说很不错，希望你能把代码和文档完善一下，做成一个内部使用的库，给大家用，如何？”你心头一阵花枝乱颤：“靠！来部门这么久了，C++手段终于可以展露一下了。”于是废寝忘食地按照STL文档标准，遵照C++先贤们的教诲，写了一个漂漂亮亮的文档出来。里面Concept井井有条，Requirements一丝不苟… 

动态语言的老大们常挂在嘴边的话是什么？——需求总是在变的。又一天，你发现某个Concept需要revise了，比如原来的代码是这样的：

```c++
template<typename XXX>
void f(XXX a)
{
  	…
	a.m1();

```

本来XXX所属的那个Concept只要求有m1成员函数。后来因需求变更，XXX上需要一个新的成员函数m2。于是你的代码变成了：

```c++
template<typename XXX>
void f(XXX a)
{
  	…
	a.m1();
	…
	a.m2();
}
```

但仅改代码是不行的，文档里面关于XXX所属的那个Concept的描述也要同步修改…可惜天色已晚，良宵苦短，你准备睡一觉明天再说…结果第二天一早你就被boss叫去商量一个新的项目（因为你最近表现不错），于是你把这事给忘了。于是跟代码不一致的文档就留在那里了…

这种文档和代码不一致的情况太常见了，根本原因是因为代码和文档是物理上分离的，代码不能说谎，因为要运行，但文档呢？什么东西能验证文档精确反映了代码呢？除了往往忽视文档的程序员们之外没有其他人。这个问题是如此广泛和严重以至于程序员们干脆就近乎鸵鸟式地倡导“代码就是文档”了，这句话与其说是一个陈述句，不如说是一个美好的愿景（远景？）。

好吧，好吧，你记性好，这点小事你不会忘掉，第二天一早你就把文档给改了，你真是劳模。可惜过一天，需求居然又改变了（你心说是哪个家伙负责客户需求分析的？！），这下你需要修改Concept继承体系了… 

你看，可能造成文档和代码脱节的因素太多了，一般一段时间以后，能说得上话的也就剩代码，文档只能拿来看看“系统应该是什么样子的”，只有代码才陈述了“系统实际是什么样子的”。

然而，如果文档就在代码当中呢？不，我不是说注释，你又不是不知道要写出合格的注释比写出合格的小说还要难。我是说，代码就是文档文档就是代码…

此外，把Concept约束写在代码里面还有一个好处就是能够使得被调用函数和调用方之间的契约很明显，Concept的作用就像门神，告诉每一个来调用该函数的人：“你要进去的话必须满足以下条件…”。Ruby的Duck Typing被诟病的原因之一就是它的Concept在代码里面是隐式的，取决于对象上的哪些方法被使用到了。

###### 事儿#5

[重构重不重要](http://www.amazon.com/Refactoring-Improving-Design-Existing-Code/dp/0201485672/ref=pd_bbs_sr_1/102-6714136-3446564?ie=UTF8&s=books&qid=1186194354&sr=8-1)？Martin Fowler叔叔笑了…

原来我抽屉里有这么一段代码：

```c++
template<typename XXXConcept>
void foo(XXXConcept t)
{
	…
	t.m1(); // #1
	…
}

template<typename XXXConcept>
void bar(XXXConcept t)
{
	…
	t.m1(); // #2
	…
}
```

现在我想对代码作一种最简单的重构——改名。m1这个名字不好听，我想改成mem。于是我指望编译器能替我完成这个简单的任务，我把鼠标指到#1处，在m1上右击，然后重命名m1为mem。同时很显然我期望“智能”的编译器能够帮我把#2处也改过来，因为它们用的是同一个concept上的成员函数。

但编译器不干，原因见事儿#3。或者见[这篇blog](http://beust.com/weblog/archives/000414.html)，后者举了一个类似的例子——如果我们重命名实现了那个XXXConcept的类上的m1方法，那么#1和#2处的调用能被自动重命名吗？[Ruby Refactoring Browser](http://www.kmc.gr.jp/proj/rrb/structure-en.html)的作者笑了…

###### 事儿#6

很久很久以前…我写了一个容器类。这个容器类里面该有的功能都有了…唯一的问题是，当时我还不知道STL（准确地说是就算知道也没用），结果呢？这个各方面功能都完备的容器类的使用界面（接口）并不符合STL容器的规范。比如我把begin()叫做start()，把end()叫做…还是叫做end()（不然总不能叫finish()吧？）。我还把empty()叫做isEmpty()了…而另一方面我的empty()实际却做的是clear()的工作…

后来，我又写了一个算法，这个算法是针对STL容器的，你问我干嘛不针对迭代器编程？很简单，因为我要用到empty、front/back、clear等成员函数。基于迭代器编写也…不是不行，就是得再费一袋烟…此外还有两个问题，一是效率，而是影响算法使用…呃…说到效率… 

现在，我想让我的这个算法也能操纵我原来那个古董容器（我不是指我家那个慈禧尿壶），但因为那个古董容器的接口跟我的算法要用到的接口不一致：

```c++
class MyCont { … bool isEmpty(); … };

template<typename Cont>
void f(Cont cont){ … cont.empty(); … }
```

怎么办？修改MyCont的实现？可以，因为这个MyCont是我写的，后者意味着两点：一，我有权修改它。二，我写的库没其他人用。可是如果MyCont是位于另一个库当中的呢？如果有一堆依赖于它的既有代码呢？

或者，写个wrap类？还是太麻烦了，况且wrap手法也不是没有自己的问题。我们只不过想适配一下接口而已。 

其实，我们只想对编译器说一句：MyCont的isEmpty其实就是empty，您行行好就放行吧…

###### 事儿#7

函数重载重不重要？废话。多态重不重要？还是废话。

那[SFINAE](http://www.boost.org/libs/utility/enable_if.html)重不重要？C++ Templates的geeks们都笑了…

在C++里面，SFINAE技术已经成为GP的奠基技术之一（老大当然是sizeof技术）。boost里面为此专门引入了一个库，叫[boost::enable_if](http://www.boost.org/libs/utility/enable_if.html)。该库，如boost::mpl一样，被boost里面的众多子库依赖。比如[boost::function库](http://blog.csdn.net/pongba/archive/2007/04/11/1560773.aspx)就用到了该技术。 

简而言之，SFINAE技术允许你根据类型的编译期信息实现多态行为：

```c++
template <class T>
typename enable_if<boost::is_arithmetic<T>, T>::type
foo(T t) { return t; }
```

如果T是算术类型，那么这个foo函数模板就能够实例化，否则就不会。

另一项相关的模板技术是Tag Dispatch。STL中大量运用这种手法：

```c++
template<typename InputIter>
typename iterator_traits<InputIter>::difference_type
distance(InputIter it1, InputIter it2)
{
  	return distance(it1, it2,
	typename iterator_traits<InputIter>::iterator_category());
}
```

如果`typename iterator_traits<InputIter>::iterator_category`是`random_access_iterator_tag`的话就会跳转到： `distance(InputIter it1, InputIter it2 , random_access_iterator_tag){ … }`；如果是`bidirectional_iterator_tag`的话就会跳到：`distance(InputIter it1, InputIter it2 , bidirectional_iterator_tag) { … }`。

然而，这些与其说是技术（techniques），不如说是技巧（tricks）。它们的存在增加了C++中GP的accidental complexity。它们，正如大多数的C++模板技巧一样，本不在C++设计的考虑之内，而是后来被人们发现出来的。当然你可以说C++是唯一一门语言之父本人需要别人教他怎么用的语言，这的确很奇妙，然而当程序员抓耳挠腮地对付这些技巧的时候，恐怕更多的是恼火。比如这个：

```c++
template <typename _Iterator>
struct iterator_traits {
  typedef typename _Iterator::difference_type difference_type;
};

template <typename _InputIterator>
inline typename iterator_traits<_InputIterator>::difference_type
distance(_InputIterator, _InputIterator);
double distance(const int&, const int&);

void f() {
  int i = 0;
  int j = 0;
  double d = distance(i, j);
}
```

符合SFINAE的条件吗？不符合吗？符合吗？…

说到底，这些细节本不该由程序员来操心。我们想要的是一个简洁明了地表达我们想法的工具。

###### 事儿#8

如果一个生物走起路来像个火星人，

说起话来像个火星人，

回起贴来像个火星人，

那他肯定就是火星人。

——火星人判别最高纲领

 

Ruby的串红使[火星人类型系统](http://en.wikipedia.org/wiki/Duck_typing)焕发出了第二春。

Rubyers的口号是，我不关心你是不是真的是火星人，看你丫的回帖像刚从火星回来的，你一定就是火星人！ 

Sorry，用严肃一点的话来说，就是“不关心一个对象的具体类型，而只关心一个对象的行为”。用镐头书上的例子就是：

```ruby
class Customer
    def initialize(first_name, last_name)
        @first_name = first_name
        @last_name = last_name
    end

    def append_name_to_file(file)
        file << @first_name << " " << @last_name
    end
end
```

file << @first_name，这里file并不一定要是真正的文件，而只要是一个支持“<<”操作的对象即可（想起C++的流插入符了吗？）。所以要测试这个Customer类的append_name_to_file，也就不一定要真正创建一个文件出来传给它当参数，只要传一个支持<<操作的对象给它就可以了——比如一个String对象。用String对象的好处就是检查被写入到这个String里面的东西很容易，而用File对象的话还得开文件关文件的，麻烦。

事实证明Ruby的火星人类型系统是非常灵活的。镐头书上还举了另一个实际的例子：有这么一坨代码，它遇到数据量大的时候就变得奇慢，项目期限在即，当花了一点时间检查问题所在之后，发现问题在于代码中创建了许多的String临时对象，而速度问题则是因为GC运行起来了，之所以有这么多String临时对象是因为一个循环里面不断往一个String对象上面Append新子串，导致原来的串对象被丢弃，一地鸡毛。 

结果还是火星人类型系统“to the rescue”。通过仅改变一两行非关键代码，该项目得救了。作出的改变就是把那个String对象换成一个Array，这样每次往上面Append新串的时候都会把这个新串当作一个新的元素挂到这个Array对象上，活像一串串腊肉；由于没有旧串被丢弃，因此也就不会出现遍地垃圾的情况。

好吧，我承认我在练习中学语文老师教的欲抑先扬手法。不过Ruby Fans大可不必激动，因为两个原因：一，C++也有同样的问题，所有的模板代码用的本质上也都是火星人类型系统（C++98只支持完全unconstrained templates）——管你实际上是不是迭代器，只要你有++、--、*、->等操作就行。二，火星人…类型系统的危险是理论上的，实际上谁也没有案例证明它导致了什么灾难。比如在Cedric的blog上这篇“The Perils of Duck Typing”后面就有人跟贴说一个JarFile上有一个explode（解压）和一个NuclearBomb上有一个explode（爆炸），于是你的算法把一个核弹给“解压”（爆炸）了。从这个例子的极端也不难看出其实这种危险的可能性并不大。有一次我在新闻组上发帖，扯到这个问题上，也有人举了一个例子，说手指有一个方法叫“插”（Plug），而插头也有一个方法叫“插”（Plug），于是不管三七二十一的算法就面临把一根手指（谁的谁倒霉）插到插座中去的危险。

这些例子说到底都有点飘逸，不切实际。具体的例子你问我我也没有，或许C++里面倒是可以捏造出一个“比较”实际一点的来：

```c++
template<typename StreamT>
void f(StreamT& stream) { stream << 1; }

int i;
…
f(i);
```

这段代码编译器也乐呵呵地编译了，因为整型是支持<<（位移）操作的。这里的错误很显然，但若是藏在成千上万代码当中，因为一个打字错误而漏进去的话，也许就不那么显然了。

从本质上说，火星人类型系统是[将语法结构的同一性视为语意层面的同一性](http://blog.csdn.net/pongba/archive/2007/07/29/1715263.aspx)（即所谓的[Structural Conformance](http://en.wikipedia.org/wiki/Structural_type_system)）；这才是它的根本问题。而另一方面，传统的接口继承（即所谓的[Nominal Subtyping](http://en.wikipedia.org/wiki/Nominative_type_system)）则更严格：当你继承自一个接口的时候，你明确而清醒地知道你是要实现该接口所表达的抽象（语意），你不会“一不小心”（accidentally）实现了一个接口的，你必须写上“implements …（Java）/ public … （C++）”几个大字…母才行。

###### Concept to the rescue

话说到这份上如果你还不知道我要说什么…咳…那我就继续说吧…

以上八大问题一直都是GP中被广为争论的问题，其中duck typing（#8）在动态语言社群争论得比在C++里面还要激烈得多；同时它们也都是由来已久的问题，有的甚至久远到Bjarne在D&E中就已经遇见到了，只是当时C++标准化的进度太紧来不及解决而已，这一晃就是10年…

没错，它们全部都可以用Concept来漂亮地解决。或者换个说法，Concept的出现就是为了解决以上这些问题的——

1（编译错误问题）——有了Concept，C++的类型系统抽象层次便提高了一个级别，在遇到编译错误的时候便能说“XXX不满足XXXConcept”这样的话了。 

2（模块式类型检查问题）——有了Concept，原本所谓的unconstrained templates便可以做成constrained。比如STL的for_each算法就变成了这样： 

```c++
template<InputIter Iter, Callable1 Fun>
Fun for_each(Iter first, Iter last, Fun func)
{
  	for(;first != last; ++first) func(*first);
  	return func;
}
```

其中InputIterator和Callable1都是Concepts：

```c++
auto concept Callable1<typename F, typename X> {
  	typename result_type;
  	result_type operator()(F&, X);
};

concept InputIterator<typename X> : EqualityComparable<X>, …
{
	…
  	pointer operator->(X);
  	X& operator++(X&);
  	postincrement_result operator++(X&, int);
  	reference operator*(X);
}
```

有了这些Concept，编译器在对for_each作类型检查的时候便能够往InputIterator/Callable1里面进行名字查找：“first != last”可不可行？只要看first的类型支不支持“!=”，那first的类型支不支持“!=”呢？因为first的类型Iter是满足concept InputIterator的，那就只要看InputIterator里面有没有“!=”就行了，有吗？没有？哦，不好意思，忘记说了，InputIterator是继承自EqualityComparable的，后者里面定义了“!=”。

```c++
auto concept EqualityComparable<typename T, typename U = T> {
	bool operator==(T a, U b);
	bool operator!=(T a, U b) { return !(a == b); }
}
```

同样的，“++first”这个表达式可行吗？只要看first的类型支不支持“++”就可以了，后者只要看InputIterator这个concept支不支持“++”，答案是支持。 

3（IDE智能提示问题）——编译器既然知道了concept的存在，当你敲下iter，ctrl+空格的时候编译器便能够通过解析InputIterator这个concept的定义来告诉你iter对象支持哪些操作了。 

4（文档代码分离问题）——瞧一瞧for_each的声明，原来（C++98）是： 

```c++
template<typename InputIterator … >
void for_each(InputIterator iter …);
```

现在（C++09）是

```c++
concept InputIterator
{
	…
}

template<InputIterator Iter … >
void for_each(Iter …);
```

区别在什么地方？原来的代码中没有concept InputIterator这样的声明，你看着InputIterator这么个单词，得去STL的文档里面查才知道它到底有那些requirements。有了concept之后呢？只要翻开InputIterator这个concept的定义就看到了，后者将位于C++09的<iterator>头文件中。 

5（重构问题）——重构？当然！有了concept，要重构的时候只要修改concept定义，所有使用了该concept内的函数的地方都可以容器地作出改变。

6（接口适配问题）——实际上前面“事儿#6”里面提到的古董容器+非古董算法的例子虽然能说明问题，但总是不够巧妙。还不如直接抄Concept六君子在[OOPSLA ‘06上的牛paper](http://www.osl.iu.edu/publications/prints/2006/Gregor06:Concepts.pdf)中的例子，Douglas在paper里面举了一个图论库的例子：有一个矩阵算法，但另外还有一个图（Graph），就算没吃过图总见过图走路吧——图是可以用矩阵来表示的，所以只要用concept_map把图类的接口适配一下就可以拿那个现成的矩阵算法来操纵了，如下：

```c++
template<Graph G>
concept_map Matrix<G>
{
 	typedef int value_type;
  	int rows(const G& g) { return num_vertices(g); }
  	int columns(const G& g) { return num_vertices(g); }
  	double operator()(const G& g, int i, int j)
  {
	if (edge_e = find_edge(ith_vertex(i, g), ith_vertex(j, g), g))
  		return 1;
	else 
		return 0;
  }
}; 
```

7（SFINAE、traits、tag dispatching等等）——有了concept，前文的代码就可以这么写：

```c++
template<Arithmetic T>
T foo(…) { … }
```

只有对于符合Arithmetic这个concept的类型T，该函数才“存在”。 

concept还给函数重载带来了更强大的能力，比如： 

```c++
concept Initializable<typename T>
{
  void initialize();
}

template<Initializable T>
void foo(…) { … }

template<typename T>
void foo(…) { … }
```

为什么说它更强大？不信你在C++98下实现看看。

8（火星人…类型系统问题）——火星人的危险性前文已经阐述了。其危险在于将语法结构同一性视为语意同一性。用传统的接口继承就没有这个问题，因为当你继承自一个接口的时候，你明确知道你在干嘛——实现这个接口的语意。因此，在C++09的concept中，缺省的concept是“非auto”的，也就是说：

```c++
concept Drawable<typename T>
{
  void T::draw() const;
}

class MyClass
{
  void draw() const;
};
```

这种情况下MyClass是不会自动满足Drawable这个concept的（尽管它的确实现了一个一模一样的draw()函数），这是为了避免无意间实现了一个不该实现的concept。要想让MyClass实现Drawable这个concept，必须显式地说明这一点：

concept_map Drawable<MyClass> { }

是不是看上去很像模板特化？实际上concept的内部编译器实现正是利用既有的一套模板特化系统来进行的。

但是如果每个concept都要靠concept_map来支持的话太麻烦，有些基本的concept比如EqualityComparable——只要一个类型重载了operator==，那么就肯定是EqualityComparable的。这个论断几乎肯定是安全的，因为没有谁会不知道operator==的语意吧？所以，EqualityComparable是auto的：

```c++
auto concept EqualityComparable<typename T, typename U = T> {
	bool operator==(T a, U b);
	bool operator!=(T a, U b) { return !(a == b); }
}
```

这样一来如果你的类实现了operator ==，你不需要将它concept_map到EqualityComparable，就能自动(auto)实现EqualityComparable；一句话，回到原始的“结构一致性”上面。 

关于auto的另一个作用，我想到了一个绝妙的介绍，但这里空白太小写不下了，请听下回分解:-)

延伸阅读

1. [Concepts: Linguistic Support for Generic Programming in C++](http://www.osl.iu.edu/publications/prints/2006/Gregor06:Concepts.pdf)

   此篇Concepts的权威饲养指南，高屋建瓴巨细靡遗地介绍了Concept的方方面面。

2. [An Extended Comparative Study of Language Supports for Generic Programming](http://portal.acm.org/citation.cfm?id=1230756.1230757) 

   此篇对各种语言对GP的支持做了极其详尽的survey，其中也提到了concept的一些东西，很有价值的一篇paper。

3.  [Concept checking – A more abstract complement to type checking](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2003/n1510.pdf) 

   当年，C++的老豆率先发难，写了这篇最早的concept paper，其间对三大实现策略作了高屋建瓴的比较，对掌握concept的本质有非常好的帮助。

4.  [Concepts](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2003/n1510.pdf) 

   一番刀光剑影你来我往之后，user-pattern派（由Bjarne本人发起）和function signature派（由Douglas带领）终于联合起来；这是第一篇署名Bjarne Stroustrup & Douglas Gregor的Concept Proposal。Function Signature的做法被正式确定下来（主要原因之一是它提供了#6（类型适配）这个大大的好处

5. [Proposed Wording for Concepts(rev#1)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2307.pdf)

   这个就不用说了，截止到最近的concepts标准提案。

6. [Concepts for the C++0x Standard Library Utilities(rev#2)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2322.pdf)

   这个自然也不必说了，C++0x标准库里面的一些基本的concepts定义。

7. http://www.generic-programming.com 

   ConceptGCC的官方站，含GCC实现的下载，以及历届concepts相关paper。

8. [Yet Another Haskell Tutorial](http://www.cs.utah.edu/~hal/docs/daume02yaht.pdf)

   其实concept在Haskell里面早就实现了，不过名字不叫concept，而叫type class。看一看haskell的实现对理解C++09 Concepts肯定也是有帮助的。

目录(展开《C++0x漫谈》系列文章)
