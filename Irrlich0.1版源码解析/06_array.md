# array



具有附加功能的自分配模板数组（如std  vector）.一些特性是： 堆排序，二分查找方法，更容易调试



```c++
template <class T>
class array
{

public:

	array()
		: data(0), used(0), allocated(0),
			free_when_destroyed(true), is_sorted(true)
	{}

	//! 构造一个数组并分配一个初始的内存块。
	//! \param start_count: 需要分配内存的元素的数量.
	array(unsigned int start_count)
		: data(0), used(0), allocated(0),
			free_when_destroyed(true),	is_sorted(true)
	{
		reallocate(start_count);
	}

	//! Copy constructor
	array(const array<T>& other)
		: data(0)
	{
		*this = other;
	}

	//! Destructor. Frees allocated memory, if set_free_when_destroyed
	//! was not set to false by the user before.
	~array()
	{
		if (free_when_destroyed)
			delete [] data;
	}

	//!重新分配数组内存，让他更大或者更小.
	//! \param new_size: 数组新的大小.
	void reallocate(unsigned int new_size)
	{
		T* old_data = data;

		data = new T[new_size];
		allocated = new_size;
		
		int end = used < new_size ? used : new_size;
		for (int i=0; i<end; ++i)
			data[i] = old_data[i];

		if (allocated < used)
			used = allocated;
		
		delete [] old_data;
	}

	//! 在数组后面添加一个元素。 如果数组太小，无法添加此新元素，则将数组变大。
	//! \param element: 添加在数组后面的元素
	void push_back(const T& element)
	{
		if (used + 1 > allocated)
			reallocate(used * 2 +1);

		data[used++] = element;
		is_sorted = false;
	}

	//! 清空数组，删除所有分配的内存
	void clear()
	{
		delete [] data;
		data = 0;
		used = 0;
		allocated = 0;
		is_sorted = true;
	}

	//! 将指针设置为新数组，并将其用作新工作区。
	//! \param newPointer: 指向新数组元素的指针.
	//! \param size: 新数组的大小.
	void set_pointer(T* newPointer, unsigned int size)
	{
		delete [] data;
		data = newPointer;
		allocated = size;
		used = size;
		is_sorted = false;
	}



	//! 设置数组是否应删除其使用的内存。
	//! \param f:如果为true，则数组在其析构函数中释放分配的内存，否则不释放。 默认值为true。
	void set_free_when_destroyed(bool f)
	{
		free_when_destroyed = f;
	}

	//! 设置数组的大小
	//! \param usedNow:正在使用的元素的大小.
	void set_used(unsigned int usedNow)
	{
		if (allocated < usedNow)
			reallocate(usedNow);

		used = usedNow;
	}

	//! Assignement operator
	void operator=(const array<T>& other)
	{
		if (data)
			delete [] data;

		//if (allocated < other.allocated)
		data = new T[other.allocated];

		used = other.used;
		free_when_destroyed = other.free_when_destroyed;
		is_sorted = other.is_sorted;
		allocated = other.allocated;

		for (u32 i=0; i<other.used; ++i)
			data[i] = other.data[i];
	}

	//! Direct access operator
	T& operator [](u32 index)
	{
		#ifdef _DEBUG
		if (index>=used)
			_asm int 3 // access violation
		#endif

		return data[index];
	}

	//! Direct access operator
	const T& operator [](u32 index) const
	{
		#ifdef _DEBUG
		if (index>=used)
			_asm int 3 // access violation
		#endif

		return data[index];
	}

	//! 返回指向数组的指针
	T* pointer()
	{
		return data;
	}

	//! 返回指向数组的const 指针
	const T* const_pointer() const
	{
		return data;
	}



	//! 返回已使用数组的大小
	unsigned int size() const
	{
		return used;
	}

	//! 返回分配内存的大小
	unsigned int allocated_size() const
	{
		return allocated;
	}

	//! Returns true if array is empty
	//! \return True if the array is empty, false if not.
	bool empty() const
	{
		return used == 0;
	}



	//! Sorts the array using heapsort. There is no additional memory waste and
	//! the algorithm performs (O) n log n in worst case.
	void sort()
	{
		if (is_sorted || used<2)
			return;

		heapsort(data, used);
		is_sorted = true;
	}

	//! Performs a binary search for an element, returns -1 if not found.
	//! The array will be sorted before the binary search if it is not
	//! already sorted.
	//! \param element: Element to search for.
	//! \return Returns position of the searched element if it was found,
	//! otherwise -1 is returned.
	unsigned int binary_search(const T& element)
	{
		return binary_search(element, 0, used-1);
	}

	//! Performs a binary search for an element, returns -1 if not found.
	//! The array will be sorted before the binary search if it is not
	//! already sorted.
	//! \param element: Element to search for.
	//! \param left: First left index
	//! \param right: Last right index.
	//! \return Returns position of the searched element if it was found,
	//! otherwise -1 is returned.
	unsigned int binary_search(const T& element, s32 left, s32 right)
	{
		if (!used) return -1;
		sort();
		unsigned int m;
		do
		{
			m = (left+right)>>1;

			if (element < data[m])
				right = m - 1;
			else
				left = m + 1;

		} while((element < data[m] || data[m] < element) && left<=right);

		// this last line equals to:
		// " while((element != array[m]) && left<=right);"
		// but we only want to use the '<' operator.
		// the same in next line, it is "(element == array[m])"

		if (!(element < data[m]) && !(data[m] < element))
			return m;
		return -1;
	}

	//! 用后面的元素覆盖前面的元素进行删除，会很慢
	void erase(unsigned int index)
	{
		#ifdef _DEBUG
		if (index>=used || index<0)
			_asm int 3 // access violation
		#endif

		for (unsigned int i=index+1; i<used; ++i)
			data[i-1] = data[i];

		--used;
	}

	//! Erases some elements from the array. may be slow, because all elements 
	//! following after the erased element have to be copied.
	//! \param index: Index of the first element to be erased.
	//! \param count: Amount of elements to be erased.
	void erase(unsigned int index, int count)
	{
		#ifdef _DEBUG
		if (index>=used || index<0 || count<1 || index+count>used)
			_asm int 3 // access violation
		#endif

		for (unsigned int i=index+count; i<used; ++i)
			data[i-count] = data[i];

		used-= count;
	}

			
	private:

		T* data;
		unsigned int allocated;
		unsigned int used;
		bool free_when_destroyed;
		bool is_sorted;
};
```



