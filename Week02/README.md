1. go就是普通的error接口
	type error interface{
		Error() string
	}
	
	type stringError struct {
		s string
	}
	
	func (e *stringError)Error() string {
		return e.s
	}
	
	func New(test string) error {
		return &stringError{test}
	}
	
	
2. go1.13 error扩展
	
3. 约定：error在多返回值的最后一个位置

4. request请求中 recover住panic，打印出堆栈，然后返回5xxx。但是recover不住异步的goroutinue

5. 野生goroutinue，recover不住。

6. 自动封装一层go
	func Go(x func() {}) {
		if ree := recover(); err != nil {
			//xxxx
		}
		go x()
	}
	
7. 不要来一个请求，就立马开一个goroutinue。应封装成一个message，通过channel传到后台工作池中，进行集中处理。

8. main函数中，强依赖的启动失败，可以panic；init初始化panic；配置文件panic

9. dao层一个记录找不到，是返回error，还是返回空指针？

10. sentinel error：预定义error，哨兵error。无法携带上下文，使用不灵活，尽可能避免。业务错误码可以处理？？

11. 不透明的错误模型。只对err进行nil判断，不对内容进行判断。

12. 代码示例：
	==土鳖代码===
	func CountLines(r. io.Reader)(int, error){
		var (
			br := bufio.NewReader(r)
			lines int
			err error
		)
		
		for {
			_, err := br.ReadString("\n")
			lines++
			if err != nil {
				break
			}
		}
		
		if err != io.EOF {
			return 0, err
		}
		return lines, nil
		
	}
	
	==清真代码==

	func CountLines(r io.Reader) (int, error) {
		sc := bufio.NewScanner(r)
		lines := 0
		
		for sc.Sacn() {
			lines++
		}
		
		return lines, sc.Err()
	}


	==土鳖代码==
	type Head struct{
		Key, Value string
	}
	type Status struct {
		Code int
		Reasom string
	}
	
	func WriteResponsew(w io.Writer, st Status, headers []Header, body io.Reader) error{
		_, err := fmt.Fprintf(w, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
		if err != nil {
			return err
		}
		
		for _, h := range headers {
			_, err := fmt.Fprintf(w, "%s: %s\r\n", h.Key, h.Value)
			if err != nil {
				return err
			}
		}
		
		if _, err := fmt.Fprintf(w, "\r\n"); err != nil {
			return err
		}
		_, err = io.Copy(w, body)
		return err
		
	}
		
	==清真代码==
	type errWriter struct{
		io.Writer
		err error
	}
	
	func (e *errWriter) Write(buf []byte) (int, error) {
		if e.err != nil {
			return 0, e.err
		}
		
		var n int
		n, e.err = e.Writer.Write(buf) //返回时不做处理，当下次调用时首先处理
		return n, nil
	}

	func WriteResponsew(w io.Writer, st Status, headers []Header, body io.Reader) error{
		ew := &errWriter{w}
		fmt.Fprintf(ew, "HTTP/1.1 %d %d\r\n", st.Code, st.Reason)
		for _, h := range headers {
			fmt.Fprintf(ew, "%s: %s\r\n", h.Key, h.Value)
		}
		fmt.Fprintf(ew, "\r\n")
		io.Copy(ew, body)
		
		return e.err // 代码改进后只做一次判断
	
	}

12. 无错误的正常流程代码，将成为一条直线，而不是缩进的代码。//不要在所缩进里处理正常的业务逻辑.

13. 你只应该处理error一次：
		反面示例：
		func xxx() {
		data, err := do()
		if err != nil {
			log.Info("这里出错:", err) //第一次处理
			return err                // 返回后，上面还会在处理一次
		
		}
		正确处理：
		a. 吞掉错误，给data一个默认值。
		b. 错误要被日志记录
		c. 之后不再暴露当前错误
