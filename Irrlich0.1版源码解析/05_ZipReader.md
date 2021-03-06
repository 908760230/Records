# ZipReader





读取Zip文件



```c++
#include "IUnknown.h"
#include "IReadFile.h"
#include "array.h"
#include "irrstring.h"

const s16 ZIP_FILE_ENCRYPTED =			0x0001; // 设置文件是否加密
const s16 ZIP_INFO_IN_DATA_DESCRITOR =	0x0008; // CRC-32校验码, 本地头中的压缩大小和未压缩大小设置为零

struct SZIPFileDataDescriptor
{
	int CRC32;
	int CompressedSize;
	int UncompressedSize;
} PACK_STRUCT;

struct SZIPFileHeader
{
	int Sig;
	short VersionToExtract;
	short GeneralBitFlag;
	short CompressionMethod;
	short LastModFileTime;
	short LastModFileDate;
	SZIPFileDataDescriptor DataDescriptor;
	short FilenameLength;
	short ExtraFieldLength;
} PACK_STRUCT;

struct SZipFileEntry
{
	core::stringc zipFileName;
	core::stringc simpleFileName;
	core::stringc path;
	int fileDataPosition; // 在文件中解压数据的位置
	SZIPFileHeader header;

	bool operator < (const SZipFileEntry& other) const
	{
		return simpleFileName < other.simpleFileName;
	}


	bool operator == (const SZipFileEntry& other) const
	{
		return simpleFileName == other.simpleFileName;
	}
};

//不解压缩数据，仅读取文件并能够打开未压缩的条目。
class CZipReader : public IUnknown
	{
	public:

		CZipReader(IReadFile* file, bool ignoreCase, bool ignorePaths);
		virtual ~CZipReader();

		//! 用过文件名打开文件
		virtual IReadFile* openFile(const char* filename);

		//! 通过索引打开文件
		IReadFile* openFile(int index);

		//! returns 包中文件的数量
		int getFileCount();

		//! returns 文件的数据
		const SZipFileEntry* getFileInfo(int index) const;

		//! returns 文件索引
		int findFile(const char* filename);

	private:
		
		//! 扫描本地头文件，如果没有更多本地文件头文件，则返回false。
		bool scanLocalHeader();

		//! 将zip文件中的文件名拆分为有用的文件名和路径
		void extractFilename(SZipFileEntry* entry);

		//! 删除文件名中的路劲
		void deletePathFromFilename(core::stringc& filename);

		IReadFile* File;

		core::array<SZipFileEntry> FileList;

		bool IgnoreCase;
		bool IgnorePaths;
	};
```





