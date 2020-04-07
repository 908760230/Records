# ReadFile



***IReadFile***         提供对文件的读取访问权限的接口。基类为 IUnkonwn

***CReadFile***         对父类 ***IReadFile*** 的实现，从磁盘读取文件





## ***IReadFile***         

```c++
//  IReadFile.h

class IReadFile : public IUnknown
	{
	public:

		virtual ~IReadFile() {};

		// 从文件中读取指定大小字节
		virtual size_t read(void* buffer, size_t sizeToRead) = 0;

		// 改变文件中的位置，如果成功则返回true
		// 如果 relativeMovement=true, 那么改变的位置为相对于目前pos的相对位置,
		// 否则从文件开始处执行
		virtual bool seek(int finalPos, bool relativeMovement = false) = 0;

		// 返回文件大小
		virtual size_t getSize() = 0;

		// 返回目前在文件中的读写位置
		virtual int getPos() = 0;

		// 返回文件的名字
		virtual const char* getFileName() = 0;
	};

	//! Internal function, please do not use.
	IReadFile* createReadFile(const char* fileName);
	//! Internal function, please do not use.
	IReadFile* createLimitReadFile(const char* fileName, IReadFile* alreadyOpenedFile, size_t areaSize);
	//! Internal function, please do not use.
	IReadFile* createMemoryReadFile(void* memory, int size, const char* fileName, bool deleteMemoryWhenDropped);
```





## CReadFile



```c++
	//CReadFile.h
#include <stdio.H>
#include "IReadFile.h"
#include "irrstring.h"

	class CReadFile : public IReadFile
	{
	public:

		CReadFile(const wchar_t* fileName);
		CReadFile(const char* fileName);

		virtual ~CReadFile();

		//  返回读取字节的数目
		virtual size_t read(void* buffer, size_t sizeToRead);

		// 改变文件中的位置，如果成功则返回true
		// 如果 relativeMovement=true, 那么改变的位置为相对于目前pos的相对位置,
		// 否则从文件开始处执行
		virtual bool seek(int finalPos, bool relativeMovement = false);

		// 返回文件大小
		virtual size_t getSize();

		// 返回文件是否被打开
		bool isOpen();

		// 返回目前在文件中的读写位置
		virtual int getPos();

		// 返回文件的名字
		virtual const char* getFileName();

	private:

		// 打开文件
		void openFile();	

		core::stringc Filename;
		FILE* File;
		size_t FileSize;
	};
```



```c++
//CReadFile.cpp
#include "CReadFile.h"
#include <stdio.h>


CReadFile::CReadFile(const c8* fileName)
: FileSize(0)
{
	#ifdef _DEBUG
	setDebugName("CReadFile");
	#endif

	Filename = fileName;
	openFile();
}



CReadFile::~CReadFile()
{
	if (File) fclose(File);
}



//! returns if file is open
inline bool CReadFile::isOpen()
{
	return File != 0;
}



//! returns how much was read
size_t CReadFile::read(void* buffer, size_t sizeToRead)
{
	if (!isOpen())
		return 0;

	return fread(buffer, 1, sizeToRead, File);
}



//! changes position in file, returns true if successful
//! if relativeMovement==true, the pos is changed relative to current pos,
//! otherwise from begin of file
bool CReadFile::seek(int finalPos, bool relativeMovement)
{
	if (!isOpen())
		return false;

	return fseek(File, finalPos, relativeMovement ? SEEK_CUR : SEEK_SET) == 0;
}



//! returns size of file
size_t CReadFile::getSize()
{
	return FileSize;
}



//! returns where in the file we are.
int CReadFile::getPos()
{
	return ftell(File);
}



//! opens the file
void CReadFile::openFile()
{
	File = fopen(Filename.c_str(), "rb");

	if (File)
	{
		// get FileSize

		fseek(File, 0, SEEK_END);
		FileSize = ftell(File);
		fseek(File, 0, SEEK_SET);
	}
}



//! returns name of file
const char* CReadFile::getFileName()
{
	return Filename.c_str();
}



IReadFile* createReadFile(const char* fileName)
{
	CReadFile* file = new CReadFile(fileName);
	if (file->isOpen())
		return file;

	file->drop();
	return 0;
}

```

