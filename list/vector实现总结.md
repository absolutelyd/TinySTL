## 1.vector容器变量

1. 三个迭代器：

   ```cpp
   iterator begin_;  // 表示目前使用空间的头部
   iterator end_;    // 表示目前使用空间的尾部
   iterator cap_;    // 表示目前储存空间的尾部
   ```



## 2.vector构造函数

### 1.构造阶段使用的函数

#### 1.try_init

没有长度要求时使用，默认长度为16。

```cpp
    // try_init 函数，若分配失败则忽略，不抛出异常
	// 在不要求长度时，会初始化16的空间
    template <class T>
    void vector<T>::try_init() noexcept{
        try{
            begin_ = data_allocator::allocate(16);
            end_ = begin_;
            cap_ = begin_ + 16;
        }
        catch (...){
            begin_ = nullptr;
            end_ = nullptr;
            cap_ = nullptr;
        }
    }
```

#### 2.init_space

初始化给定size的vector，但是不填入初值。

```cpp
    template <class T>
    void vector<T>::init_space(size_type size, size_type cap){
        try{
            begin_ = data_allocator::allocate(cap);
            end_ = begin_ + size;
            cap_ = begin_ + cap;
        }
        catch (...){
            begin_ = nullptr;
            end_ = nullptr;
            cap_ = nullptr;
            throw;
        }
    }
```

#### 3.fill_init

初始化给定大小的vector，并且填入初值value。

注意初始容量最小也是16。

```cpp
	template <class T>
    void vector<T>::
        fill_init(size_type n, const value_type& value) {
        const size_type init_size = mystl::max(static_cast<size_type>(16), n); // 最小空间为16
        init_space(n, init_size);
        mystl::uninitialized_fill_n(begin_, n, value);
        // 最终调用的是memset
    }
```

#### 4.range_init

用于迭代器初始化，可以使用另外一个vector的两个迭代器，将vector初始化成迭代器之间的数据。

同样，初始尺寸最小为16。

求迭代器差值就是了STL迭代器的difference_type的差值，这一迭代器型别需要traits获取。

```cpp
    template <class T>
    template <class Iter>
    void vector<T>::
        range_init(Iter first, Iter last){
        const size_type len = mystl::distance(first, last);
        const size_type init_size = mystl::max(len, static_cast<size_type>(16));
        init_space(len, init_size);
        mystl::uninitialized_copy(first, last, begin_);
    }
```

### 2.构造函数

构造阶段就是使用上面的四个函数。

```cpp
// 给定尺寸初始化，会填入value_type()函数返回的默认值。
// value_type也是迭代器的一个型别
// 所以vector<int> 就是初值为0。
explicit vector(size_type n)
        {
            fill_init(n, value_type());
        }

// 给定初值和大小初始化
        vector(size_type n, const value_type& value)
        {
            fill_init(n, value);
        }

// 给定迭代器初始化
        template <class Iter, typename std::enable_if<
            mystl::is_input_iterator<Iter>::value, int>::type = 0>
        vector(Iter first, Iter last)
        {
            MYSTL_DEBUG(!(last < first));
            range_init(first, last);
        }

// 拷贝构造，但还是调用新vector的两个迭代器，使用上面的迭代器初始化
        vector(const vector& rhs)
        {
            range_init(rhs.begin_, rhs.end_);
        }

// 移动构造，传入的是右值引用。
// 右值引用一般用于即将要销毁的对象
// 这里直接将rhs的三个指针交给vector，销毁rhs即可。
        vector(vector&& rhs) noexcept
            :begin_(rhs.begin_),
            end_(rhs.end_),
            cap_(rhs.cap_)
        {
            rhs.begin_ = nullptr;
            rhs.end_ = nullptr;
            rhs.cap_ = nullptr;
        }

// initializer_list 就是 {}中的数据，支持用{}初始化
        vector(std::initializer_list<value_type> ilist)
        {
            range_init(ilist.begin(), ilist.end());
        }
```

### 3.赋值运算符

#### 1.复制赋值

- 要点一：空间大小不同时的处理。

- 要点二：uninitialized_copy意思是未初始化时的拷贝，在空间可能不足时使用。

  copy直接调用=，不会负责空间初始化的问题，uninitialized_copy会调用拷贝构造函数。

