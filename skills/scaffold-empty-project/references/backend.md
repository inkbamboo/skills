# 后端模板（Go）

所有占位符（`{{PROJECT_NAME}}`、`{{SERVER_PORT}}`、`{{DB_NAME}}`）替换后写入对应路径。
`go.mod` 中的 go 版本按本机 `go version` 调整。

## go.mod

```
module {{PROJECT_NAME}}

go 1.27

require (
	github.com/gin-contrib/cors v1.7.8
	github.com/gin-gonic/gin v1.12.0
	github.com/inkbamboo/ares v0.0.7
	github.com/labstack/gommon v0.5.0
	github.com/urfave/cli/v2 v2.27.7
)
```

（执行 `go mod tidy` 会自动补全 indirect 依赖）

## cmd/server/main.go

```go
package main

import (
	"fmt"
	"os"
	"{{PROJECT_NAME}}/internal/bootstrap"
	"time"

	. "github.com/inkbamboo/ares/utils"
	"github.com/labstack/gommon/color"
	"github.com/urfave/cli/v2"
)

// Commands 其余子命令（server 为默认任务，不放在这里）
func Commands() []*cli.Command {
	return []*cli.Command{
		{
			Name:        "test",
			Aliases:     []string{"t"},
			Usage:       "Run test",
			Description: "Run test",
			Action: func(c *cli.Context) error {
				OnStop(func() {
					fmt.Println("OnStop, clean server...")
				})
				bootstrap.Init(c)
				bootstrap.RunTest()
				return nil
			},
		},
	}
}

func main() {
	line := "==============================="
	fmt.Println(fmt.Sprintf("%s%s%s%s",
		color.White(line),
		color.Bold(color.Green("任务列表")),
		color.Bold(color.Yellow("["+time.Now().Format("2006-01-02 15:04:05")+"]")),
		color.White(line)))
	fmt.Println(color.Bold(color.White("包含以下任务:")))

	app := cli.NewApp()
	app.Name = "{{PROJECT_NAME}}"
	app.Usage = "A {{PROJECT_NAME}} application powered by Ripple framework"
	app.Flags = []cli.Flag{
		&cli.StringFlag{
			Name:        "conf",
			Value:       "./config",
			DefaultText: "./config",
			Usage:       "配置文件路径",
		},
		&cli.StringFlag{
			Name:        "env",
			DefaultText: "dev",
			Usage:       "执行环境 (开发环境dev、测试环境test、线上环境prod)",
		},
	}
	app.Version = "1.0.0"
	// 默认任务：启动 server（直接运行，无需子命令）
	app.Action = func(c *cli.Context) error {
		OnStop(func() {
			fmt.Println("OnStop, clean server...")
		})
		bootstrap.Init(c)
		bootstrap.RunServer()
		return nil
	}
	app.Commands = Commands()

	fmt.Println("任务0：启动 web 服务（默认，直接运行无需参数）")
	for key, command := range app.Commands {
		fmt.Println(fmt.Sprintf("任务%d：%s %s %s", key+1, command.Name, command.Usage, command.Description))
	}

	if err := app.Run(os.Args); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

## internal/bootstrap/bootstrap.go

```go
package bootstrap

import (
	"os"

	"github.com/inkbamboo/ares"
	"github.com/inkbamboo/ares/middlewares/logger"
	"github.com/urfave/cli/v2"
)

var Logger *logger.Logger

// Init 加载配置并初始化日志
func Init(c *cli.Context) {
	ares.InitConfigWithPath(c.String("env"), c.String("conf"))
	ares.GetConfig().Set("env", c.String("env"))
	Logger = NewLogger()
}

func NewLogger() *logger.Logger {
	log, err := logger.NewLogger("server", 1, os.Stdout)
	if err != nil {
		panic(err) // Check for error
	}
	return log
}
```

## internal/bootstrap/run.go

```go
package bootstrap

import (
	"fmt"

	"{{PROJECT_NAME}}/internal/controllers"

	"github.com/inkbamboo/ares"
)

