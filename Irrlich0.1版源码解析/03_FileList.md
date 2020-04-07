# FileList



FileList 列出了目录下所有的文件



IFileList  基类定义了主要的接口

CFileList 为子类实现



## IFileList

```c++
#include "IUnknown.h"

class IFileList : public IUnknown
{
public:

	//! destructor
	virtual ~IFileList() {};

	//! 返回FileList中文件的数目
	virtual int getFileCount() = 0;

	//! 基于索引返回文件的名字.
	//! 如果发生错误则返回0
	virtual const char* getFileName(int index) = 0;

	//! 判断文件是否为目录
	//!  如果错误发生则结果是未定义的
	virtual bool isDirectory(int index) = 0;
};
```



## CFileList



```c++
// CFileList.h

#include "IFileList.h"
#include "irrstring.h"
#include "array.h"

class CFileList : public IFileList
{
public:

	//! constructor
	CFileList();

	//! destructor
	virtual ~CFileList();

	//! 返回FileList中文件的数目
	virtual int getFileCount();

	//! 基于索引返回文件的名字.
	//! 如果发生错误则返回0
	virtual const char* getFileName(int index);

	//! 判断文件是否为目录
	//!  如果错误发生则结果是未定义的
	virtual bool isDirectory(int index);

private:
	
    // 文件实体
	struct FileEntry
	{
		core::stringc Name;
		int Size;
		bool isDirectory;
	};

	core::array< FileEntry > Files;
};
```



```c++
// CFileList.cpp

#include "CFileList.h"

// TODO: Code a Linux version

#ifdef WIN32
#include <stdio.h>
#include <io.h>
#endif

CFileList::CFileList()
{
	#ifdef WIN32
    //存储文件各种信息的结构体
	struct _finddata_t c_file;
	int hFile;
	FileEntry entry;
	
    //查找当前目录的所有文件
	if( (hFile = _findfirst( "*", &c_file )) != -1L )
	{
		do
		{
			entry.Name = c_file.name;
			entry.Size = c_file.size;
			entry.isDirectory = (_A_SUBDIR & c_file.attrib) != 0;
			Files.push_back(entry);
		}
		while( _findnext( hFile, &c_file ) == 0 );

		_findclose( hFile );
	}

	//TODO add drives
	//entry.Name = "E:\\";
	//entry.isDirectory = true;
	//Files.push_back(entry);
	#endif

}


CFileList::~CFileList()
{
}


int CFileList::getFileCount()
{
	return Files.size();
}


const char* CFileList::getFileName(int index)
{
	if (index < 0 || index > (int)Files.size())
		return 0;

	return Files[index].Name.c_str();
}


bool CFileList::isDirectory(int index)
{
	if (index < 0 || index > (int)Files.size())
		return false;

	return Files[index].isDirectory;
}
```