```cpp
    // 复制赋值操作符
    template <class T>
    vector<T>& vector<T>::operator=(const vector& rhs){
        if (this != &rhs){
            const auto len = rhs.size();
            if (len > capacity()){ // 原空间不够大，就赋值到一个新vector上，再把新vector换过来。
                vector tmp(rhs.begin(), rhs.end());
                swap(tmp);
            }
            else if (size() >= len){ // 如果原空间比新空间大一些，就把多出来的尾巴destroy掉
                auto i = mystl::copy(rhs.begin(), rhs.end(), begin());
                data_allocator::destroy(i, end_);
                end_ = begin_ + len;
            }
            else { // 如果原空间更小，就分两部分拷贝过去，注意后面这部分可能会有空间不足的问题。
                mystl::copy(rhs.begin(), rhs.begin() + size(), begin_);
                mystl::uninitialized_copy(rhs.begin() + size(), rhs.end(), end_);
                cap_ = end_ = begin_ + len;
            }
        }
        return *this;
    }
```

#### 2.移动赋值：

```cpp
    // 移动赋值操作符
    template <class T>
    vector<T>& vector<T>::operator=(vector&& rhs) noexcept
    {
        destroy_and_recover(begin_, end_, cap_ - begin_);
        begin_ = rhs.begin_;
        end_ = rhs.end_;
        cap_ = rhs.cap_;
        rhs.begin_ = nullptr;
        rhs.end_ = nullptr;
        rhs.cap_ = nullptr;
        return *this;
    }
```

#### 3.初始化列表赋值

```cpp
        vector& operator=(std::initializer_list<value_type> ilist)
        {
            vector tmp(ilist.begin(), ilist.end());
            swap(tmp);
            return *this;
        }
```

### 4.容量相关函数

此处的size()等简单函数不写。

#### 1.reserve()预留空间

reserve可以直接预留较大空间，避免重复的拓展空间开销。

```cpp
// 预留空间大小，当原容量小于要求大小时，才会重新分配
template <class T>
void vector<T>::reserve(size_type n){
  if (capacity() < n){
    THROW_LENGTH_ERROR_IF(n > max_size(),
                          "n can not larger than max_size() in vector<T>::reserve(n)");
    const auto old_size = size();
    auto tmp = data_allocator::allocate(n);
    mystl::uninitialized_move(begin_, end_, tmp);
    data_allocator::deallocate(begin_, cap_ - begin_);
    begin_ = tmp;
    end_ = tmp + old_size;
    cap_ = begin_ + n;
  }
}
```

#### 2.shrink_to_fit 要求函数退回空间

```cpp
    // 放弃多余的容量
    template <class T>
    void vector<T>::shrink_to_fit(){
        if (end_ < cap_){
            reinsert(size()); // reinsert中移除掉多余空间
        }
    }
```

### 5.插入操作

#### 1.emplace

- emplace：在pos前面插入元素。

- emplace的运行效率比insert更高：

  insert()向vector插入类对象，需要调用类的构造函数和拷贝构造函数。

  而emplace只需要调用构造函数。

- 但是需要注意的是，emplace这种高效仅仅在传入构造参数时存在。

  可以参考：https://zhuanlan.zhihu.com/p/183861524

  如果传入的是对象实例、临时变量，那么都一定会调用到拷贝构造，效率一样。

  push_back和emplace_back同理。

```cpp
    // 在 pos 位置就地构造元素，避免额外的复制或移动开销
    template <class T>
    template <class ...Args>
    typename vector<T>::iterator
        vector<T>::emplace(const_iterator pos, Args&& ...args){
        MYSTL_DEBUG(pos >= begin() && pos <= end());
        iterator xpos = const_cast<iterator>(pos);
        const size_type n = xpos - begin_;
        if (end_ != cap_ && xpos == end_){ // 在end前插入，就是尾部插入
            data_allocator::construct(mystl::address_of(*end_), mystl::forward<Args>(args)...);
            ++end_;
        }
        else if (end_ != cap_){
            auto new_end = end_;
            data_allocator::construct(mystl::address_of(*end_), *(end_ - 1)); // 多创建一个尾部元素
            ++new_end;
            mystl::copy_backward(xpos, end_ - 1, end_);
            //pos之后的元素整体后移一位
            *xpos = value_type(mystl::forward<Args>(args)...);
            end_ = new_end;
        }
        else{
            reallocate_emplace(xpos, mystl::forward<Args>(args)...);
        }
        return begin() + n;
    }
```

#### 2.insert

insert的操作是先构造完成对象，完成了pos后面元素的移动后，再调用拷贝构造函数放到pos处。

- insert效率不如emplace

```cpp
    // 在 pos 处插入元素
    template <class T>
    typename vector<T>::iterator
        vector<T>::insert(const_iterator pos, const value_type& value)
    {
        MYSTL_DEBUG(pos >= begin() && pos <= end());
        iterator xpos = const_cast<iterator>(pos);
        const size_type n = pos - begin_;
        if (end_ != cap_ && xpos == end_)
        {
            data_allocator::construct(mystl::address_of(*end_), value);
            ++end_;
        }
        else if (end_ != cap_)
        {
            auto new_end = end_;
            data_allocator::construct(mystl::address_of(*end_), *(end_ - 1));
            ++new_end;
            auto value_copy = value;  // 避免元素因以下复制操作而被改变
            mystl::copy_backward(xpos, end_ - 1, end_);
            *xpos = mystl::move(value_copy);
            end_ = new_end;
        }
        else
        {
            reallocate_insert(xpos, value);
        }
        return begin_ + n;
    }
```

