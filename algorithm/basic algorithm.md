# 基础算法

## quick sort

```c++
void quick_sort(int q[], int l, int r)
{
    if (l >= r) return;
    
    int x = q[l + r >> 1], i = l - 1, j = r + 1;
    while(i < j)
    {
        do i ++; while(q[i] < x);
        do j --; while(q[j] > x);
        if (i < j) swap(q[i], q[j]);
    }
    quick_sort(q, l, j);
    quick_sort(q, j + 1, r);
}
```

## merge sort

```c++
const int N = 1e5 + 10;
int q[N], tmp[N];

void merge_sort(int q[], int l, int r)
{
    if (l >= r) return;
    int mid = l + r >> 1;
    merge_sort(q, l, mid);
    merge_sort(q, mid + 1, r);
    
    int k = 0, i = l, j = mid + 1;
    while(i <= mid && j <= r)
        if (q[i] <= q[j]) tmp[k++] = q[i++];
        else tmp[k++] = q[j++];
    
    while(i <= mid) tmp[k++] = q[i++];
    while(j <= r) tmp[k++] = q[j++];
    
    for (i = l, j = 0; i <= r; i++, j++) q[i] = tmp[j];
    
}
```

## 整数二分

```c++
// 区间[l, r] 被划分成[l, mid] 和 [mid + 1, r] 时使用
int bsearch1(int l, int r)
{
    while (l < r) {
        int mid = l + r >> 1;
        if (check(mid)) r = mid;
        else l = mid + 1;
    }
    return l;
}
// 区间[l, r] 被划分成[l, mid - 1] 和 [mid , r] 时使用
int bsearch2(int l, int r)
{
    while(l < r) {
        int mid = l + r + 1 >> 1;
        if (check(mid)) l = mid;
        else r = mid - 1;
    }
    return l;
}
```

## 前缀和

```tet
a1 a2 a3 ... an
前缀和：si = a1 + a2 + ... + ai
作用: 一次运算，多次查询区间和
```

```c++
for (int i = 1; i <= n; i++) scanf("%d", a[i]);
for (int i = 1; i <= n; i++) s[i] = s[i - 1] + a[i];
```

## 差分

```text
a1 a2 a3 ... an
构造 b1 b2 ... bn    b 序列是 a 序列的差分
使得 ai = b1 + b2 + ... + bi
作用： a[l~r]+-c   b[l] + c,  b[l+1]-c
实现：原数组可以看成[i,i]区间+a[i]执行了n次
```

```c++
void insert(int l, int r, int c)
{ // 构造差分数组
    b[l] += c;
    b[r + 1] -= c;
}

for (int i = 1; i <= n; i++) cin >> a[i];
for (int i = 1; i <= n; i++) insert(i, i, a[i]);   // 构造差分数组

while(m--) {   // m 次区间添加操作
    int l, r, c;
    scanf("%d%d%d", l , r, c);
    insert(l, r, c);
}

for (int i = 1; i <= n; i++) b[i] += b[i - 1];  // b 数组变成自己的前缀和
```

## 双指针

> 把$\mathrm{O}(\mathrm{n}^2)$的时间复杂度优化到$\mathrm{O}(\mathrm{n})$

```c++
for (int i = 0, j = 0; i < n; i++) {
    while(j < i && check(i, j)) j++;
    // 每道题的具体逻辑
}
```

## 位运算

- `lowbit(x)` 返回 `x` 的最后一位 1

  ```c++
  lowbit(x) = x & -x = x & (~n + 1)
  ```
  
- 

## 离散化

```text
原序列：1  3  100  2000 500000
离散化：0  1   2    3   4
```

两个问题：

- 原序列可能存在重复元素——去重
- 如何算出原序列 a[i] 离散化后的值——二分

```c++
vector<int> alls;  // 存储所有待离散化额值
sort(alls.begin(), alls.end());  // 将所有值排序
alls.erase(unique(alls.begin(), alls.end()), alls.end());  // 去重

// 二分查找出x对应的离散化的值
int find(int x)
{
    int l = 0, r = all.size() - 1;
    while(l < r) {  // 找到第一个大于等于x的位置
        int mid = l + 1 >> 1;
        if (alls[mid] >= x) r = mid;
        else l = mid + 1;
    }
    return r + 1；  // 映射到1, 2 ... n
}
```