func RunServer() {
	Logger.Info("Run server ....")
	controllers.RouteAPI()
	ares.Default().Run()
}

func RunTest() {
	// 临时调试代码写在这里
	fmt.Println("run test ...")
}
```


## internal/controllers/router.go

```go
package controllers

import (
	"net/http"

	"{{PROJECT_NAME}}/internal/controllers/v1"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"github.com/inkbamboo/ares"
)

func RouteAPI() {
	ginMux := ares.Default().GetGin()

	// CORS 跨域
	ginMux.Use(cors.New(cors.Config{
		AllowOrigins: []string{"*"},
		AllowHeaders: []string{"*"},
		ExposeHeaders: []string{"*"},
		AllowMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE", "HEAD", "OPTIONS"},
	}))
	// 健康检查
	ginMux.GET("/ping", func(c *gin.Context) {
		c.String(http.StatusOK, "pong")
	})

	registerAPI(ginMux)
}

func registerAPI(ginMux *gin.Engine) {
	v1Group := ginMux.Group("/api/v1")

	// 在此注册各业务 Controller，例如：
	demo := v1.NewDemoController(v1Group.Group("/demo"))
	demo.Setup()
}
```

## internal/controllers/v1/demo.controller.go

```go
package v1

import (
	"{{PROJECT_NAME}}/internal/consts/ecode"
	"{{PROJECT_NAME}}/internal/base"
	"{{PROJECT_NAME}}/internal/dto"
	"{{PROJECT_NAME}}/internal/services"

	"github.com/gin-gonic/gin"
)

type DemoController struct {
	base.BaseController
	Group *gin.RouterGroup
}

func NewDemoController(group *gin.RouterGroup) *DemoController {
	return &DemoController{Group: group}
}

func (c *DemoController) Setup() {
	c.Group.GET("/hello", c.Hello)
}

// Hello
// @Summary 示例接口
// @Description 返回问候语
// @Tags 示例
// @Produce json
// @Param name query string false "名称"
// @Success 200 {object} ecode.Response{data=dto.DemoHelloOut}
// @Router /v1/demo/hello [get]
func (c *DemoController) Hello(ctx *gin.Context) {
	var (
		req dto.DemoHelloIn
		res *dto.DemoHelloOut
		err error
	)
	defer func() {
		ecode.JSONResponse(ctx, res, err)
	}()
	if err = ctx.ShouldBind(&req); err != nil {
		err = ecode.ParamError.ErrorWithMessage(err.Error())
		return
	}
	res, err = services.GetDemoService().Hello(&req)
}
```

## internal/dto/base_dto.go

```go
package dto

type PageQuery struct {
	Page     int `form:"page" json:"page"`
	PageSize int `form:"page_size" json:"page_size"`
}
```

## internal/dto/demo_dto.go

```go
package dto

type DemoHelloIn struct {
	Name string `form:"name" json:"name"`
}

type DemoHelloOut struct {
	Message string `json:"message"`
}
```

## internal/model/base_model.go

```go
package model

import (
	"time"
)

type BaseModel struct {
	ID        int64      `gorm:"primaryKey;column:id;not null;unsigned" json:"id"`               // 自增唯一ID
	CreatedAt *time.Time `gorm:"column:created_at;type:datetime;null;autoCreateTime" json:"created_at"` // 创建时间
	UpdatedAt *time.Time `gorm:"column:updated_at;type:datetime;null;autoUpdateTime" json:"updated_at"` // 更新时间
}
```

## internal/services/demo_service.go

```go
package services

import (
	"fmt"

	"{{PROJECT_NAME}}/internal/dto"
)

type DemoService struct{}

func GetDemoService() *DemoService {
	return &DemoService{}
}

func (s *DemoService) Hello(req *dto.DemoHelloIn) (res *dto.DemoHelloOut, err error) {
	name := req.Name
	if name == "" {
		name = "world"
	}
	res = &dto.DemoHelloOut{Message: fmt.Sprintf("hello, %s!", name)}
	return
}
```

## internal/services/demo_service_test.go

```go
package services

