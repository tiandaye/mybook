## 运行git book

## 安装

```shell
npm install gitbook-cli -g
npm list --depth=0 -g
```

## 卸载

```shell
npm uninstall gitbook-cli -g
```

## 初始化

```shell
gitbook init
```

## 运行

```
git clone git@github.com:tiandaye/mybook.git
gitbook serve
http://localhost:4000 查看

gitbook build
只生成网页而不开启服务器
```

## 安装插件

```
gitbook install
```

## 输出PDF

输出为PDF文件，需要先安装gitbook pdf

```
npm install gitbook-pdf -g
```

然后，用下面的命令就可以生成PDF文件了。

```
gitbook pdf {book_name}
```

如果，你已经在编写的gitbook当前目录，也可以使用相对路径。

```
gitbook pdf .
```

然后，你就会发现，你的目录中多了一个名为 `book.pdf` 的文件。

## 自动生成目录

先全局安装一个模块：

```shell
npm install -g gitbook-summary
```

然后在图书目录下执行：

```
book sm -i _book
```

`-i` 参数用于忽略目录。`_book`是gitbook自动生成的输出目录，它是应该被忽略的。

如果有多个目录需要忽略，可以这样设置参数：

```
book sm -i [_book,node_modules, styles]
```

如果书籍目录下有 `book.json` 文件，就是配置文件在起作用了：

```
{
  "ignores": ["_book","styles","node_modules"],
  ...
```

执行`book sm`，即可自动生成相关章节。当然也可以手动添加章节文件。