## 区间合并

```text
合并有交集的区间
做法：
1. 按区间左端点排序
2. 维护一个区间集合，搜遍待合并区间
	当前区间和维护区间有三种情况：维护区间包含当前区间
	                           维护区间和当前区间有交集
	                           维护区间和当前区间没有交集
```

````c++
typedef pair<int, int> PII;
void merge(vector<PII> &segs)
{
    vector<PII> res;
    
    sort(segs.begin(), segs.end());
    
    int st = -2e9, ed = -2e9;
    for (auto seg : ses) {
        if (ed < seg.first) {
            if (st != -2e9) res.push_back({st, ed});
            st = seg.first, ed = seg.second;
        } else ed = max(ed, seg.second);
    }
    if (st != -2e9) res.push_back({st, ed});
    
    segs = res;
}
````

# 数据结构

## 链表

> 数组模拟链表

```c++
// head 表示头节点的下标
// e[i] 表示节点i的值
// ne[i] 表示节点i的next指针下标
// idx 存储当前用了哪个点
int head, e[N], ne[N], idx;

// 初始化
void init()
{
    head = -1;
    idx = 0;
}

// 将 x 插入到头节点
void add_to_head(int x)
{
    e[idx] = x;
    ne[idx] = head;
    head = idx;
    idx++;
}

// 将 x 查到下标是 k 的点的后面
void add(int k, int x)
{
    e[idx] = x;
    ne[idx] = ne[k];
    ne[k] = idx;
    idx++;
}

// 将下标是k的点后面的点删除
void remove（int k)
{
    ne[k] = ne[ne[k]];
}
```

## 栈

```c++
int stk[N], tt;

//插入
stk[++tt] = x;

// 弹出
tt--;

// 判断栈是否为空
if (tt > 0) not empty
else empty

// 栈顶
stk[tt];
```

## 队列

```c++
// 队尾插入元素，队头弹出元素
int q[N], hh, tt = -1;

// 插入元素
q[++tt] = x;

// 弹出元素
hh++;

// 判断是否为空
if (hh <= tt) not empty;
else empty;

// 取出对头元素
q[hh]
```

## 单调栈

```c++
// i 左边离i最近的且比a[i]小的数
int stk[N], tt;

for (int i = 0; i < n; i++) {
	int x;
	cin >> x;
	while(tt && stk[tt] >= x) tt--;
	if (tt) cout << stk[tt] << endl;
	else cout << -1 << endl;
	stk[++tt] = x;
}
```

## 单调队列

```c++
// 滑动窗口求最小值
int n, a[N], q[N];

int main()
{
    scanf("%d%d", &n, &k);
    for (int i = 0; i < n; i++) scanf("%d", &a[i]);
    
    int hh = 0, tt = -1;
    for (int i = 0; i < n; i++) {
        // 判断对头是否画出窗口
        if (hh <= tt && i - k + 1 > q[hh]) hh++;
        while(hh <= tt && a[q[tt]] >= a[i]) tt--;

        q[++tt] = i;
        if (i >= k - 1) printf("%d", a[q[hh]]);
    }
}
```

## kmp

> 字符串匹配算法

**暴力做法**

```text
s[n], p[n]  原串和匹配串
for (int i = 1; i <= n + 1 - j; i++) {
	bool flag = true;
	for (int j = 1; j <= m; j++) {
		if (s[i + j - 1] != p[j]) {
			flag = false;
			break;
		}
	}
}
```

```c++
int n, m;
char p[N], s[M];
int ne[N];

// 求next
for (int i = 2, j = 0; i <= n; i++) {
    while(j && p[j] != p[j + 1]) j = ne[j];
    if (p[i] == p[j + 1]) j++;
    ne[i] = j;
}

// kmp 匹配过程
for (int i = 1, j = 0; i <= m; i++) {
    while(j && s[i] != p[j + 1]) j = ne[j];
    if (s[i] == p[j + 1]) j++;
    if (j == n) {
        printf("%d", i - n);
        j = ne[j];  // 
    }
}
```