#### 3.emplace_back：

```cpp
    // 在尾部就地构造元素，避免额外的复制或移动开销
    template <class T>
    template <class ...Args>
    void vector<T>::emplace_back(Args&& ...args){
        if (end_ < cap_){
            data_allocator::construct(mystl::address_of(*end_), mystl::forward<Args>(args)...);
            ++end_;
        }
        else{
            reallocate_emplace(end_, mystl::forward<Args>(args)...);
        }
    }
```

#### 4.push_back

- 要点一：push_back效率比emplace_back要低，原因和insert一样，push_back是先完成了对象的构造，再拷贝到对应位置。但是注意使用情景。
- push_back的扩容机制：二倍扩容。但是MSVC是1.5倍扩容。事实证明二倍扩容更好。

push_back()将新元素插入尾端时，首先检查还有没有备用空间，如果有就直接构造，如果没有备用空间，就扩充空间（重新配置、移动数据、释放原空间）。

```cpp
void push_back(const T& x) {
    if (finish != end_ofstorage) {
        construct(finish, x); // 全局函数
        ++finish;
    }else {
        insert_aux(end(), x);
        //如果空间不够，在insert_aux函数里进行扩容操作。
    }
}
```

insert_aux：

```cpp
// 需要注意的是，并不仅仅是push_back使用insert_aux，insert()也会使用到这个函数。
// push_back()其实就是这样使用了insert_aux(finish, x), 特定位置的insert()。
template <class T, class Alloc>
void vector<T, Alloc>::insert_aux(iterator position, const T& x) {
    if (finish != end_of_storage) { //还有备用空间
        consturct(finish, *(finish - 1));
        // 多制造一个原本的尾部元素
        // 在备用空间起始处构造一个元素，并以vector最后一个元素作为初值。
        ++finish;
        T x_copy = x;
        copy_backward(position, finish - 2, finish - 1);
        // copy_backward(first, end, target_first);
        // first、end是拷贝内容的开始和结束。
        // target_first是拷贝目标的起始位置。
        // 其实就是copy函数，区别就是他是从target_first倒着开始拷贝的。
        // 这里的功能就是把position 到 finish-1 的所有元素后移一位，并且第一步多出来的最后一位元素移到了finish之后，丢掉了。
        // 特殊的是，如果是insert到尾部，此时position和finish-1的位置重合，什么都不会发生
        *position = x_copy;
    } else { // 没有备用空间
        const size_type old_size = size();
        const size_type len = old_size != 0 ? 2 * old_size : 1;
        // 原大小为0，新大小为1；
       	// 否则新大小为原大小的两倍；
        // 前半段用来放原数据，后半段放新数据
        
        iterator new_start = data_allocator::allocate(len); // 申请新大小的空间
        iterator new_finish = new_start;
        try {
            // 原来的从start -> position 的内容都拷贝过来
            new_finish = uninitialized_copy(start, position, new_start);
            // 在插入位置position，插入x
            construct(new_finish, x);
            ++new_finish;
            // 将原来position -> finish 的内容也拷贝来
            new_finish = uninitialized_copy(position, finish, new_finish);
            // 同样需要注意，不仅仅是push_back会用到这个函数，insert也会用到，这个函数还要保证insert的正确性。
        } catch(...) {
            destroy(new_start, new_finish);
            data_allocator::deallocate(new_start, len);
            throw;
        }
        
        // 析构并释放原vector
        destroy(begin(), end());
        deallocate(); // deallocate释放对象所占的内存
        
        //调整迭代器，指向新的vector
        start = new_start;
        finish = new_finish;
        end_of_storage = new_start + len;
    }
}
```

### 6.元素访问操作

1.at和[]运算符

这两个运算符应该返回引用，因为还要修改这个位置的数据。

```cpp
        // 访问元素相关操作
        reference operator[](size_type n)
        {
            MYSTL_DEBUG(n < size());
            return *(begin_ + n);
        }
        const_reference operator[](size_type n) const
        {
            MYSTL_DEBUG(n < size());
            return *(begin_ + n);
        }
        reference at(size_type n)
        {
            THROW_OUT_OF_RANGE_IF(!(n < size()), "vector<T>::at() subscript out of range");
            return (*this)[n];
        }
        const_reference at(size_type n) const
        {
            THROW_OUT_OF_RANGE_IF(!(n < size()), "vector<T>::at() subscript out of range");
            return (*this)[n];
        }
```