```c++
#include <string.h>
#include "CZipReader.h"
#include "os.h"
#include "zlib\zlib.h"


CZipReader::CZipReader(IReadFile* file, bool ignoreCase, bool ignorePaths)
: File(file), IgnoreCase(ignoreCase), IgnorePaths(ignorePaths)
{
	#ifdef _DEBUG
	setDebugName("CZipReader");
	#endif

	if (File)
	{
		File->grab();

		// scan local headers
		while (scanLocalHeader());

		// prepare file index for binary search
		FileList.sort();
	}
}

CZipReader::~CZipReader()
{
	if (File)
		File->drop();

	int count = (int)FileList.size();
}



//! splits filename from zip file into useful filenames and paths
void CZipReader::extractFilename(SZipFileEntry* entry)
{
	int lorfn = entry->header.FilenameLength; // lenght of real file name

	if (!lorfn)
		return;

	if (IgnoreCase)
		entry->zipFileName.make_lower();

	const char* p = entry->zipFileName.c_str() + lorfn;
	
	// 寻找一个斜线或开始

	while (*p!='/' && p!=entry->zipFileName.c_str())
	{
		--p;
		--lorfn;
	}

	bool thereIsAPath = p != entry->zipFileName.c_str();

	if (thereIsAPath)
	{
		// there is a path
		++p;
		++lorfn;
	}

	entry->simpleFileName = p;
	entry->path = "";

	//复制路径
	if (thereIsAPath)
	{
		lorfn = (int)(p - entry->zipFileName.c_str());
		entry->path.append(entry->zipFileName, lorfn);
	}

	if (!IgnorePaths)
		entry->zipFileName = entry->simpleFileName;
}



//! scans for a local header, returns false if there is no more local file header.
bool CZipReader::scanLocalHeader()
{
	char tmp[1024];

	SZipFileEntry entry;
	entry.fileDataPosition = 0;
	memset(&entry.header, 0, sizeof(SZIPFileHeader));

	File->read(&entry.header, sizeof(SZIPFileHeader));

	if (entry.header.Sig != 0x04034b50)
		return false; // local file headers end here.

	// read filename
	entry.zipFileName.reserve(entry.header.FilenameLength+2);
	File->read(tmp, entry.header.FilenameLength);
	tmp[entry.header.FilenameLength] = 0x0;
	entry.zipFileName = tmp;

	extractFilename(&entry);

	// move forward lenght of extra field.

	if (entry.header.ExtraFieldLength)
		File->seek(entry.header.ExtraFieldLength, true);

	// if bit 3 was set, read DataDescriptor, following after the compressed data
	if (entry.header.GeneralBitFlag & ZIP_INFO_IN_DATA_DESCRITOR)
	{
		// read data descriptor
		File->read(&entry.header.DataDescriptor, sizeof(entry.header.DataDescriptor));
	}

	// store position in file
	entry.fileDataPosition = File->getPos();

	// move forward lenght of data
	File->seek(entry.header.DataDescriptor.CompressedSize, true);

	#ifdef _DEBUG
	//os::Debuginfo::print("added file from archive", entry.simpleFileName.c_str());
	#endif

	FileList.push_back(entry);

	return true;
}



//! opens a file by file name
IReadFile* CZipReader::openFile(const char* filename)
{
	int index = findFile(filename);

	if (index != -1)
		return openFile(index);

	return 0;
}



//! opens a file by index
IReadFile* CZipReader::openFile(int index)
{
	//0 - The file is stored (no compression)
	//1 - The file is Shrunk
	//2 - The file is Reduced with compression factor 1
	//3 - The file is Reduced with compression factor 2
	//4 - The file is Reduced with compression factor 3
	//5 - The file is Reduced with compression factor 4
	//6 - The file is Imploded
	//7 - Reserved for Tokenizing compression algorithm
	//8 - The file is Deflated
	//9 - Reserved for enhanced Deflating
	//10 - PKWARE Date Compression Library Imploding

	switch(FileList[index].header.CompressionMethod)
	{
	case 0: // no compression
		{
			File->seek(FileList[index].fileDataPosition);
			return createLimitReadFile(FileList[index].simpleFileName.c_str(), File, FileList[index].header.DataDescriptor.UncompressedSize);
		}
	case 8:
		{
			int uncompressedSize = FileList[index].header.DataDescriptor.UncompressedSize;
			u32 compressedSize = FileList[index].header.DataDescriptor.CompressedSize;

			void* pBuf = new c8[ uncompressedSize ];
			if (!pBuf)
			{
				os::Warning::print("Not enough memory for decompressing", FileList[index].simpleFileName.c_str());
				return 0;
			}

			char *pcData = new char[ compressedSize ];
			if (!pcData)
			{
				os::Warning::print("Not enough memory for decompressing", FileList[index].simpleFileName.c_str());
				return 0;
			}

			//memset(pcData, 0, compressedSize );
			File->seek(FileList[index].fileDataPosition);
			File->read(pcData, compressedSize );
			
			// Setup the inflate stream.
			z_stream stream;
			int err;

			stream.next_in = (Bytef*)pcData;
			stream.avail_in = (uInt)compressedSize;
			stream.next_out = (Bytef*)pBuf;
			stream.avail_out = uncompressedSize;
			stream.zalloc = (alloc_func)0;
			stream.zfree = (free_func)0;

			// Perform inflation. wbits < 0 indicates no zlib header inside the data.
			err = inflateInit2(&stream, -MAX_WBITS);
			if (err == Z_OK)
			{
				err = inflate(&stream, Z_FINISH);
				inflateEnd(&stream);
				if (err == Z_STREAM_END)
					err = Z_OK;

				err = Z_OK;
				inflateEnd(&stream);
			}


			delete[] pcData;
			
			if (err != Z_OK)
			{
				os::Warning::print("Error decompressing", FileList[index].simpleFileName.c_str());
				delete [] pBuf;
				return 0;
			}
			else
				return io::createMemoryReadFile ( pBuf, uncompressedSize, FileList[index].simpleFileName.c_str(), true);
		}
		break;
	default:
		os::Warning::print("file has unsupported compression method.", FileList[index].simpleFileName.c_str());
		return 0;
	};
}



//! returns count of files in archive
s32 CZipReader::getFileCount()
{
	return FileList.size();
}



//! returns data of file
const SZipFileEntry* CZipReader::getFileInfo(s32 index) const
{
	return &FileList[index];
}



//! deletes the path from a filename
void CZipReader::deletePathFromFilename(core::stringc& filename)
{
	// delete path from filename
	const char* p = filename.c_str() + filename.size();

	// suche ein slash oder den anfang.

	while (*p!='/' && *p!='\\' && p!=filename.c_str())
		--p;

	core::stringc newName;

	if (p != filename.c_str())
	{
		++p;
		filename = p;
	}
}



//! returns fileindex
int CZipReader::findFile(const c8* simpleFilename)
{
	SZipFileEntry entry;
	entry.simpleFileName = simpleFilename;

	if (IgnoreCase)
		entry.simpleFileName.make_lower();

	if (IgnorePaths)
		deletePathFromFilename(entry.simpleFileName);

	int res = FileList.binary_search(entry);

	#ifdef _DEBUG
	if (res == -1)
	{
		for (unsigned int i=0; i<FileList.size(); ++i)
			if (FileList[i].simpleFileName == entry.simpleFileName)
			{
				os::Debuginfo::print("File in archive but not found.", entry.simpleFileName.c_str());
				break;
			}
	}
	#endif

	return res;
}

```