## 字典树：trie

> 字符串存储查询的数据结构

```c++
int son[N][26], cnt[N], idx;
// 0号点既是根节点，又是空节点
// son[][]存储树中每个节点的子节点
// cnt[]存储以每个节点结尾的单词数量

// 插入一个字符串
void insert(char *str)
{
    int p = 0;
    for (int i = 0; str[i]; i ++ )
    {
        int u = str[i] - 'a';
        if (!son[p][u]) son[p][u] = ++ idx;
        p = son[p][u];
    }
    cnt[p] ++ ;
}

// 查询字符串出现的次数
int query(char *str)
{
    int p = 0;
    for (int i = 0; str[i]; i ++ )
    {
        int u = str[i] - 'a';
        if (!son[p][u]) return 0;
        p = son[p][u];
    }
    return cnt[p];
}
```

## 并查集

**朴素并查集**

```c++
int p[N]; // 存储每个点的祖宗节点

// 返回x的祖宗节点
int find(int x)
{
    if (p[x] != x) p[x] = find(p[x]);
    return p[x];
}

// 初始化，假定节点的编号是1~n
for (int i = 1; i <= n; i++) p[i] = i;

// 合并a和b所在的两个集合
p[find(a)] = find(b);
```

**维护size的并查集**

```c++
int p[N], size[N];
// p[] 存储每个点的祖宗节点，size[] 只有祖宗节点的有意义，表示祖宗节点所在节点仲点的数量

// 返回x的祖宗节点
int find(int x)
{
    if (p[x] != x) p[x] = find(p[x]);
    return p[x];
}

// 初始化，假定节点编号是1~n
for (int i = 1; i <= n; i++) {
    p[i] = i;
    size[i] = 1;
}

// 合并a和b所在的两个集合
size[find(b)] += szie[find(a)];
p[find(a)] = find(b);
```

**维护到祖宗节点距离的并查集**

```c++
int p[N], d[N];
//p[]存储每个点的祖宗节点, d[x]存储x到p[x]的距离

// 返回x的祖宗节点
int find(int x)
{
if (p[x] != x)
{
int u = find(p[x]);
d[x] += d[p[x]];
p[x] = u;
}
return p[x];
}

// 初始化，假定节点编号是1~n
for (int i = 1; i <= n; i ++ )
{
p[i] = i;
d[i] = 0;
}

// 合并a和b所在的两个集合：
p[find(a)] = find(b);
d[find(a)] = distance; // 根据具体问题，初始化find(a)的偏移量
```

## 堆

```c++
// h[N]存储堆中的值, h[1]是堆顶，x的左儿子是2x, 右儿子是2x + 1
// ph[k]存储第k个插入的点在堆中的位置
// hp[k]存储堆中下标是k的点是第几个插入的
int h[N], ph[N], hp[N], size;

// 交换两个点，及其映射关系
void heap_swap(int a, int b)
{
    swap(ph[hp[a]],ph[hp[b]]);
    swap(hp[a], hp[b]);
    swap(h[a], h[b]);
}

void down(int u)
{
    int t = u;
    if (u * 2 <= size && h[u * 2] < h[t]) t = u * 2;
    if (u * 2 + 1 <= size && h[u * 2 + 1] < h[t]) t = u * 2 + 1;
    if (u != t)
    {
        heap_swap(u, t);
        down(t);
    }
}

void up(int u)
{
    while (u / 2 && h[u] < h[u / 2])
    {
        heap_swap(u, u / 2);
        u >>= 1;
    }
}

// O(n)建堆
for (int i = n / 2; i; i -- ) down(i);
```

## 一般哈希

**拉链法**

```c++
int h[N], e[N], ne[N], idx;

// 向哈希表中插入一个数
void insert(int x)
{
    int k = (x % N + N) % N;
    e[idx] = x;
    ne[idx] = h[k];
    h[k] = idx ++ ;
}

// 在哈希表中查询某个数是否存在
bool find(int x)
{
    int k = (x % N + N) % N;
    for (int i = h[k]; i != -1; i = ne[i])
        if (e[i] == x)
            return true;

    return false;
}
```

