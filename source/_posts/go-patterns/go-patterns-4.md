---
title: Go编程模式 - 4.错误处理
date: 2021-02-20 18:31:57
categories: 
- 经典品读
tags:
- Go-Programming-Patterns
---

>  注：本文的灵感来源于GOPHER 2020年大会陈皓的分享，原PPT的[链接](https://www2.slideshare.net/haoel/go-programming-patterns?from_action=save)可能并不方便获取，所以我下载了一份[PDF](https://github.com/Junedayday/code_reading/tree/master/doc/Go_Programming_Patterns.pdf)到git仓，方便大家阅读。我将结合自己的实际项目经历，与大家一起细品这份文档。



## 目录

- [函数式处理](#Functional)
- [对象嵌入错误](#ErrorObject)
- [错误包装](#Wrap)



## Functional

```go
type Number struct {
	a int
	b string
	c bool
	d []int32
	e error
}

func (n *Number) parse(r io.Reader) error {
	if err := binary.Read(r, binary.BigEndian, &n.a); err != nil {
		return err
	}
	if err := binary.Read(r, binary.BigEndian, &n.b); err != nil {
		return err
	}
	if err := binary.Read(r, binary.BigEndian, &n.c); err != nil {
		return err
	}
	if err := binary.Read(r, binary.BigEndian, &n.d); err != nil {
		return err
	}
	if err := binary.Read(r, binary.BigEndian, &n.e); err != nil {
		return err
	}
	return nil
}
```



引入了函数式编程的方式，我们看看有什么改变

```go
func (n *Number) parse(r io.Reader) error {
	// 先定义一个error
	var err error

	// 定义函数，注意这里的err的作用域是来自上面定义的
	read := func(data interface{}) {
		// 先检查error，如果已经有错误则不检查
		if err != nil {
			return
		}
		err = binary.Read(r, binary.BigEndian, data)
	}

	// 注意，这个函数的调用逻辑和之前的差别在于一点：
	// 即使前面的发生了error，下面的函数也会被调用
	read(&n.a)
	read(&n.b)
	read(&n.c)
	read(&n.d)
	read(&n.e)

	return err
}
```



## ErrorObject

先看一个标准库中的实现

```go
func main() {
	input := bytes.NewReader([]byte("hello"))
	
	// 扫描数据，这里不会直接返回错误
	scanner := bufio.NewScanner(input)
	for scanner.Scan() {
		token := scanner.Text()
		fmt.Println(token)
	}
	
	// 从Err()方法中获取错误
	if err := scanner.Err(); err != nil {
		fmt.Println(err)
	}
}
```

它的根本思想，是将`error`嵌入到了对象中。那我们借鉴一下

```go
type Reader struct {
	r   io.Reader
	err error
}


func (r *Reader) read(data interface{}) {
	if r.err == nil {
		r.err = binary.Read(r.r, binary.BigEndian, data)
	}
}

func (n *Number) parse(reader io.Reader) error {
	r := Reader{r: reader}

	r.read(&n.a)
	r.read(&n.b)
	r.read(&n.c)
	r.read(&n.d)
	r.read(&n.e)

	return r.err
}
```

捎带提一句：个人不太喜欢上面`scanner`的错误处理方式，这个要求使用方对这个包很熟悉，否则很容易忘掉后面的错误处理逻辑。但后面处理错误的逻辑，就很直接地将错误返回，可读性很强。

## Wrap

耗子叔给的例子是调用了`github.com/pkg/errors`下的wrap包，不过我更倾向于直接用原生的。

```go
func main() {
	// 原始 error
	err := errors.New("level 1")
	fmt.Println(err)
	// level 1

	// wrap一下error，注意error的占位符是%w
	wraped := fmt.Errorf("%v: %w", "level 2", err)
	fmt.Println(wraped)
	// level 2: level 1

	// unwrap 后获得原来的错误
	unwraped := errors.Unwrap(wraped)
	fmt.Println(unwraped)
	// level 1

	// 过度unwrap会导致错误变成nil
	unwraped2 := errors.Unwrap(unwraped)
	fmt.Println(unwraped2)
	// nil
}
```

但在实际项目实践中，`Wrap`的这个特性并不好用：

如何Wrap Error，在多人协同开发、多模块开发过程中，很难统一。而一旦不统一，容易出现示例中的过度Unwrap的情况。

所以，我认为与其花大精力在制定错误的标准上，还不如利用`fmt.Errorf`将错误信息直观地表述出来。



> Github: https://github.com/Junedayday/code_reading
>
> Blog: http://junes.tech/
>
> Bilibili：https://space.bilibili.com/293775192
>
> 公众号：golangcoding

