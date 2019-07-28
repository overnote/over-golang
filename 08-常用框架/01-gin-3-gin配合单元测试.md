## gin配合单元测试

https://github.com/stretchr/testify/assert  是个很好的单元测试框架。  

在上一节中配置了笔者自己项目的路由模块化思路，下面是配套的单元测试demo：

userRouter_test.go
```go
package test

import (
	"Demo1/router"
	"github.com/stretchr/testify/assert"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestUserRouter_userLogin(t *testing.T) {
	r := router.InitRouter()
	w := httptest.NewRecorder()
	req, _ := http.NewRequest(http.MethodGet, "/user/login", nil)
	r.ServeHTTP(w, req)
	assert.Equal(t, http.StatusOK, w.Code)
	assert.Equal(t, `{"code":10001,"data":null,"msg":"登录成功"}`, w.Body.String())
}

```