**开放寻址法**

```c++
int h[N];

// 如果x在哈希表中，返回x的下标；如果x不在哈希表中，返回x应该插入的位置
int find(int x)
{
    int t = (x % N + N) % N;
    while (h[t] != null && h[t] != x)
    {
        t ++ ;
        if (t == N) t = 0;
    }
    return t;
}
```

**字符串哈希**

> 核心思想：将字符串看成P进制数，P的经验值是131或13331，取这两个值的冲突概率低
> 小技巧：取模的数用2^64，这样直接用unsigned long long存储，溢出的结果就是取模的结果

````c++
typedef unsigned long long ULL;
ULL h[N], p[N]; // h[k]存储字符串前k个字母的哈希值, p[k]存储 P^k mod 2^64

// 初始化
p[0] = 1;
for (int i = 1; i <= n; i ++ )
{
    h[i] = h[i - 1] * P + str[i];
    p[i] = p[i - 1] * P;
}

// 计算子串 str[l ~ r] 的哈希值
ULL get(int l, int r)
{
    return h[r] - h[l - 1] * p[r - l + 1];
}
````

# C++ STL 简介

```text
vector, 变长数组，倍增的思想
    size()  返回元素个数
    empty()  返回是否为空
    clear()  清空
    front()/back()
    push_back()/pop_back()
    begin()/end()
    []
    支持比较运算，按字典序

pair<int, int>
    first, 第一个元素
    second, 第二个元素
    支持比较运算，以first为第一关键字，以second为第二关键字（字典序）

string，字符串
    size()/length()  返回字符串长度
    empty()
    clear()
    substr(起始下标，(子串长度))  返回子串
    c_str()  返回字符串所在字符数组的起始地址

queue, 队列
    size()
    empty()
    push()  向队尾插入一个元素
    front()  返回队头元素
    back()  返回队尾元素
    pop()  弹出队头元素

priority_queue, 优先队列，默认是大根堆
    size()
    empty()
    push()  插入一个元素
    top()  返回堆顶元素
    pop()  弹出堆顶元素
    定义成小根堆的方式：priority_queue<int, vector<int>, greater<int>> q;

stack, 栈
    size()
    empty()
    push()  向栈顶插入一个元素
    top()  返回栈顶元素
    pop()  弹出栈顶元素

deque, 双端队列
    size()
    empty()
    clear()
    front()/back()
    push_back()/pop_back()
    push_front()/pop_front()
    begin()/end()
    []

set, map, multiset, multimap, 基于平衡二叉树（红黑树），动态维护有序序列
    size()
    empty()
    clear()
    begin()/end()
    ++, -- 返回前驱和后继，时间复杂度 O(logn)

    set/multiset
        insert()  插入一个数
        find()  查找一个数
        count()  返回某一个数的个数
        erase()
            (1) 输入是一个数x，删除所有x   O(k + logn)
            (2) 输入一个迭代器，删除这个迭代器
        lower_bound()/upper_bound()
            lower_bound(x)  返回大于等于x的最小的数的迭代器
            upper_bound(x)  返回大于x的最小的数的迭代器
    map/multimap
        insert()  插入的数是一个pair
        erase()  输入的参数是pair或者迭代器
        find()
        []  注意multimap不支持此操作。 时间复杂度是 O(logn)
        lower_bound()/upper_bound()

unordered_set, unordered_map, unordered_multiset, unordered_multimap, 哈希表
    和上面类似，增删改查的时间复杂度是 O(1)
    不支持 lower_bound()/upper_bound()， 迭代器的++，--

bitset, 圧位
    bitset<10000> s;
    ~, &, |, ^
    >>, <<
    ==, !=
    []

    count()  返回有多少个1

    any()  判断是否至少有一个1
    none()  判断是否全为0

    set()  把所有位置成1
    set(k, v)  将第k位变成v
    reset()  把所有位变成0
    flip()  等价于~
    flip(k) 把第k位取反
```



