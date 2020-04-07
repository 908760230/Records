# IUnKnown



  **是很多类的基类，内部实现引用计数机制，对于计数器为0的对象会被自动删除，同时其子类继承该机制**。

![](https://github.com/908760230/Records/blob/master/Irrlich0.1%E7%89%88%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/images/iunknow.png)



## 代码

```c++
	class IUnknown
	{
	public:

		//! Constructor.
		IUnknown()
			: ReferenceCounter(1), DebugName(0)
		{}

		//! Destructor.
		virtual ~IUnknown(){}
        
		//引用计数+1
		void grab() { ++ReferenceCounter; }

		//引用计数器自减一，如果计数器为0则删除物体
		bool drop(){
			--ReferenceCounter;
			if (!ReferenceCounter){
				delete this;
				return true;
			}
			return false;
		}

		//! 返回物体的调试名称，并且只能通过它自身进行设置和修改
		//! 该方法仅调试模式可用
		const char* getDebugName() const{
			return DebugName;
		} 

	protected:
		//设置调试名称
        //仅调试模式可用
		void setDebugName(const char* newName){
			DebugName = newName;
		}

	private:

		int	ReferenceCounter;			//引用计数器
		const char* DebugName;			//调试名称
	};
```