import (
	"testing"

	"{{PROJECT_NAME}}/internal/dto"
)

func TestDemoService_Hello(t *testing.T) {
	service := GetDemoService()

	t.Run("with name", func(t *testing.T) {
		res, err := service.Hello(&dto.DemoHelloIn{Name: "{{PROJECT_NAME}}"})
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if res.Message != "hello, {{PROJECT_NAME}}!" {
			t.Errorf("unexpected message: %s", res.Message)
		}
	})

	t.Run("without name", func(t *testing.T) {
		res, err := service.Hello(&dto.DemoHelloIn{})
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if res.Message != "hello, world!" {
			t.Errorf("unexpected message: %s", res.Message)
		}
	})
}
```

## internal/utils/util.go

```go
package utils

// 通用工具函数统一放在本包
```

## internal/consts/constant.go

```go
package consts

const (
	PASSWORD_DEFAULT   = "123456"
	USER_STATUS_BANNED = 1
)
```

## internal/consts/cache_keys.go

```go
package consts

// Redis 缓存 key 统一在此定义，例如：
// const CacheKeyUserToken = "user:token:%s"
```

## internal/consts/ecode/ecode.go

```go
package ecode

import (
	"errors"
	"fmt"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

type Response struct {
	Code int         `json:"code"`                                // code码, 0为成功
	Data interface{} `json:"data,omitempty" swaggertype:"object"` // 返回数据
	Msg  string      `json:"msg,omitempty"`                       // 返回消息
	Now  int64       `json:"now"`                                 // 服务器时间戳
}

func (r *Response) Error() string {
	return fmt.Sprintf("ecode=%d, message=%s", r.Code, r.Msg)
}

func (r *Response) ErrorWithMessage(msg string) *Response {
	return Error(r.Code, msg)
}

func Error(code int, msg string) *Response {
	return &Response{Code: code, Msg: msg, Now: time.Now().Unix()}
}

// JSONResponse 统一 JSON 响应，配合 Controller 中的 defer 使用
func JSONResponse(ctx *gin.Context, data interface{}, err error) {
	var res *Response
	defer func() {
		if e := recover(); e != nil {
			res = &Response{
				Code: ServerError.Code,
				Msg:  e.(error).Error(),
				Now:  time.Now().Unix(),
			}
		}
		ctx.JSON(http.StatusOK, res)
		ctx.Abort()
	}()
	if err == nil {
		res = &Response{
			Code: Success.Code,
			Data: data,
			Msg:  Success.Msg,
			Now:  time.Now().Unix()}
	} else if errors.As(err, &res) {
		res.Now = time.Now().Unix()
	} else {
		res = &Response{
			Code: ServerError.Code,
			Msg:  err.Error(),
			Now:  time.Now().Unix(),
		}
	}
}

var (
	Success     = &Response{Code: 0, Msg: "成功"}
	ServerError = &Response{Code: 10001, Msg: "服务器内部错误"}
	ParamError  = &Response{Code: 10002, Msg: "参数错误"}
)
```

## internal/base/controller.go

Controller 基类放在独立的 `internal/base` 包（不能放在 `controllers` 包：`controllers/router.go` 要导入 `controllers/v1` 注册路由，v1 又要用基类，包级循环导入会编译报错）。

```go
package base

import (
	"strings"

	"github.com/gin-gonic/gin"
)

type BaseController struct {
}

func (c *BaseController) GetIp(ctx *gin.Context) string {
	forwardedFor := ctx.GetHeader("X-Forwarded-For")
	if forwardedFor != "" {
		ips := strings.Split(forwardedFor, ",")
		return strings.TrimSpace(ips[0])
	}
	return ctx.ClientIP()
}
```

## sql/init.sql

```sql
-- ============================================================
-- 数据库初始化脚本
-- ============================================================
CREATE DATABASE IF NOT EXISTS `{{DB_NAME}}` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
USE `{{DB_NAME}}`;

-- 建表语句统一写在这里
```

## static/files/.gitkeep

空文件即可。
