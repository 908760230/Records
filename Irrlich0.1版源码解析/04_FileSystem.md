# FileSystem



​	这个 FileSystem 管理文件和存档，并提供对它们的访问。

​	它管理文件的位置，因此使用IO的模块无需知道每个文件的位置。 文件可以位于.zip归档文件中，也可以位于磁盘上的文件中，使用IFileSystem对此没有任何影响。



IFileSystem 定义了接口







## IFileSystem

```c++
#include "IUnknown.h"

class IFileSystem : public IUnknown
{
public:

	//! destructor
	virtual ~IFileSystem() {};

	//! 打开文件进行读取访问。
	//! \param filename: 打开文件的名字
	//! \return 返回被创建文件接口的指针
	//! 当不需要的时候，指针会被丢弃
	virtual IReadFile* createAndOpenFile(const char* filename) = 0;

	//! 添加一个 zip 归档文件到 FileSystem
	//! \param filename: zip压缩包的名字
	//! \param ignoreCase: 如果设置为 true, 无需在适当的情况下写所有字母，就可以访问档案中的文件。
	//! \param ignorePaths: 如果设置为 true, 无需完整路径即可访问添加的存档中的文件。
	//! \return 如果成功添加了存档，则返回true，否则返回false。
	virtual bool addZipFileArchive(const char* filename, bool ignoreCase = true, bool ignorePaths = true) = 0;
	
	//! 返回当前工作目录的字符串。
	virtual const char* getWorkingDirectory() = 0;

	//! 将当前工作目录更改为所给的字符串。
	virtual bool changeWorkingDirectoryTo(const char* newDirectory) = 0;

	//! 在当前工作目录中创建文件和目录的列表，并将其返回。
	virtual IFileList* createFileList() = 0;

	//! 确定文件是否存在并且是否可以打开。
	virtual bool existFile(const char* filename) = 0;
};
```



## CFileSystem



```c++
#include "IFileSystem.h"
#include "array.h"

class CZipReader;
const int FILE_SYSTEM_MAX_PATH = 1024;

//使用普通文件和一个zip文件的FileSystem
class CFileSystem : public IFileSystem
{
public:

	//! constructor
	CFileSystem();

	//! destructor
	virtual ~CFileSystem();

	//! opens a file for read access
	virtual IReadFile* createAndOpenFile(const char* filename);

	//! adds an zip archive to the filesystem
	virtual bool addZipFileArchive(const char* filename, bool ignoreCase = true, bool ignorePaths = true);

	//! Returns the string of the current working directory
	virtual const char* getWorkingDirectory();

	//! Changes the current Working Directory to the string given.
	//! The string is operating system dependent. Under Windows it will look
	//! like this: "drive:\directory\sudirectory\"
	virtual bool changeWorkingDirectoryTo(const char* newDirectory);

	//! Creates a list of files and directories in the current working directory 
	//! and returns it.
	virtual IFileList* createFileList();

	//! determinates if a file exists and would be able to be opened.
	virtual bool existFile(const char* filename);

private:

	array<CZipReader*> ZipFileSystems;
	char WorkingDirectory[FILE_SYSTEM_MAX_PATH];
};

```



```c++
#include "CFileSystem.h"
#include "IReadFile.h"
#include "CZipReader.h"
#include "CFileList.h"
#include "stdio.h"
#include "os.h"

#ifdef WIN32
#include <direct.h> // for _chdir
#endif


//! constructor
CFileSystem::CFileSystem()
{
	#ifdef _DEBUG
	setDebugName("CKFileSystem");
	#endif
}



//! destructor
CFileSystem::~CFileSystem()
{
	for (u32 i=0; i<ZipFileSystems.size(); ++i)
		ZipFileSystems[i]->drop();
}



//! opens a file for read access
IReadFile* CFileSystem::createAndOpenFile(const char* filename)
{
	IReadFile* file = 0;

	for (int i=0; i<ZipFileSystems.size(); ++i)
	{
		file = ZipFileSystems[i]->openFile(filename);
		if (file)
			return file;
	}

	file = createReadFile(filename);
	return file;
}



//! adds an zip archive to the filesystem
bool CFileSystem::addZipFileArchive(const char* filename, bool ignoreCase, bool ignorePaths)
{
	IReadFile* file = createReadFile(filename);
	if (file)
	{
		CZipReader* zr = new CZipReader(file, ignoreCase, ignorePaths);
		if (zr)
			ZipFileSystems.push_back(zr);

		file->drop();
		return zr != 0;
	}

	#ifdef _DEBUG
	os::Warning::print("Could not open file. Zipfile not added", filename);
	#endif
	return false;
}



//! Returns the string of the current working directory
const c8* CFileSystem::getWorkingDirectory()
{
#ifdef WIN32
	_getcwd(WorkingDirectory, FILE_SYSTEM_MAX_PATH);
	return WorkingDirectory;
#endif

	return 0;
}



//! Changes the current Working Directory to the string given.
//! The string is operating system dependent. Under Windows it will look
//! like this: "drive:\directory\sudirectory\"
//! \return
//! Returns true if successful, otherwise false.
bool CFileSystem::changeWorkingDirectoryTo(const char* newDirectory)
{
#ifdef WIN32
	return (_chdir(newDirectory) == 0);
#endif
	return false;
}


//! Creates a list of files and directories in the current working directory 
IFileList* CFileSystem::createFileList()
{
	return new CFileList();
}


//! determinates if a file exists and would be able to be opened.
bool CFileSystem::existFile(const char* filename)
{
	for (u32 i=0; i<ZipFileSystems.size(); ++i)
		if (ZipFileSystems[i]->findFile(filename)!=-1)
			return true;

	FILE* f = fopen(filename, "rb");
	if (f) 
	{
		fclose(f);
		return true;
	}

	return false;
}



//! creates a filesystem which is able to open files from the ordinary file system,
//! and out of zipfiles, which are able to be added to the filesystem.
IFileSystem* createFileSystem()
{
	return new CFileSystem();
}
```

