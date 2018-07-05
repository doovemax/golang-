# golang–archive/tar

## index

+ type Format

  Format 类型里面是各种tar包的格式，具体格式内容参考[GUN tar man page](http://www.gnu.org/software/tar/manual/tar.html)

  -  [func (f Format) String() string](https://golang.org/pkg/archive/tar/#Format.String)

+ type Header 

  tar 文件头部信息

   -  [func FileInfoHeader(fi os.FileInfo, link string) (*Header, error)](https://golang.org/pkg/archive/tar/#FileInfoHeader)

      FileInfoHeader 的作用是将 FileInfo 实例转化为tar的Header，但转化的Header并不完全，

      例如name只是basename，而不是fullname（全路径），当文件为软连接文件的时候，link字段为源文件

      的全路径

   -  [func (h *Header) FileInfo() os.FileInfo](https://golang.org/pkg/archive/tar/#Header.FileInfo)

      将tar.Header 转化为os.FileInfo

+ type Reader

  Reader提供对tar文件的顺序访问，Reader.Next进入tar存档的下一个文件（包括第一个文件），

  Reader可以作为io.Reader接口来访问文件的数据

   -  [func NewReader(r io.Reader) *Reader](https://golang.org/pkg/archive/tar/#NewReader)

      从一个io.Reader接口类型创建一个tar.Reader的结构体

   -  [`func (tr *Reader) Next() (*Header, error)`](https://golang.org/pkg/archive/tar/#Reader.Next)

      读取下一个文件Header

   -  [`func (tr *Reader) Read(b []byte) (int, error)`](https://golang.org/pkg/archive/tar/#Reader.Read)

      读取文件内容	

+ type Writer

  Writer类型提供给tar文件的顺序写入，Write.WriteHeader从给定的第一个Header开始，

  可以将Writer视为io.Writer来提供该文件的数据

  - [func NewWriter(w io.Writer) *Writer](https://golang.org/pkg/archive/tar/#NewWriter)

    创建一个新的tar.Writer

  - [func (tw *Writer) Close() error](https://golang.org/pkg/archive/tar/#Writer.Close)

    关闭打开的wirter，同时写入数据。必须调用close方法，结束文件读写，close会向tar文件写入结束字段

  - [func (tw *Writer) Flush() error](https://golang.org/pkg/archive/tar/#Writer.Flush)

    将数据写入tar文件

  - [`func (tw *Writer) Write(b []byte) (int, error)`](https://golang.org/pkg/archive/tar/#Writer.Write)

    将数据写入当前存档文件。如果在writeHeader之后写入超过Header.Size字节，则Write返回ErrWriteTooLong错误

  - [func (tw *Writer) WriteHeader(hdr *Header) error](https://golang.org/pkg/archive/tar/#Writer.WriteHeader)

    将文件头部信心hdr写入当前的tar文件





## Tar程序思路：

1. 了解tar文件结构
2. 如何获取文件信息，写入tar文件
3. 创建tar文件，判断是否存在，存在是否覆盖
4. 如何写入多个文件和目录



```golang
package main

// 非正常文件，如：软连接，硬链接文件没有处理，打包文件存在没有处理
import (
	"archive/tar"
	"fmt"
	"io"
	"os"

	"path/filepath"
)

var (
	fileAllList []newFileInfo //文件信息列表
)

type newFileInfo struct {
	fileFullName string
	fileinfo     os.FileInfo
}

func main() {
	if len(os.Args) <= 1 {
		fmt.Println("No args ")
		help()
		os.Exit(2)
	}

	fileArg := os.Args[1:]
	fileTar := fileArg[0]
	fileList := fileArg[1:]

	// fmt.Println(fileTar, fileList)

    // 创建tar文件
	var fileTarW *tar.Writer
	if _, err := os.Stat(fileTar); err != nil {
		if os.IsNotExist(err) {
			fmt.Println(err)
			w, err := os.Create(fileTar)
			if err != nil {
				panic(err)
			}
			defer w.Close()

			fileTarW = tar.NewWriter(w)
			if err != nil {
				panic(err)
			}
			defer fileTarW.Close()

		} else {
			fmt.Println(err)
			panic(err)
		}
	}

	for _, file := range fileList {
		fileStat, err := os.Stat(file)
		if err != nil {
			fmt.Println(err)
			os.Exit(3)
		}
		
		// 判断参数是文件或文件夹
		if fileStat.IsDir() {
			err = filepath.Walk(file, walkFunc)
			if err != nil {
				fmt.Println(err)
				os.Exit(2)
			}

		} else {

			fileAllList = append(fileAllList, newFileInfo{
				file,
				fileStat,
			})
		}

	}

	for _, f := range fileAllList {
		// fmt.Println(f.fileFullName)
		err := addToTar(f.fileFullName, f.fileinfo, fileTarW)
		if err != nil {
			fmt.Println("123")
			panic(err)
		}
	}

}

func help() {
	fmt.Println("xintar *.tar file1 file2 dir1 dir2")

}
// walk方法会调用walkFunc方法，通过这个方法将文件加入fileAllList
func walkFunc(path string, info os.FileInfo, err error) error {
	if !info.IsDir() {
		fileAllList = append(fileAllList, newFileInfo{path, info})
		//fmt.Println(path, "----", info.Name())
		return nil
	} else {
		return nil
	}
}

//将文件写入tar文件
func addToTar(name string, info os.FileInfo, tarw *tar.Writer) error {
	file, err := os.Open(name)
	if err != nil {
		return err
	}
	defer file.Close()

	fileTarInfo, _ := tar.FileInfoHeader(info, "")
	fileTarInfo.Name = name
	err = tarw.WriteHeader(fileTarInfo)
	if err != nil {
		return err
	}
	_, err = io.Copy(tarw, file)
	if err != nil {
		return err
	}
	tarw.Flush()
	return nil
}
```



